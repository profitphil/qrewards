# QREWARDS

An open, multi-pool loyalty & dividend contract for Qubic.

Anyone can create a **pool**. Holding that pool's registered assets earns a
weighted share of its dividends. Anyone can feed QU or tokens into a pool via
`depositDividend`; the pot pays out pro-rata at each epoch end. Pools are
fully isolated — a pool can only ever pay out what was deposited to it.

## How it works

- **Pure snapshot, no claim step.** At `END_EPOCH`, each pool reads live
  holdings straight from the asset ledger, computes every holder's weight,
  and pays out automatically. No stored balances, no stale state, no
  `claimDividends`.
- **Weight** = `(held / unit) × concentrationMultiplier × weightBps`, summed
  across a pool's registered assets. The multiplier rises with holding size,
  so splitting a balance across wallets never increases total weight.
- **List mode.** A pool can instead pay an explicit admin-uploaded wallet
  list rather than asset holders.
- **NFT-weighted pools.** A pool can also weight by held NFTs from a
  registered collection, via an admin-curated catalog with round-robin
  snapshot spreading for pools with many NFTs.
- **Fees.** 5% on every dividend deposit, 5,000,000 QU to create a pool,
  100,000 QU/epoch operating fee — all platform fees are capped and only
  adjustable downward from their default by the platform owner.

## Contents

- [`QRewards.h`](QRewards.h) — the contract.
- [`contract_qrewards.cpp`](contract_qrewards.cpp) — GoogleTest suite.
- [`reports/`](reports) — live-devnet test reports.

## Testing

Unit tests live in `contract_qrewards.cpp`. The authoritative build is CI on
Windows/MSVC (platform layer uses MSVC intrinsics). The `reports/` directory
has results from running the contract live against a real Qubic devnet
(pools, dividends, fee caps, NFT-weighted pools, etc).
