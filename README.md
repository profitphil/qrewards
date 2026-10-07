# QREWARDS — Open Multi-Pool Loyalty & Dividend Platform: Build Guide

> Reference implementation: `src/contracts/QRewards.h`
> Purpose: let anyone create a **pool**. Holding a pool's registered ecosystem assets
> earns a **weighted share** of that pool's dividends. Outside contracts (or anyone)
> feed QU or tokens into a **specific pool** via `depositDividend(poolId, …)`; the
> accumulated pot is paid out **at each epoch end, pro-rata by live holdings**. Pools
> are fully isolated: a pool can only ever pay out what was deposited to it.

---

## 1. Distribution model — pure snapshot (why there is no claim step)

QREWARDS follows the **qRWA pattern** (`src/contracts/qRWA.h`): it does **not** keep a
persistent reward-token balance. Instead:

1. Every `depositDividend` (minus the 5% fee) accumulates into a per-currency **pot**
   for the pool.
2. At **`END_EPOCH`**, for each active pool the contract reads **live holdings straight
   from the asset ledger**, computes each holder's weight, and transfers out the pot
   pro-rata by weight — QU via `qpi.transfer`, tokens via `transferShareOwnershipAndPossession`.

**Why this matters:** because every epoch re-reads the real ledger, a holder who sold
their tokens has a live balance of 0 and is paid nothing, and the same tokens can never
be counted under two positions. There are **no stale "ghost" balances**, no double-count,
and therefore **no `claimDividends` and no `syncProfit`** — holders are simply paid.

This replaced an earlier MasterChef-style accumulator whose stored per-user "points"
could drift from reality (a seller who was never re-synced kept earning, and a buyer of
the same tokens was counted a second time, inflating the denominator). The snapshot model
removes that whole class of bug at the root.

### Scale & timing caveats (read these)
- **Scale:** `END_EPOCH` iterates each active pool's registered assets' holders into a
  scratch map of up to `QREWARDS_SNAPSHOT_CAP = 16384` unique holders per pool. This is
  sized for **a handful of active pools of up to a few thousand holders each** (the design
  target). It is *not* a mass sweep of thousands of busy pools in one epoch.
- **Timing:** distribution uses the **END_EPOCH snapshot only**. That fully prevents
  stale/ghost over-payment. It does **not** by itself stop someone buying a large position
  right before the epoch boundary to grab that epoch's dividend and selling after. qRWA
  blocks that with a `min(begin-epoch, end-epoch)` rule; adding it here would require a
  `BEGIN_EPOCH` snapshot kept through the epoch (more state/code). Ship end-only first; add
  min(begin,end) if dividend-sniping ever matters.

---

## 2. Decisions baked in

| Topic | Decision |
|---|---|
| Multi-tenant | Anyone can `createPool`; each pool has its own admin, registry, dividend pots. |
| Scale | `MAX_POOLS = 1024`; per-epoch snapshot `QREWARDS_SNAPSHOT_CAP = 16384` holders/pool. |
| State size | ~10–15 MB (the 4.2M-position store is gone). |
| Reward basis | **Live holdings**, read each epoch; no persistent reward-token balance. |
| Dividend currencies | **QU** (slot 0) + up to 3 asset currencies per pool (`MAX_DIV_CURRENCIES = 4`). |
| Distribution | At `END_EPOCH`, each pool's per-currency **pot** split pro-rata by live weight. Rounding dust carries to the next epoch's pot. |
| Dividend fee | **5%** of every deposit (QU **and** tokens), split **80% / 20%**: 20% → QREWARDS shareholders; 80% → **QPAYHUB dividends account for QU**, **QRaffle charity address for tokens**. **95% reaches holders.** |
| Pool creation | **Open**. Fee **5,000,000 QU** (protocol-owner tunable), split **70% QREWARDS shareholders / 15% QPAYHUB address / 15% burn**. Any excess seeds the pool's operating balance. |
| Operating fee | **100,000 QU/epoch per pool** (protocol-owner tunable), drawn at `END_EPOCH` from a per-pool operating balance the admin tops up (`depositOperating`); same 70/15/15 split. Underfunded → **paused up to `QREWARDS_MAX_MISSED_EPOCHS` (2) epochs, then deactivated**. On deactivation the pool's remaining funds (every currency's `pot + distributable`, plus the leftover operating balance) are **refunded to the pool admin** so nothing is stranded. |
| Excluded addresses | Per-pool `setExcludedAddress`; an excluded address's weight is forced to 0 (earns nothing, dilutes nobody). |
| Split-resistance | Per-asset weight = `(held/unit) × concentrationMultiplier × weightBps`; the multiplier rises with holding size, so splitting a balance across wallets never increases total weight. |

---

## 3. State layout (`src/contracts/QRewards.h`)

```cpp
struct AssetRule   { uint64 assetName; id issuer; uint64 unit; uint32 weightBps; uint8 kind; uint8 active; };
struct DivCurrency { uint64 assetName; id issuer; uint64 pot; uint64 lifetime; uint8 active; }; // assetName==0&&issuer==NULL_ID => QU
struct PoolMeta    { id admin; uint64 label, operatingBalance, lastTotalWeight;
                     uint32 numAssets, missedEpochs, listVersion; uint8 numCurrencies, active, paused, targetMode;
                     Array<DivCurrency, 4> currencies; };
struct KeyProto    { uint64 poolId; id wallet; };   // zeroed then hashed -> composite exclusion/recipient key
struct RecipientEntry { id wallet; uint64 poolId, weight; uint32 version; }; // list-mode payout target

struct StateData {
    HashMap<id, uint8,  QREWARDS_MAX_EXCLUSIONS> excluded;   // key = K12(poolId, address) -> 1
    HashMap<id, uint64, QREWARDS_SNAPSHOT_CAP>   snapshot;   // scratch holder->weight, rebuilt per pool at END_EPOCH
    HashMap<id, RecipientEntry, QREWARDS_MAX_RECIPIENTS> recipients; // key = K12(poolId, wallet), list-mode targets
    uint64 pendingDivFeeQU;                                  // QU dividend-fee skim, flushed 80/20 at END_EPOCH
    Array<PoolMeta, QREWARDS_MAX_POOLS>  poolMeta;
    Array<AssetRule, MAX_POOLS*MAX_ASSETS_PER_POOL> registry; // flat: pool p asset i at p*MAX_ASSETS + i
    HashMap<id, uint64, QREWARDS_MAX_FUNDERS> fundingRoute;   // sourceId -> poolId (tagged QU transfers)
    uint32 numPools;
    id platformOwner, qpayhubAddress, qpayTokenDividendsAddress;
    uint64 createPoolFee, operatingFee;
};
```

Notes:
- The **composite exclusion key** is hashed from a zeroed proto (`setMemory(proto,0)` before
  setting fields) so padding bytes can't make `K12` nondeterministic.
- `snapshot` is scratch: `reset()` at the start of each pool's distribution, rebuilt from the
  live ledger, then iterated to pay out. It is `reset()` again at the end of `END_EPOCH`.

---

## 4. Reward / weight model (per pool, per asset)

```
basePoints = held / unit                              // floor; held < unit => 0  (held = live possessed shares)
mult       = concentrationMultiplier(basePoints)      // bps
weight_a   = basePoints * mult * weightBps / (10000*10000)
weight     = Σ weight_a over the pool's active assets // a holder's total weight
```

`concentrationMultiplier`: `≥200 → 2.0x`, `≥50 → 1.5x`, `≥10 → 1.25x`, `≥1 → 1.0x`, else 0.
The multiplier is **per asset** (keyed on that asset's `basePoints`), so a holder's total
weight is the sum of independent per-asset contributions — which is exactly what lets the
`END_EPOCH` snapshot total everyone up by walking each asset's holder list once.

**Overflow safety:** each per-asset contribution is computed in **128-bit and saturated** to
`uint64` (`weightContribution`), and a holder's summed weight is saturated too (capping a
numerator can only under-pay). The pool's `totalWeight` denominator is the **exact `uint128` sum**
of the stored per-holder weights, so `pot × weight / totalWeight` is computed with no 64-bit wrap
and `Σ payouts ≤ pot` always holds.

`previewWeight(poolId, wallet)` returns this live weight; divide by `getPool().lastTotalWeight`
for an approximate share of the next payout.

---

## 5. `END_EPOCH` (the engine)

For the whole contract, once per epoch:
1. **Operating fee:** draw `operatingFee` from each active pool's `operatingBalance`. Underfunded
   → pause (and deactivate after `MAX_MISSED_EPOCHS`; on deactivation, remaining pot/distributable
   and leftover operating balance are refunded to the pool admin). Collected fees → `DistributeFee`.
2. **Dividend-fee flush:** `pendingDivFeeQU` → `DistributeDivFee` (20% shareholders via
   `distributeDividends`, 80% + rounding to `qpayhubAddress`, burned if unset).
3. **Distribution:** for each pool, `DistributePool`:
   - **Pass 1** — walk each active registered asset's possessors (`AssetPossessionIterator`),
     read live `numberOfPossessedShares`, skip `SELF` and excluded addresses, add the per-asset
     weight into the `snapshot` map and into `totalWeight`.
   - **Pass 2** — for each currency with `pot > 0`, **freeze** the amount and persist `pot = 0`
     *before* paying (checks-effects-interactions: stored state never shows funds that are
     mid-transfer), then iterate the snapshot and pay each holder `frozen × weight / totalWeight`
     (128-bit multiply to avoid overflow); the undistributed remainder (rounding dust + any failed
     sends) is returned to `pot` after the loop. `lastTotalWeight` is recorded for front ends.
   - If `totalWeight == 0` (no eligible holders) the pots carry to a future epoch.

This all-at-once path (`distributionMode == QREWARDS_DIST_END_EPOCH`, the default) is simplest and
fine for a handful of pools. For scale, flip to streamed mode (below).

---

## 5b. Optional: streamed distribution (scale mode, dormant by default)

A second engine spreads distribution across the **whole epoch** instead of one tick, so it scales
to many pools / large holder counts. It is **off by default** (`distributionMode = 0`); both code
paths ship, and the protocol owner flips it on with one transaction — **no redeploy**:

- `setDistributionMode(1)` → streamed; `setDistributionMode(0)` → back to END_EPOCH.
- The start delay (`QREWARDS_STREAM_DELAY_TICKS ≈ 24h`) and per-tick batch (`QREWARDS_STREAM_BATCH`)
  are **fixed constants** — there is no setter to change them. `getPlatform` reports the mode, the
  fixed params, and whether a cycle is active.

**How it runs** (`END_TICK`, once `delayTicks` into the epoch, once per epoch):
a cycle walks pools in order via a persisted cursor. For each pool it (1) rolls `pot → distributable`
and **freezes** the holder snapshot in one tick (`SnapshotPool`), then (2) pays the frozen amounts
out over as many subsequent ticks as needed (`pot × weight / totalWeight`), carrying dust in
`distributable`. `END_EPOCH` skips its own distribution loop in this mode.

**Why it's safe/idempotent:** the snapshot is frozen in a single tick (the ledger iterator can't be
resumed across ticks), but payout streams freely. All cursors live in state, so a cycle resumes
exactly across ticks — and even across epoch boundaries (it's never reset mid-cycle) — so no holder
is paid twice and undistributed funds always stay in `pot`/`distributable` and carry.

**Limits:** payout duration and pool count are effectively unbounded; the only ceiling is a *single*
pool whose holder set is too large to **freeze in one tick** (~100K+, far beyond typical use). A
giant single pool would need intra-pool freeze pagination — not implemented (cap such a pool instead).

---

## 6. Fees

**Create / operating fee** (`DistributeFee`): 70% QREWARDS shareholders (`distributeDividends`),
15% `qpayhubAddress`, 15% burn (its 15% burns too if no QPAYHUB address is set).

**Dividend fee** — 5% of every deposit, always 80/20 (20% → QREWARDS shareholders), but the
80% destination differs by currency:
- **QU dividend → 80% to the QPAYHUB dividends account (`qpayhubAddress`).** The whole fee
  accrues to `pendingDivFeeQU` and is flushed at `END_EPOCH`: 20% via `distributeDividends`,
  80% (+ rounding) via a bare `qpi.transfer` that QPAYHUB's `POST_INCOMING_TRANSFER`
  auto-credits into its `feePool`. Burned if `qpayhubAddress` is unset.
- **Token dividend → 80% to the QRaffle charity address (`qpayTokenDividendsAddress`).** Settled
  inline at deposit: 20% to shareholders paid **in the token itself** by iterating the contract
  share asset (`"QREWARD"` packed = `19230739006837329`, issuer `NULL_ID`) via
  `DistributeTokenToShareholders`; the rest (80% + dust + any failed shareholder sends) → the
  charity address. Unset → no fee is taken on tokens.

Why two destinations: QPAYHUB's `feePool` only accepts **plain QU** (its `POST_INCOMING_TRANSFER`
ignores incoming assets), so token fees can't be credited there. They are instead sent to the
**QRaffle charity address** (a plain wallet that simply receives the tokens) — note QRaffle itself
has no handler to redistribute tokens sent to its *contract* address, so the charity *wallet* is
used, not the QRaffle contract. `qpayTokenDividendsAddress` is owner-tunable.

The per-share flooring in the token shareholder leg means it only pays out once `20% of the fee
≥ NUMBER_OF_COMPUTORS (676)`; below that it rounds to 0 and the whole fee folds to the charity address.

---

## 7. Procedures & functions

**Procedures** (index): `createPool(1)` · `registerAsset(2)` · `updateAsset(3)` · `updateWeight(4)`
· `setPoolAdmin(5)` · `depositDividend(6)` · `depositOperating(7)` · `setPlatformParams(8)` (owner)
· `TransferShareManagementRights(9)` · `registerAssets(10)` (batch, ≤16) · `addDividendCurrency(11)`
· `setFundingRoute(12)` · `setExcludedAddress(13)` (pool admin) · `setDistributionMode(14)` (owner)
· `setTargetMode(15)` · `addRecipients(16)` · `clearRecipients(17)` (all pool admin).

**Recipient-list distribution (`targetMode == LIST`):** instead of paying asset holders, a pool can
pay an **explicit, admin-uploaded wallet list**, pro-rata by weight. No QU is attached per wallet —
the pool's accumulated `pot` is distributed automatically by the normal engine (END_EPOCH or
streamed), exactly like a holdings pool; only the recipient set changes.

- **`setTargetMode(poolId, mode)`** — switch a pool between by-holdings (`0`, default) and by-list
  (`1`). Admin only.
- **`addRecipients(poolId, count, wallets[], weights[])`** — add/update (`weight > 0`) or remove
  (`weight == 0`) recipients. For an **equal split**, give every wallet the same weight (e.g. `1`);
  for pro-rata, weight them accordingly. Deduped by wallet and tagged with the pool's current
  `listVersion`. Up to `QREWARDS_MAX_RECIP_BATCH` (24) per call (bounded by `MAX_INPUT_SIZE = 1024`);
  call repeatedly for longer lists. A pool's list is **hard-capped at `QREWARDS_MAX_LIST_PER_POOL`
  (5,000)**: once full, further *new* wallets are skipped (`output.rejected` counts them, and
  `output.listCount` reports the live total), while existing entries can still be updated or removed.
  Because 5,000 is below `QREWARDS_SNAPSHOT_CAP` (16,384), **every listed recipient is always paid in
  full each epoch** — nobody is truncated. Weights are clamped to `QREWARDS_MAX_LIST_WEIGHT` (1e12) to
  keep the payout product within uint128.
- **`clearRecipients(poolId)`** — empties the list: it scans the shared map and physically removes
  the pool's entries (reclaiming their slots) and bumps `listVersion`. The scan is O(map size) but
  admin-initiated and rare; a bare version bump would leak slots permanently.

**Capacity — two limits.** (1) *Per pool:* a list is hard-capped at `QREWARDS_MAX_LIST_PER_POOL`
(5,000), enforced by a live `listCount` in `PoolMeta`. Since 5,000 < `SNAPSHOT_CAP` (16,384), a list
pool always pays everyone in full. (2) *Shared:* all list-mode pools share one `recipients` map of
`QREWARDS_MAX_RECIPIENTS` (2^20 = 1,048,576) entries, keyed by `K12(poolId, wallet)` — a shared total,
not per pool. At 5,000 each that comfortably covers ~209 full list pools; beyond that `addRecipients`
starts counting `rejected` (graceful, no corruption). At ~96 MB the map is the contract's largest
structure but well under the 1 GB state limit, so both caps can be raised later if needed. Removed
(`weight 0`) and cleared entries are reclaimed by `cleanupIfNeeded()` at `END_EPOCH`.

If a list-mode pool has no eligible recipients at distribution time (empty/cleared list), its pot is
simply retained and carries to the next epoch, same as a holdings pool with no eligible holders.

**Hardcoded in `INITIALIZE` (no setters):** `platformOwner` (the QPay wallet — not a NULL
first-caller bootstrap, and with **no transfer function**, so it is fixed for the life of the
deployment), `qpayhubAddress = id(29,0,0,0)`, and `qpayTokenDividendsAddress` (the QRaffle charity
wallet). The streamed delay (~24h) and batch size are fixed constants. The **only** owner-only
functions are `setPlatformParams` (fees) and `setDistributionMode` (END_EPOCH ⇄ streamed switch);
the fee-destination, stream-pacing, and ownership-transfer setters were all removed, so none of
those values can be repointed after deployment. A compromised owner key can at most change fee
levels or the distribution mode — it cannot redirect funds or seize the role; rotating the owner
requires a redeploy.

**Functions** (index): `getPool(1)` (admin, label, operatingBalance, `lastTotalWeight`, counts,
active/paused, and per-currency `assetName`/`pot`/`lifetime`) · `getPoolAsset(2)` ·
`getAllPoolAssets(3)` · `getPlatform(4)` · `getFundingRoute(5)` · `getPoolsByAdmin(6)` (paginated,
256/page) · `isExcluded(7)` · `previewWeight(8)`.

> **Breaking change vs. the old accumulator model:** `claimDividends`, `syncProfit`,
> `getPosition`, `getPositions`, and `previewProfit` are **removed**; `createPool` no longer takes
> a token `supply`. Front ends read `previewWeight` + `getPool().lastTotalWeight`/`currencyPot`
> instead of positions, and holders are paid automatically (no claim).

---

## 8. Funding a pool from outside

- **Direct:** `depositDividend(poolId, currencyIndex, amount)` — QU from the invocation reward,
  or `amount` of a token pulled from the caller (the caller must grant QREWARDS management of the
  token first, e.g. QX `TransferShareManagementRights`).
- **Tagged QU transfer:** `setFundingRoute(poolId)` maps the caller's address → a pool; thereafter
  a **plain QU transfer** from that address to the contract is auto-credited to the pool's QU pot
  by `POST_INCOMING_TRANSFER` (standard/qpi transfers only — procedure transfers are handled by
  `depositDividend`, so they are skipped to avoid double counting). This lets even lower-index
  contracts fund a pool with no cross-contract procedure call.

**How fees reach QPAYHUB:** QPAYHUB (qubic/core PR #1015, `src/contracts/QPayhub.h`, CONTRACT_INDEX
29) has no deposit procedure — its `POST_INCOMING_TRANSFER` auto-credits plain QU into its `feePool`.
So `qpi.transfer(qpayhubAddress, …)` from QREWARDS is all that's needed; set
`qpayhubAddress = id(29,0,0,0)`.

---

## 9. Index & deployment

QPAYHUB occupies **CONTRACT_INDEX 29**, index 30 is another already-deployed contract, so **QREWARDS
is wired at index 31**. Indices 29/30 are empty placeholder contracts (`QRewardsReserved29.h` /
`QRewardsReserved30.h`) purely to keep the positional contract array consistent. The contract share
asset ticker is `"QREWARD"` (≤7 chars).

---

## 10. Testing notes

Unit tests live in `test/contract_qrewards.cpp`. The authoritative test build is **MSVC on Windows**
(`.github/workflows/build-tests.yml`); the platform layer uses MSVC intrinsics, so a Linux clang/gcc
build of the full test target is not supported — **rely on CI for compile-verification.** The tests
cover: weight computation, QU/token distribution at epoch end, the **sell-then-rebuy no-double-count**
case, pro-rata splits, excluded addresses, the QU and token dividend-fee splits, operating-fee pausing
(which stops distribution), funding routes, and admin/owner gating.
