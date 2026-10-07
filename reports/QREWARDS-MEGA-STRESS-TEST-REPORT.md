# QREWARDS Mega-Stress Test Report

Follow-up to [QREWARDS-TEST-REPORT.md](QREWARDS-TEST-REPORT.md) (the first-ever
single-instance live test). This round scales QREWARDS up to a 20-pool, 64-asset,
40-wallet live devnet scenario spanning 10 real epoch transitions in END_EPOCH
mode followed by a switch to the global streamed-distribution mode, with every
contract procedure and function exercised multiple times, in multiple ways,
against real on-chain state — not GoogleTest mocks.

## Scope

| | |
|---|---|
| Pools | 20, each registering the maximum 64 shared real QX-issued assets |
| Pool roles | 14 NORMAL · 2 STALE_DELETE · 2 STALE_RECOVER · 2 LIST_MODE |
| Wallets | 40 (20 pool admins, issuer, 4 holders, funder, 10 list recipients, spares) |
| Assets | 64 real QX-issued tokens (ASSET01–ASSET64), distributed ISSUER 3% / HOLDER1 55% / HOLDER2 25% / HOLDER3 12% / HOLDER4 5% each |
| Epoch transitions | 10 in END_EPOCH mode (222→232), then 2 streamed cycles (232, 233) |
| Checks run | 143 (137 + 6), **0 failures** |

## The one genuine finding: admin-wallet spectrum pruning after `createPool`

`createPool` sweeps **100%** of its invocation reward into the create-fee split
and the pool's `operatingBalance` — it refunds nothing to the admin
([QRewards.h:764-771](../core-lite/src/contracts/QRewards.h#L764)):

```cpp
locals.dfi.amount = state.get().createPoolFee;
CALL(DistributeFee, locals.dfi, locals.dfo);
locals.meta.operatingBalance = (uint64)qpi.invocationReward() - state.get().createPoolFee;
```

The mega-test's rebuild script funded each admin wallet with *exactly* the
amount it sends in its `createPool` call (create fee + operating margin).
That left every admin at **0 QU** immediately after pool creation. On this
devkit, a 0-balance identity gets pruned from the spectrum — so every later
admin-authenticated call (`registerAssets`, `updateAsset`, `setDistributionMode`,
`addRecipients`, ...) silently had an invalid, nonexistent sender and never
reached the contract at all.

This produced a confusing symptom chase: `registerAssets` calls appeared to
broadcast successfully (real tx hashes, no HTTP errors) but 6 sampled pools
all showed `0` registered assets. Three rounds of **timing** fixes (tick-offset
tuning, chunked batching, sequential-incrementing ticks) all failed to help,
because timing was never the problem. Root cause was only confirmed by adding
temporary `debugGetLast()` instrumentation directly into the contract's
`registerAssets` procedure body: `callCount` stayed at `0` even for a single,
generously-timed, otherwise-correct call — meaning the transaction never
reached the loop body at all, consistent with an invalid sender, not a late
arrival.

**This is a test-infrastructure finding, not a QREWARDS contract defect** — a
real admin funding a real pool would not transfer their entire wallet balance
as the call's `amount`. It's the same class of bug already seen and fixed once
before in this session's QPOOL mega-test (rebuild-script funding, not contract
logic). The fix: an `ADMIN_RESERVE` (1,000,000 QU) added on top of what each
admin sends to `createPool`, so the wallet survives as a valid sender for the
rest of the test (`rebuild_genesis_qrewards_mega_test.sh`). The same issue was
found and fixed for the TEST-ONLY `platformOwner` identity, which needed to
remain a valid sender to later flip the global distribution mode.

The temporary `debugGetLast()` diagnostic (4 state fields, 2 write-sites, 1
function) has been **fully removed** from the devkit-wired `QRewards.h` and the
GoogleTest file; a clean rebuild afterward reconfirmed all 18 `ContractQRewards`
GoogleTest cases pass.

## Phase 1 — setup

64 assets issued and distributed, 20 pools created, all 64 assets registered
into every pool (`qrewards_mega_setup.py`). Once the funding fix landed, this
ran cleanly on the first try: `[PASS] sampled pools (first 3 + last 3) all show
64 registered assets -- got [64, 64, 64, 64, 64, 64]`, finishing in under 5
minutes of wall-clock (well inside one ~10-minute epoch).

## Phase 2 — 10 END_EPOCH transitions, every function exercised

`qrewards_mega_epoch_loop.py`, 137/137 checks passed. Three of the ten
transitions happened naturally during setup/diagnostics; the script picked up
live mid-epoch and ran the remaining seven, recomputing its own epoch budget
dynamically rather than assuming a fixed start point.

Confirmed live, with real on-chain state checks after every transition:

- **QU dividends every epoch** into all 16–18 live pools (`depositDividend`),
  **periodic token dividends** (`ASSET01`) every third epoch, from a real
  holder's own asset balance.
- **Stale → deleted**: pools 14/15, deliberately funded with zero operating
  margin, correctly went `paused=1` after each missed epoch and were
  **deactivated** exactly when `missedEpochs` exceeded `QREWARDS_MAX_MISSED_EPOCHS`
  (the 3rd consecutive miss) — confirmed at epoch 226, and confirmed to *stay*
  deactivated through the rest of the run.
- **Stale → recovered**: pools 16/17, started the same way, but topped up via
  `depositOperating` from the FUNDER wallet before their 3rd miss. Confirmed
  live: `missedEpochs` reset to `0`, `paused=0`, and they rejoined the normal
  dividend-paying rotation for the rest of the test, accumulating real pot
  balances from subsequent deposits.
- **`addDividendCurrency` + `CURRENCIES_FULL` boundary**: pool1 pushed to
  `QU + 3 tokens` (the max, `QREWARDS_MAX_DIV_CURRENCIES=4`); a 5th attempt
  correctly left `numCurrencies` at 4.
- **`registerAsset` (singular) + `REGISTRY_FULL` boundary**: every pool is
  already at the 64-asset max, so a fresh singular registration attempt on
  pool3 correctly left `numAssets` at 64.
- **`setPoolAdmin` rotation**: pool5's admin moved from `ADMIN05` to a fresh
  `NEWADMIN` wallet; the *rotated* admin then successfully called
  `updateWeight` on the same pool, confirming re-authentication works
  immediately after a rotation.
- **`setFundingRoute` + plain-transfer `POST_INCOMING_TRANSFER`**: a plain QU
  transfer (not a procedure call) to the QREWARDS contract address correctly
  auto-credited pool6's QU pot at exactly 95% of the transferred amount (5%
  dividend fee skimmed), then the route was cleared.
- **`setExcludedAddress`**: HOLDER4 excluded from pool7 (confirmed via
  `previewWeight` dropping to 0 and `isExcluded` reporting true), then
  re-included later (weight restored).
- **`clearRecipients` + `addRecipients`**: one list-mode pool's recipient list
  was cleared and re-uploaded with a different wallet set and weights mid-test.
- **Full read-only sweep**: `getPool`, `getPoolAsset`, `getAllPoolAssets`,
  `getPlatform`, `getFundingRoute`, `getPoolsByAdmin`, `isExcluded`,
  `previewWeight` all called and cross-checked against expected state
  repeatedly across the loop.

## Phase 3 — switch to streamed distribution

`qrewards_mega_streamed_phase.py`, 6/6 checks passed. This closes the gap the
single-instance test's report left open ("streamed-mode live confirmation...
documented limitation").

- **Owner gate confirmed both ways**: a non-owner (`ADMIN00`) attempting
  `setDistributionMode(1)` left the platform at `distributionMode=0`; the real
  TEST-ONLY `platformOwner` wallet's call correctly flipped it to `1`.
- **A real streamed cycle ran and paid real holders**, within the *same*
  epoch it was switched (the devkit's TEST-ONLY `streamDelayTicks=60` window
  had already elapsed). Five pools' QU pots (950,000 QU deposited moments
  earlier, still in END_EPOCH mode) all drained to exactly 0, and holder QU
  balances increased by real, verifiable amounts matching the expected
  concentration-tier split:

  | Holder | QU gained |
  |---|---|
  | ISSUER | 62,815 |
  | HOLDER1 | 1,718,531 |
  | HOLDER2 | 649,627 |
  | HOLDER3 | 314,306 |
  | HOLDER4 | 104,731 |

- **`cycleActive` cleared** correctly once the cycle finished.
- **A second cycle on the following epoch**, seeded with a fresh round of
  deposits, also ran and drained all test pools' pots to 0 — confirming the
  once-per-epoch cursor (`lastCycleEpoch`) resets cleanly across a real epoch
  boundary.

## Own-script bugs caught along the way (not contract issues)

- A `struct.unpack("<Q", b"ZZTEST1")` call in the epoch-loop script required
  exactly 8 bytes but got 7, crashing the script mid-run. Caught, fixed
  (padded to 8 bytes like the rest of the asset-name encoders), and the script
  was resumed *without* re-running the already-landed, non-idempotent
  `addDividendCurrency` calls — confirmed via live state query before
  resuming that nothing had been double-applied.
- An early `getPlatform_output` byte-offset mistake (missing 3 bytes of
  padding between the `uint8 distributionMode` and the following
  `uint32 streamDelayTicks`) was caught via a direct raw-byte probe before it
  could produce a wrong result.

## Scope notes (not covered this round, not failures)

- `setPlatformParams` (fee changes) and `TransferShareManagementRights` (QX
  relay) were exercised in the single-instance test but not repeated here.
- `addDividendCurrency`'s `ASSET_NOT_ISSUED` rejection path wasn't explicitly
  tried this round.
- Streamed-mode verification checked QU pots only; the periodic token
  dividends' streamed payout wasn't separately isolated and verified.
- Reverting from streamed back to END_EPOCH mode mid-cycle wasn't tested.

## TEST-ONLY scaffolding (disclosed, not part of the delivered contract)

- 40-wallet roster + 4-way pool-role mapping, all generated for this test
  (`qrewards_mega_wallets.json`, `qrewards_mega_pool_roles.json`).
- `platformOwner` / `qpayhubAddress` / `qpayTokenDividendsAddress` identities
  substituted in `INITIALIZE()` (pre-existing from the single-instance test,
  reused here) — the pristine delivered `QRewards.h` hardcodes real addresses.
- `streamDelayTicks` compressed from 86400 (~24h) to 60 (pre-existing,
  reused) — devkit-wired copy only.
- QREWARDS wired at contract index 48.
- The temporary `debugGetLast()` diagnostic used mid-investigation has been
  fully reverted; see "the one genuine finding" above.

## Verification

- `rebuild_genesis_qrewards_mega_test.sh` — idempotent genesis rebuild with
  the funding fix, runs clean every time.
- `qrewards_mega_setup.py` — 2/2 checks passed.
- `qrewards_mega_epoch_loop.py` — 137/137 checks passed.
- `qrewards_mega_streamed_phase.py` — 6/6 checks passed.
- `ContractQRewards.*` GoogleTest suite — 18/18 passed after the debug-code
  revert and a clean rebuild.
