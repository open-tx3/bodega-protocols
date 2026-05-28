# Bodega Market

[Bodega](https://bodega.market/) is a decentralised prediction-market protocol on Cardano. Users buy positions on binary outcomes (YES / NO) of real-world events — election results, sports outcomes, financial milestones — and, once the event resolves via oracle, redeem winning shares for the underlying payout token (ADA on most markets). Liquidity providers and project teams can deploy new markets and earn fees from trading and resolution flows.

Each market is its own deployment of the same contract suite: a `ProjectInfoScript` UTxO carries the static market configuration, a `PredictionScript` UTxO carries the AMM state, and per-user position UTxOs sit at a market-specific `PositionScript`. The Bodega web app and API expose the active markets and pool state.

This tx3 covers the **user-facing flow** end-to-end: buying positions, selling positions, submitting winning shares for reward, and deploying a new market (admin operation). The batcher-side processing (the actual fills, refunds, and reward distributions) is not implementable in tx3 today.

## Overview

Bodega uses an AMM pricing model with a licensed-batcher execution layer. The user side is structurally simple: every position-buy, position-sell, and reward-submit transaction is a plain payment to the `PositionScript` address with an inline datum — no validators run on submission. A licensed batcher later consumes the queued position UTxOs together with the `PredictionScript` UTxO to apply the AMM, update pool state, and pay out users.

Scope of this tx3:

- Implemented: user-facing buy, sell, reward submit; admin market creation.
- Not implemented (blocked by tx3 dynamic input/output lists): batcher fills, batcher refunds, batcher reward distribution, admin fee withdrawal, market close.

## Transactions

| Transaction | Description |
|---|---|
| `buy_position_yes` / `buy_position_no` | Buy a prediction position (BuyPos) on a candidate outcome |
| `submit_reward_yes` / `submit_reward_no` | Submit share tokens for reward claim after market resolution |
| `sell_position_yes` / `sell_position_no` | Sell a position back (RefundPos) at the current AMM price |
| `create_market` | Deploy a new prediction market (admin operation) |

## Important considerations

- **Duplicate YES / NO variants.** Transactions are split into `_yes`/`_no` variants because `trix invoke` does not support enum parameters. They differ only in the `CandidateIdx` value.
- **Position submissions are script-free.** `buy_position`, `submit_reward`, and `sell_position` are plain payments to the `PositionScript` address — no validators execute. A licensed batcher processes them later.
- **On-chain datum divergence.** The deployed contract has 9 datum fields (not 7 as in the public V2 GitHub source). Fields `pos_admin_fee_percent` and `pos_unit_price` are undocumented in the public code but required on-chain.
- **Constructor order swap.** The deployed contract swaps `Reward` / `Refund` constructors vs. the GitHub source. On-chain, constructor 2 = `RewardPos` (confirmed via transaction analysis).
- **Pre-computed total lovelace.** `buy_position` requires the caller to pre-compute `total_lovelace` because tx3 lacks `*` and `/`. Formula: `amount * unit_price + amount * admin_fee_percent * 1_000_000 / 10_000 + batcher_fee + envelope_amount`.
- **`create_market` is admin-only.** Mints auth NFTs via an inline PlutusV3 witness, requires BODEGA pledge tokens, and pays an open fee to the protocol treasury.
- **Per-market configuration.** Each market has its own `share_policy_id`, `PositionScript` address, and auth-token policy. These are passed as transaction parameters rather than environment variables.
- **`admin_fee_percent` is per-position.** On-chain `pos_admin_fee_percent` can differ from `ProjectInfoDatum.admin_fee_percent` within the same market (likely a BODEGA holder discount mechanism), so it must remain a caller param even though the project-info datum is readable.

## Caller preparation

All per-market values live in two on-chain UTxOs. The caller must query them before invoking any transaction.

### Step 1 — Query the `ProjectInfoScript` UTxO

Query the `ProjectInfoScript` address for the UTxO holding the `PROJECT_INFO_NFT`. Its inline datum (`ProjectInfoDatum`, 17 fields) contains the market configuration.

| Datum field | Used as | Needed for |
|---|---|---|
| `outref_id` (tx_hash + output_index) | `project_outref_tx`, `project_outref_idx` | All txs — identifies the market |
| `position_script_hash` | `PositionScript` party address | All txs — where positions are sent |
| `pi_share_policy_id` | `share_policy_id` param | `submit_reward`, `sell_position` |
| `admin_fee_percent` | `admin_fee_percent` param | `buy_position`, `sell_position` |
| `pi_envelope_amount` | `envelope_amount` param | `submit_reward`, `sell_position`, and `total_lovelace` calc |
| `candidate_yes_name` / `candidate_no_name` | `candidate_name` param | `submit_reward` |

### Step 2 — Query the `PredictionScript` UTxO

Query the `PredictionScript` address for the UTxO holding the `PROJECT_PREDICTION_NFT` with the same `outref_id`. Its inline datum (`PredictionDatum`) contains the AMM state.

| Datum field | Used as | Needed for |
|---|---|---|
| `yes_price` | `unit_price` (YES variant) | `buy_position_yes`, `sell_position_yes` |
| `no_price` | `unit_price` (NO variant) | `buy_position_no`, `sell_position_no` |

### Step 3 — Compute `total_lovelace` (for `buy_position`)

tx3 lacks `*` and `/`, so the caller must pre-compute:

```
total_lovelace = amount * unit_price
               + amount * admin_fee_percent * 1_000_000 / 10_000
               + batcher_fee
               + envelope_amount
```

### Summary — where each param comes from

| Parameter | Source |
|---|---|
| `project_outref_tx`, `project_outref_idx` | `ProjectInfoDatum.outref_id` |
| `batcher_fee_amount` | `ProjectInfoDatum` (typically 700 000 lovelace) |
| `admin_fee_percent` | `ProjectInfoDatum.admin_fee_percent` (or per-position discount value) |
| `unit_price` | `PredictionDatum.yes_price` or `no_price` |
| `total_lovelace` | Computed (see formula above) |
| `envelope_amount` | `ProjectInfoDatum.pi_envelope_amount` |
| `share_policy_id` | `ProjectInfoDatum.pi_share_policy_id` |
| `candidate_name` | `ProjectInfoDatum.candidate_yes_name` or `candidate_no_name` |
| `PositionScript` party | Address derived from `ProjectInfoDatum.position_script_hash` |

## References

- **Smart contracts:** PlutusV2 — [bodega-market/bodega-market-smart-contracts-v2](https://github.com/bodega-market/bodega-market-smart-contracts-v2)
- **Homepage / app:** [bodega.market](https://bodega.market/)
