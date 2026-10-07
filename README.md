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

## Fees

**Dividend fee** — 5% of every deposit into a pool (QU or token), split 80/20:
| Share | Goes to |
|---|---|
| 20% | QREWARDS shareholders |
| 80% | QPAYHUB's dividend account (QU deposits) or the QRaffle charity address (token deposits) |

95% of every deposit reaches the pool's holders.

**Pool creation fee** — 5,000,000 QU default, split 70/15/15:
| Share | Goes to |
|---|---|
| 70% | QREWARDS shareholders |
| 15% | QPAYHUB address |
| 15% | Burned |

Any amount paid above the fee seeds the new pool's operating balance.

**Operating fee** — 25,000 QU per pool per epoch (~weekly, on mainnet epoch
cadence), same 70/15/15 split as above. Drawn automatically from the pool's
operating balance (topped up via `depositOperating`). A pool that falls
behind is paused for up to 2 missed epochs, then deactivated — on
deactivation, all remaining pots and leftover operating balance are
refunded to the pool admin, so nothing is stranded.

**Fee caps.** The platform owner can tune the creation and operating fees
via `setPlatformParams`, but both are hardwired to a ceiling equal to
today's default — 5,000,000 QU and 25,000 QU respectively. In practice this
means the owner can only ever lower these fees from here, never raise them:
any call that would push a fee above its current value is rejected outright.

## Contents

- [`QRewards.h`](QRewards.h) — the contract.
- [`contract_qrewards.cpp`](contract_qrewards.cpp) — GoogleTest suite.
- [`reports/`](reports) — live-devnet test reports.

## Testing

Unit tests live in `contract_qrewards.cpp`. The authoritative build is CI on
Windows/MSVC (platform layer uses MSVC intrinsics). The `reports/` directory
has results from running the contract live against a real Qubic devnet
(pools, dividends, fee caps, NFT-weighted pools, etc).
