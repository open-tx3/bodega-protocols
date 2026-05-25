# TX3 Limitations Found During Bodega Market Implementation

Discovered with trix 0.21.1 (2026-03-13). Updated 2026-04-14 with tx3c v0.17.0 / trix 0.22.0 findings.

---

## Active Limitations (impact on production use)

### 1. No Multiplication or Division Operators

The grammar only supports `+` and `-` for data expressions (`data_infix = _{ data_add | data_sub }`). Bodega Market requires arithmetic for fee and payment calculations:

```
payment = buy_amount * unit_price
admin_fee = buy_amount * admin_fee_percent * 1_000_000 / 10_000
total = payment + admin_fee + batcher_fee + envelope_amount
```

**Impact:** Forces `total_lovelace` to be pre-computed by the API caller. This leaks protocol internals (fee formula, AMM pricing) to the caller layer. Same applies to `unit_price` (LMSR price per share).

**Workaround applied:** Added `total_lovelace` and `unit_price` as tx parameters. The caller computes them off-chain.

**Affected txs:** `buy_position_yes`, `buy_position_no`, `sell_position_yes`, `sell_position_no`

### 2. No Dynamic-Length Input/Output Lists (Batch Patterns)

tx3 requires each input/output to be declared statically. There's no way to express "for each position in the batch, include an input and output".

**Impact: CRITICAL** — Makes all batcher/admin operations impossible to model. The Aiken redeemer carries `pos_indices: List<(Int, Int)>` — a dynamic list of (input_idx, output_idx) pairs. Each pair maps a consumed position UTxO to a user payout output.

| Tx not implemented | Aiken Redeemer | Why |
|----|---------------|-----|
| Process Buy (batch) | `PredictionRedeemer::Apply` | Dynamic list of position inputs/outputs |
| Process Reward (batch) | `PredictionRedeemer::Reward` | Dynamic list + burn + oracle reference |
| Process Refund (batch) | `PredictionRedeemer::Refund` | Dynamic list + burn |
| Withdraw Admin Fees | `PredictionRedeemer::WithdrawFee` | Multi-validator tx |
| Close Market | `PredictionRedeemer::Close` | Multi-validator consume + NFT transfer |

**Workaround:** None — these 5 transactions cannot be implemented in tx3.

### 3. No Tuple Types — `List<(ByteArray, Int)>` Can't Be Represented

tx3 has `List<T>` and `Map<K,V>` but no tuple type `(A, B)`. The on-chain `PredictionDatum` in the GitHub V2 source uses `predictions: List<(ByteArray, Int)>` for candidate/amount pairs.

**Workaround applied:** The on-chain V2 repo type was not used directly. The actual deployed contract uses individual fields (`yes_shares`, `no_shares`, `yes_price`, `no_price`) instead of a list of tuples, so this was modeled as a flat record `PredictionDatum` with 7 fields. No data loss.

**Impact:** Low for this protocol (deployed contract structure avoids the issue). Would block modeling the V2 GitHub source faithfully.

### 4. Custom Types (Enums) Cannot Be Passed as Parameters

`from_json()` in `tx3-resolver/src/interop.rs` only supports 5 types: Int, Bool, Bytes, Address, UtxoRef. The `CandidateIdx` enum (`Candidate0` / `Candidate1`) cannot be passed dynamically.

**Impact:** Every tx that takes a candidate index must be duplicated into `_yes` / `_no` variants with the enum hardcoded in each.

**Workaround applied:** Split 3 user-facing txs into 6 variants:
- `buy_position_yes` / `buy_position_no`
- `submit_reward_yes` / `submit_reward_no`
- `sell_position_yes` / `sell_position_no`

The only difference between variants is `CandidateIdx::Candidate0 {}` vs `CandidateIdx::Candidate1 {}`. If enums were passable, all 6 would collapse into 3.

### 5. No Conditional Logic / Branching

Bodega supports markets where the payment token is either ADA or a custom token (`ProjectInfoDatum.payment_policy_id`). The position UTxO value structure differs:
- **ADA market:** lovelace includes payment + fee + envelope + batcher_fee
- **Token market:** lovelace = envelope + batcher_fee, token amount = payment + fee

tx3 has no `if/else` or `when` to branch on `payment_policy_id`.

**Workaround:** Duplicate txs into `_ada` / `_token` variants (same approach as the `_yes`/`_no` split for CandidateIdx). Would affect `buy_position` and `sell_position`.

**Status:** Not implemented — all active mainnet markets use ADA as payment token. If a token-payment market appears, the `_token` variants need to be added.

### 6. Reference Datum Fields Only Usable in Datum Construction

Discovered with tx3c v0.17.0 / trix 0.22.0 (2026-04-14).

tx3c v0.17.0 (#318) added `datum_is` on reference blocks, allowing typed field access on reference input datums. However, the **resolver** only supports these fields inside **output datum construction** — not in **amount expressions** (`Ada(...)`, `AnyAsset(...)`, `min_amount`, change calculations).

```tx3
reference project_info {
    ref: project_info_ref,
    datum_is: ProjectInfoDatum,
}

// WORKS — datum field in output datum
output {
    datum: PositionDatum {
        outref_id: project_info.outref_id,  // OK
    },
}

// FAILS — datum field in amount expression
output {
    amount: Ada(project_info.pi_envelope_amount + batcher_fee_amount),  // resolver error
}

// FAILS — datum field in AnyAsset
locals {
    shares: AnyAsset(project_info.pi_share_policy_id, project_info.candidate_yes_name, amount),  // resolver error
}
```

**Error:** `expected assets, got EvalBuiltIn(Add(Assets([...]), EvalBuiltIn(Property(EvalCoerce(IntoAssets(EvalParam(ExpectInput(...))))), Number(13)))))`

**Impact:** Fields that would eliminate caller params (`pi_envelope_amount`, `pi_share_policy_id`, `candidate_yes/no_name`) cannot be read from reference datums because they're used in amount calculations. Only `outref_id` (used exclusively in datum construction) can be read from the reference.

**Workaround applied:** Keep `envelope_amount`, `share_policy_id`, and `candidate_name` as caller-provided tx params. Use `datum_is` only for `outref_id` in the output datum.

**Affected txs:** `submit_reward_yes/no` (would eliminate 3 params each), `sell_position_yes/no` (would eliminate 1 param each)

**Potential fix:** The resolver needs to evaluate reference datum field access at resolve time (fetching the UTxO, decoding the datum, extracting the field value) before building amount expressions. Currently it defers evaluation and the amount builder doesn't know how to handle the unevaluated expression.

---

## ~~Solved~~ Limitations (fixed in recent tx3c releases)

### ~~Reference Inputs Cannot Read Datum Values~~ — SOLVED in tx3c v0.17.0

**Previously:** tx3 could add reference inputs to a transaction but datum contents were opaque — no field access on reference blocks.

**Fixed:** tx3c v0.17.0 (#318) added `datum_is` on reference blocks. Field access works for **datum construction** in outputs. Applied to all 6 user-facing txs to read `outref_id` from the `ProjectInfoDatum` reference — eliminating `project_outref_tx` + `project_outref_idx` as separate params.

**Remaining limitation:** Field access in amount expressions is still not supported (see active limitation #6 above).

### ~~Record Field Name Shadowing~~ — SOLVED in tx3c v0.17.0

**Previously:** tx param or env var names that collided with type field names caused a lowering panic (`not yet implemented` on `RecordField` symbol).

**Fixed:** tx3c v0.17.0 (#316) added support for shadowing of record field names.

---

## Non-Issues / Clarifications

### `admin_fee_percent` Must Remain a Caller Parameter

On-chain analysis (2026-04-14, market 1B60_CRUDE_OIL_CLOSES) shows `pos_admin_fee_percent` in PositionDatum can differ from `ProjectInfoDatum.admin_fee_percent` within the same market (values 200 and 10 observed). The deployed contract (9-field PositionDatum) uses per-position fee values — likely a BODEGA holder discount mechanism. Reading from the ProjectInfoDatum reference would produce incorrect datums for discounted users.

### `collateral_return` / `total_collateral` Not Generated

Real on-chain txs include explicit `collateral_return` (field 16) and `total_collateral` (field 17). The TRP does not generate these. This is cosmetic — transactions still validate.

---

## Summary

| # | Status | Description | Workaround | Impact |
|---|--------|-------------|------------|--------|
| 1 | Active | No `*` / `/` operators | Caller pre-computes `total_lovelace` | `total_lovelace` + `unit_price` as params per buy/sell tx |
| 2 | Active | No dynamic input/output lists | None — batcher txs not implementable | 5 txs blocked (all batcher/admin) |
| 3 | Active | No tuple types | N/A (deployed contract avoids tuples) | Low |
| 4 | Active | Enums not passable as params | Duplicate txs into `_yes`/`_no` variants | 3 txs -> 6 variants |
| 5 | Active | No conditional logic | Not implemented (no token markets active) | Would need `_ada`/`_token` variants for buy/sell |
| 6 | Active | Ref datum fields only in datum construction | Keep params for amount-used fields | 4 extra params across submit_reward + sell_position |
| - | ~~Solved~~ | ~~Reference inputs can't read datums~~ | Fixed in tx3c v0.17.0 (#318) | `outref_id` now read from reference |
| - | ~~Solved~~ | ~~Record field name shadowing panic~~ | Fixed in tx3c v0.17.0 (#316) | No longer need to rename params |

**Transactions blocked by tx3 limitations:** 5 out of 11 total (all batcher/admin operations).
**Transactions requiring workaround duplication:** 3 -> 6 (enum variant split).
**Pending if token-payment markets appear:** `buy_position` and `sell_position` would need `_ada`/`_token` variants.
