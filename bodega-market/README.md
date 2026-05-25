# Bodega Market

tx3 protocol for [Bodega Market](https://github.com/bodega-market/bodega-market-smart-contracts-v2) — prediction markets on Cardano.

> **Status: WIP** — This tx3 off-chain implementation is under testing and may not cover all edge cases yet.

## Overview

Bodega Market is a prediction market protocol where users buy positions on outcomes (YES/NO), then claim rewards or refunds after market resolution via oracle. It uses an AMM pricing model and a licensed batcher for processing positions.

## Transactions

| Transaction | Description |
|---|---|
| `buy_position_yes` / `buy_position_no` | Buy a prediction position (BuyPos) on a candidate outcome |
| `submit_reward_yes` / `submit_reward_no` | Submit share tokens for reward claim after market resolution |
| `sell_position_yes` / `sell_position_no` | Sell a position back (RefundPos) at current AMM price |
| `create_market` | Deploy a new prediction market (admin operation) |

## Important Considerations

- **Duplicate YES/NO variants:** Transactions are split into `_yes`/`_no` variants because `trix invoke` does not support enum parameters. They differ only in the `CandidateIdx` value.
- **Position submissions are script-free:** `buy_position`, `submit_reward`, and `sell_position` are plain payments to the position script address — no validators execute. A licensed batcher later processes them.
- **On-chain datum divergence:** The deployed contract has 9 datum fields (not 7 as in the public V2 GitHub source). Fields `pos_admin_fee_percent` and `pos_unit_price` are undocumented in the public code but required on-chain.
- **Constructor order swap:** The deployed contract swaps `Reward`/`Refund` constructors vs the GitHub source. Constructor 2 = RewardPos on-chain (confirmed via transaction analysis).
- **Pre-computed total lovelace:** `buy_position` requires the caller to pre-compute `total_lovelace` because tx3 lacks `*` and `/` operators. Formula: `amount * unit_price + amount * admin_fee_percent * 1_000_000 / 10_000 + batcher_fee + envelope_amount`.
- **Create market:** `create_market` is an admin-only operation that mints auth NFTs via an inline PlutusV3 witness, requires BODEGA pledge tokens, and pays an open fee to the protocol treasury.
- **Per-market configuration:** Each market has its own `share_policy_id`, `PositionScript` address, and auth token policy. These are passed as transaction parameters rather than environment variables.

## Caller Preparation

All per-market values live in two on-chain UTxOs. The caller must query them before invoking any transaction.

### Step 1: Query the ProjectInfoScript UTxO

Query the `ProjectInfoScript` address for the UTxO holding the `PROJECT_INFO_NFT`. Its inline datum (`ProjectInfoDatum`, 17 fields) contains the market configuration:

| Datum field | Used as | Needed for |
|---|---|---|
| `outref_id` (tx_hash + output_index) | `project_outref_tx`, `project_outref_idx` | All txs — identifies the market |
| `position_script_hash` | `PositionScript` party address | All txs — where positions are sent |
| `pi_share_policy_id` | `share_policy_id` param | `submit_reward`, `sell_position` |
| `admin_fee_percent` | `admin_fee_percent` param | `buy_position`, `sell_position` |
| `pi_envelope_amount` | `envelope_amount` param | `submit_reward`, `sell_position`, and `total_lovelace` calc |
| `candidate_yes_name` / `candidate_no_name` | `candidate_name` param | `submit_reward` |

### Step 2: Query the PredictionScript UTxO

Query the `PredictionScript` address for the UTxO holding the `PROJECT_PREDICTION_NFT` with the same `outref_id`. Its inline datum (`PredictionDatum`) contains the AMM state:

| Datum field | Used as | Needed for |
|---|---|---|
| `yes_price` | `unit_price` (for YES variant) | `buy_position_yes`, `sell_position_yes` |
| `no_price` | `unit_price` (for NO variant) | `buy_position_no`, `sell_position_no` |

### Step 3: Compute `total_lovelace` (for `buy_position`)

tx3 lacks `*` and `/` operators, so the caller must pre-compute:

```
total_lovelace = amount * unit_price
               + amount * admin_fee_percent * 1_000_000 / 10_000
               + batcher_fee
               + envelope_amount
```

### Summary: where each param comes from

| Parameter | Source |
|---|---|
| `user_pkh`, `user_stake_key` | Caller's wallet |
| `project_outref_tx`, `project_outref_idx` | `ProjectInfoDatum.outref_id` |
| `buy_amount`, `share_amount` | User's intent |
| `batcher_fee_amount` | `ProjectInfoDatum` (typically 700000 lovelace) |
| `admin_fee_percent` | `ProjectInfoDatum.admin_fee_percent` |
| `unit_price` | `PredictionDatum.yes_price` or `no_price` |
| `total_lovelace` | Computed (see formula above) |
| `envelope_amount` | `ProjectInfoDatum.pi_envelope_amount` |
| `share_policy_id` | `ProjectInfoDatum.pi_share_policy_id` |
| `candidate_name` | `ProjectInfoDatum.candidate_yes_name` or `candidate_no_name` |
| `PositionScript` party | Address derived from `ProjectInfoDatum.position_script_hash` |

## tx3 Limitations

Several tx3 language limitations affect this protocol. **5 batcher transactions cannot be implemented** due to dynamic input/output list limitations. For the full list, see [tx3-limitations-bodega-market.md](tx3-limitations-bodega-market.md).

## CBOR Verification

All user-facing transactions have been compared against real on-chain transactions. See `invoke-args/comparison-report.md` for the full field-by-field comparison.

## Smart Contracts

- PlutusV2 (V2 contracts)
- Source: [bodega-market/bodega-market-smart-contracts-v2](https://github.com/bodega-market/bodega-market-smart-contracts-v2)
