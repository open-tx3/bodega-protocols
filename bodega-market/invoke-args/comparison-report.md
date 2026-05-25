# Bodega Market: CBOR Comparison Report

Generated 2026-04-14. Compares transactions built by `trix invoke --profile mainnet --skip-submit` against real on-chain transactions. Updated to use `datum_is` reference datum access (tx3c v0.17.0) and market CC01_ADA_REACHES_060_.

## Key Changes from Previous Report (2026-03-31)

1. **`project_outref_tx` + `project_outref_idx` replaced by `project_info_ref`** — The ProjectInfoDatum UTxO is now added as a typed reference input. `outref_id` is read from the datum via `datum_is` instead of being passed as two separate params.
2. **Reference input in CBOR field 18** — All 6 user txs now include the ProjectInfoDatum UTxO as a reference input (`0f01` / field 18 in the CBOR body).
3. **Test market changed** — Using CC01_ADA_REACHES_060_ (deadline 2026-12-30), an active market with long expiry.

## Test Market: CC01_ADA_REACHES_060_

| Config | Value |
|--------|-------|
| ProjectInfo UTxO | `fc914f41696c345b1a782e53ef6117c90aee1d7561d4442574a1380d40df71c3#0` |
| PositionScript | `addr1w9jw5wpd06f5v53sltrvxpkymraugehamf86r5z3vyl9jygxlhyt4` |
| Outref (from datum) | `12d6d37a4a3b53cf3bbbf0989133e0022eee3127d797f8610615fc6935bd6bbc#3` |
| share_policy_id | `6e8181d047370418d7ef48f013ffa1bd986388e84cd9c6eec676d98e` |
| admin_fee_percent | 200 |
| envelope_amount | 2,000,000 |
| Candidate YES | `B_CC01_YES` (`425f434330315f594553`) |
| Candidate NO | `B_CC01_NO` (`425f434330315f4e4f`) |
| Current yes_price | 536,556 |
| Current no_price | 463,443 |
| Deadline | 1798610580000 (2026-12-30) |

## Generated Transactions

| Operation | Tx Hash (unsigned) | Args File | Status |
|-----------|--------------------|-----------|--------|
| buy_position_yes | `7e1ed097...` | `buy_position_yes.json` | CBOR OK |
| buy_position_no | — | `buy_position_no.json` | CBOR OK |
| submit_reward_yes | `37bff8d2...` | `submit_reward_yes.json` | CBOR OK |
| submit_reward_no | `35d2b991...` | `submit_reward_no.json` | CBOR OK |
| sell_position_yes | `88a737cc...` | `sell_position_yes.json` | CBOR OK |
| sell_position_no | `ce90f82d...` | `sell_position_no.json` | CBOR OK |

All 6 user-facing transactions generate valid CBOR with the CC01 market.

## Test Wallets

| Role | Address | Used in |
|------|---------|---------|
| User (buy/sell) | `addr1qxedpq6fw09r7f4wu8cp9sd57q7vphxedf0zn8rzh6uxydluy3ncf0haddh9k75y4s07kz4f7t2c0jr8fg8new2s7p3ssvju2w` | buy_position_yes/no, sell_position_yes/no, submit_reward_no |
| User (reward YES) | `addr1q9nsd0stv3llw42nrgz9qcm8er4nmwugdtd4rre8pdlekk8pclsjx2fsxl5as44xdxc5m86zj7p0fw2k950xxdz68n9qflm5ny` | submit_reward_yes (holds 358 B_CC01_YES) |

Note: submit_reward_yes uses a different wallet because the primary test wallet does not hold B_CC01_YES tokens. The resolver requires the user's wallet to contain the share tokens being submitted.

## Structural Analysis

### Reference Input (new in this version)

All 6 txs include the ProjectInfoDatum UTxO as a reference input in CBOR field 18:

```
0f0112d9010281825820fc914f41696c345b1a782e53ef6117c90aee1d7561d4442574a1380d40df71c300
```

This is `fc914f41...#0` — the CC01 ProjectInfoDatum. The resolver fetches this UTxO, decodes the datum, and extracts `outref_id` (field 0) for use in the output PositionDatum.

### buy_position_yes — Datum verification

Decoded from CBOR output `7e1ed097...`:

| Datum Field | Value | Source |
|-------------|-------|--------|
| outref_id | `12d6d37a...#3` | **Read from ProjectInfoDatum reference** |
| pos_user_pkh | `b2d08349...` | Param |
| pos_user_stake_key | SomePkh(`fc2467...`) | Param |
| pos_type | Constr(0) = BuyPos | Hardcoded |
| pos_amount | 10 | Param |
| pos_batcher_fee | 700,000 | Param |
| pos_admin_fee_percent | 200 | Param |
| pos_unit_price | 536,556 | Param |
| pos_candidate | Constr(0) = Candidate0 (YES) | Hardcoded (_yes variant) |

The `outref_id` is correctly resolved from the ProjectInfoDatum reference input, not from caller params.

### submit_reward_yes — Datum verification

Decoded from CBOR output `37bff8d2...`:

| Datum Field | Value | Source |
|-------------|-------|--------|
| outref_id | `12d6d37a...#3` | **Read from ProjectInfoDatum reference** |
| pos_user_pkh | `6706be0b...` | Param (different wallet) |
| pos_user_stake_key | SomePkh(`e1c7e1...`) | Param |
| pos_type | Constr(2) = RewardPos | Hardcoded |
| pos_amount | 10 | Param |
| pos_batcher_fee | 700,000 | Param |
| pos_admin_fee_percent | 0 | Hardcoded |
| pos_unit_price | 0 | Hardcoded |
| pos_candidate | Constr(0) = Candidate0 (YES) | Hardcoded (_yes variant) |

Output includes 10 B_CC01_YES share tokens sent to the PositionScript alongside 2,700,000 lovelace (envelope + batcher_fee).

### sell_position_yes — Datum verification

Decoded from CBOR output `88a737cc...`:

| Datum Field | Value | Source |
|-------------|-------|--------|
| outref_id | `12d6d37a...#3` | **Read from ProjectInfoDatum reference** |
| pos_user_pkh | `b2d08349...` | Param |
| pos_user_stake_key | SomePkh(`fc2467...`) | Param |
| pos_type | Constr(1) = RefundPos | Hardcoded |
| pos_amount | 10 | Param |
| pos_batcher_fee | 700,000 | Param |
| pos_admin_fee_percent | 200 | Param |
| pos_unit_price | 536,556 | Param |
| pos_candidate | Constr(0) = Candidate0 (YES) | Hardcoded (_yes variant) |

ADA-only output (no share tokens) — 2,700,000 lovelace (envelope + batcher_fee).

## Parameter Comparison: Before vs After

### buy_position_yes/no

| Before (9 params) | After (8 params) | Change |
|--------------------|-------------------|--------|
| user_pkh | user_pkh | — |
| user_stake_key | user_stake_key | — |
| project_outref_tx | — | Removed (read from ref datum) |
| project_outref_idx | — | Removed (read from ref datum) |
| — | project_info_ref | New (UtxoRef to ProjectInfoDatum) |
| buy_amount | buy_amount | — |
| batcher_fee_amount | batcher_fee_amount | — |
| admin_fee_percent | admin_fee_percent | — |
| unit_price | unit_price | — |
| total_lovelace | total_lovelace | — |

### submit_reward_yes/no

| Before (9 params) | After (8 params) | Change |
|--------------------|-------------------|--------|
| user_pkh | user_pkh | — |
| user_stake_key | user_stake_key | — |
| share_policy_id | share_policy_id | — |
| project_outref_tx | — | Removed (read from ref datum) |
| project_outref_idx | — | Removed (read from ref datum) |
| — | project_info_ref | New (UtxoRef to ProjectInfoDatum) |
| envelope_amount | envelope_amount | — |
| candidate_name | candidate_name | — |
| share_amount | share_amount | — |
| batcher_fee_amount | batcher_fee_amount | — |

### sell_position_yes/no

| Before (9 params) | After (8 params) | Change |
|--------------------|-------------------|--------|
| user_pkh | user_pkh | — |
| user_stake_key | user_stake_key | — |
| project_outref_tx | — | Removed (read from ref datum) |
| project_outref_idx | — | Removed (read from ref datum) |
| — | project_info_ref | New (UtxoRef to ProjectInfoDatum) |
| envelope_amount | envelope_amount | — |
| share_amount | share_amount | — |
| batcher_fee_amount | batcher_fee_amount | — |
| admin_fee_percent | admin_fee_percent | — |
| unit_price | unit_price | — |

## On-Chain Findings (1B60_CRUDE_OIL_CLOSES analysis)

During this update, 7 real user buy_position txs from market 1B60 were decoded and analyzed:

1. **`admin_fee_percent` varies per position** — Values of 200 (2%) and 10 (0.1%) observed in the same market. ProjectInfoDatum says 200. Likely a BODEGA holder discount. Must remain a caller param.

2. **`unit_price` is LMSR-computed, not spot price** — Differs from PredictionDatum `yes_price`/`no_price` by up to 56,564 lovelace. Depends on trade size. Must remain a caller param.

3. **`batcher_fee_amount` is constant (700,000)** — Consistent across all 7 txs analyzed. Not stored in any on-chain datum. Must remain a caller param.

## Architecture

```
Param source          | Values
----------------------|---------------------------------------------------
Env (instance-level)  | NFT token names, BODEGA details, script refs,
                      | batcher_policy_id, psettings NFT
JSON args (per-market)| PositionScript address, project_info_ref,
                      | share_policy_id, envelope_amount, candidate_name
JSON args (per-call)  | User address/pkh/stake_key, amounts, prices,
                      | admin_fee_percent, batcher_fee_amount
Reference datum       | outref_id (read from ProjectInfoDatum at resolve)
```

## How to Reproduce

```bash
cd protocols_tx3/bodega-market

# buy_position_yes
trix invoke --profile mainnet --skip-submit --args-json-path invoke-args/buy_position_yes.json
# select: buy_position_yes

# buy_position_no
trix invoke --profile mainnet --skip-submit --args-json-path invoke-args/buy_position_no.json
# select: buy_position_no

# submit_reward_yes
trix invoke --profile mainnet --skip-submit --args-json-path invoke-args/submit_reward_yes.json
# select: submit_reward_yes

# submit_reward_no
trix invoke --profile mainnet --skip-submit --args-json-path invoke-args/submit_reward_no.json
# select: submit_reward_no

# sell_position_yes
trix invoke --profile mainnet --skip-submit --args-json-path invoke-args/sell_position_yes.json
# select: sell_position_yes

# sell_position_no
trix invoke --profile mainnet --skip-submit --args-json-path invoke-args/sell_position_no.json
# select: sell_position_no

# create_market (unchanged from previous report)
trix invoke --profile mainnet --skip-submit --args-json-path invoke-args/create_market.json
# select: create_market
```
