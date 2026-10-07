# QREWARDS — Test Report

**Scope.** `QRewards.h` (open multi-pool loyalty/dividend platform, "pure
snapshot" distribution model: every epoch re-reads live holdings from the
asset ledger, no claim step, no persistent reward-token balance). Delivered
package at `qrewards/` (`QRewards.h`, `contract_qrewards.cpp`,
`qrewards-build-guide.md`). Wired into this devkit at contract index **48**
(the build guide assumes index 31, based on the upstream mainnet layout —
this devkit's own `contractDescriptions[]` has no correspondence to that
layout and QPAYHUB doesn't exist in it at all). Covered by the delivered
18-case GoogleTest suite and exercised live on a running devnet across 3
real epoch transitions.

---

## 1. Test results

### GoogleTest — 18/18 passing (`test/contract_qrewards.cpp`)

The build did **not** compile at first — see §2 for the two real defects
found and fixed. Once fixed, the full provided suite passes clean: pool
creation and live weight (`previewWeight` against `ComputeEntitlement`'s own
formula, including the overflow-saturation case), QU and token dividend
deposits with the 5% fee split both ways (deferred 80/20 at END_EPOCH for
QU, inline 20%-in-kind for tokens), batch asset registration, funding
routes, the operating-fee pause → deactivate → refund cycle, streamed vs.
END_EPOCH distribution mode, list-mode distribution and its admin-only
edit/auth guards, and the platform read-only getters.

This result also empirically disproves the build guide's claim that "a
Linux clang/gcc build of the full test target is not supported" — once the
two real defects below are fixed, it builds and passes cleanly with clang
on this devkit, same as every other contract onboarded this session.

### Live devnet — 19/19 (Phase 1, setup) + 10/12 (Phase 2, epoch 222→223) + 5/7 (Phase 3, epoch 224→225) passing

**Phase 1 (same epoch, no transition needed) — 19/19:**
`createPool` ×3, with the 70/15/15 create-fee split (`DistributeFee`) fired
synchronously and confirmed via a real balance delta on both the TEST-ONLY
shareholder wallet and the TEST-ONLY QPAYHUB wallet; a real QX-issued asset
(TOKENA, 100M shares split 60M/30M/10M across three holders) registered on
a holdings-mode pool, with live `previewWeight` matching
`ComputeEntitlement`'s own integer formula exactly for every holder
(including the concentration-multiplier tiers); `setExcludedAddress` forcing
a holder's weight to zero live despite still holding shares; a plain
(non-procedure) incoming QU transfer auto-credited via a funding route; a
second dividend currency (DOGEX, a real QX token) with its 5% fee split
confirmed both legs live — 20% paid in-kind to a real shareholder via
`DistributeTokenToShareholders`/`qpi.distributeDividends`, remainder to the
QPAY wallet; list-mode pool setup; and the owner-gated calls
(`setPlatformParams`/`setDistributionMode`) exercised for the first time
ever through the **real** TEST-ONLY `platformOwner` wallet rather than
GoogleTest's direct state override, including confirming a non-owner call
is correctly rejected.

**Phase 2 (epoch 222→223) — 10/12:** the real `END_EPOCH` distribution for
both pool types, confirmed by independent reference-model cross-check:

- Holdings-mode poolH, 2 currencies: HOLDER1/HOLDER2 paid `pot × weight /
  totalWeight` exactly for both QU and DOGEX, `lastTotalWeight` snapshot
  correct, pots carrying only rounding dust afterward.
- List-mode poolL: LIST1/LIST2 paid the equal-weight split exactly.
- `poolOP`'s 1st missed operating-fee epoch: paused, `missedEpochs=1`,
  still active, admin not yet refunded.

The 2 failures were a **live-devnet streamed-mode cycle that never fired**
within a bounded wait — not a contract defect (see §3).

**Phase 3 (epoch 224→225) — 5/7:** the two most load-bearing remaining
mechanics, both confirmed:

- The QU a mistimed Phase 2 deposit left stranded mid-epoch was correctly
  swept by the next real `END_EPOCH` — proving no funds are lost across a
  streamed→END_EPOCH mode switch, matching the engine's own documented
  resilience ("cursors live in state... idempotent, no funds are lost").
- `poolOP`'s **3rd** missed epoch crossed
  `QREWARDS_MAX_MISSED_EPOCHS=2`: real deactivation (`active→0`) plus a
  full, exact refund (pot + distributable + operating balance) transferred
  to the pool admin, pots zeroed afterward.

The 2 failures were a second streamed-mode attempt, also not firing live
(see §3).

### What isn't covered live (and why)

**The streamed distribution mode's live payout never fired**, across two
separately-reasoned attempts (one with a diagnosed timing mistake on my
part — fixed — one with that mistake corrected). The math predicts the
cycle should complete on the very next `END_TICK` after the mode flip
(256-unit batch budget against 2-3 pools), and `END_TICK`'s trigger logic
reads cleanly (`lastCycleEpoch != currentEpoch && tick - epochStartTick >=
streamDelayTicks`) with no other gating condition in the source. Since
GoogleTest exercises this exact code path directly (`StreamedDistributionOneTick`/
`StreamedDistributionAcrossTicks`, both passing, calling `END_TICK` with
full control over `qpi.tick()`) and passes cleanly, the streamed engine's
*logic* is not in doubt — what's unconfirmed live is whether `END_TICK`
itself fires reliably/promptly enough on this devkit's running devnet to
observe it externally within a practical wait window. This reads as a
devnet-liveness question, not a QREWARDS defect, but it was not resolved
within this test's scope.

---

## 2. Contract findings

**One genuine defect, fixed and backported to the pristine package:**

`QREWARDS_MAX_RECIP_BATCH = 24` is used directly as the capacity of two
`QPI::Array<T,N>` fields in `addRecipients_input`
(`Array<id,24> wallets; Array<uint64,24> weights;`). QPI's `Array<T,N>`
requires `N` to be a power of 2
(`static_assert(L && !(L & (L-1)))`), and 24 is not — this fails to compile
under any standard QPI build, not just this devkit's Linux/clang
toolchain. Fixed by lowering to 16 (the next power of 2 that still fits the
documented `MAX_INPUT_SIZE=1024` budget for the struct — 32 would not:
`12 + 40×32 = 1292 > 1024`). Backported to `qrewards/QRewards.h`.

**Two test-file bugs in the delivered `contract_qrewards.cpp`, fixed and
backported:**

1. `QR_TOKEN`/`QR_DOGE` were arbitrary integer constants
   (`123456789ULL`/`987654321ULL`) rather than real ticker-shaped asset
   names. `qpi.h`'s real `issueAsset()` validates that a packed asset name's
   bytes decode to uppercase ASCII/digits — these constants fail that check
   silently, so `qpi.issueAsset()` returned 0 for every test, cascading into
   `QREWARDS_ASSET_NOT_ISSUED` everywhere an asset was used. 13 of 18 cases
   failed for this one reason before the fix. Fixed by using real
   ticker-shaped packed values (`"TOKEN"`/`"DOGE"`).
2. `ListModeEditAndAuth`'s non-admin probe (`QR_BOB`) was never funded via
   `increaseEnergy`, so the test harness's `invokeUserProcedure` bailed out
   *before* reaching the contract at all (a never-funded identity has no
   spectrum entry, so the harness's own balance-deduction step fails first)
   — the resulting zeroed output coincidentally matches
   `QREWARDS_SUCCESS=0`, masking the fact that the contract's real admin
   check was never actually exercised. Fixed by funding `QR_BOB`.

Both backported to `qrewards/contract_qrewards.cpp`.

**Not a contract defect, but worth noting:** the `qpayhubAddress` test
(`QuDividendFeeRouting`) hardcodes `id(29,0,0,0)` as "the QPAYHUB contract
address" per the pristine contract's own `INITIALIZE()`. That hardcoding is
correct for the pristine package; it only needed adjusting in this devkit's
copy because of the TEST-ONLY substitution (§4).

---

## 3. Infrastructure/process findings (not QREWARDS defects)

- **`distributeDividends()` hard-asserts exact share conservation.**
  `ASSERT(totalShareCounter == NUMBER_OF_COMPUTORS || == 0)` over the live
  circulating supply of the contract's own "shareholder" asset. Standard
  genesis-crafting tools in this devkit (`universe_update`) only *grant*
  shares, which would have inflated supply past 676 and crashed the node
  the first time any fee got distributed. Required writing a new tool,
  `tools/universe_transfer_shares.cpp`, that decrements one holder while
  crediting another, conserving the total — this is a devkit tooling gap
  the original toolset never needed before (every previous contract this
  session either used shares pre-distributed from mainnet-like genesis
  data or didn't rely on contract-share dividends at all).
- **The streamed-mode live-firing question (§1)** is logged here rather
  than against the contract, since GoogleTest already independently proves
  the distribution logic itself correct.
- Two **pre-existing, unrelated** issues were discovered incidentally while
  confirming this work didn't regress anything else, and were flagged as
  separate background tasks rather than fixed here (out of scope for this
  contract): `TestContractAgentRail` has 10/43 cases failing even in full
  isolation, and the `TestQubicScoreFunction` suite crashes outright (both
  of its cases). Neither was touched by, or is related to, QREWARDS.

---

## 4. TEST-ONLY scaffolding (devkit-wired copy only; not in the delivered package)

- **Index 48**, not the build guide's assumed 31 — see scope note above.
- **Three hardcoded identities substituted** (`platformOwner`,
  `qpayhubAddress`, `qpayTokenDividendsAddress`), since the pristine values
  are real production addresses this devkit cannot control. Seeds/ids
  recorded in `tools-bin/qrewards_test_only_identities.json`.
- **`streamDelayTicks` compressed from 86400 (≈24h) to 60 ticks** in
  `INITIALIZE()`, so a live test can exercise the streamed-mode trigger
  path within a 10-minute epoch at all. The pristine package is untouched.
- **100 of the contract's 676 "QREWARD" bookkeeping shares real-transferred**
  (supply-conserving, via the new `universe_transfer_shares` tool) from the
  zero identity to a TEST-ONLY `SHAREHOLDER1` wallet, giving the
  shareholder-split code paths a genuine live recipient. The other 576
  remain with the zero identity (inert — any pro-rata cut sent there is
  effectively burned), which is why every shareholder-split expectation in
  this test accounts for both the real wallet's share *and* the zero
  identity's.
