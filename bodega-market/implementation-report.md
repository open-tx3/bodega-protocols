# Bodega Market — tx3 Implementation Report

## Protocol Overview

Bodega Market is a prediction market on Cardano (Aiken/PlutusV3). It uses a **batcher pattern**: users submit position UTxOs to a script address, and a licensed batcher processes them in batches against the market state.

- **V2 contracts repo:** `bodega-market/bodega-market-smart-contracts-v2`
- **11 validators, 18 endpoints** in V2
- **Mainnet active:** ~40k txs, ~60k ADA locked across two instances

---

## What Was Implemented

### 3 user-facing transactions

| Tx | Description | Validators Used |
|----|-------------|-----------------|
| `buy_position` | Mint share tokens + create BuyPos position UTxO | `project_shares::Buy` (mint) |
| `submit_reward` | Send shares to position script as RewardPos | None (send-to-script) |
| `submit_refund` | Send shares to position script as RefundPos | None (send-to-script) |

### Types modeled from Aiken V2

| tx3 Type | Aiken Equivalent | Notes |
|----------|-----------------|-------|
| `OutputRef` | `OutputReference` | Nested record type (tx_hash + output_index) |
| `OptionPkh` | `Option<PubKeyHash>` | Variant: SomePkh / NonePkh |
| `PositionType` | `PositionType` | Enum: BuyPos / RewardPos / RefundPos |
| `PositionDatum` | `PositionDatum` | Full datum with nested OutputRef + OptionPkh + PositionType |
| `ShareRedeemer` | `ShareRedeemer` | Variant: Buy / Reward / Refund |
| `PositionRedeemer` | `PositionRedeemer` | Record with pred_in_idx |
| `PredictionRedeemer` | `ProjectPredictionRedeemer` | Variant: Apply / Reward / Refund / WithdrawFee / Close |
| `PredictionDatum` | `ProjectPredictionDatum` | Simplified (predictions as `List<List<Int>>` instead of `List<(ByteArray, Int)>`) |

### What worked well

- **Nested types in datums:** tx3 supports nested record construction (`OutputRef { ... }` inside `PositionDatum { ... }`), which maps cleanly to Aiken's nested types.
- **Variant constructors in datums:** `PositionType::BuyPos {}`, `OptionPkh::SomePkh { value: ... }` work for constructing variant fields inside datum constructors.
- **Variant redeemers:** `ShareRedeemer::Buy { project_info_ref_idx: 0, }` works for minting policy redeemers.
- **Reference scripts:** `reference share_script { ref: share_script_ref, }` cleanly declares on-chain reference scripts.
- **AnyAsset in locals:** `AnyAsset(share_policy_id, candidate_name, buy_amount)` allows dynamic asset construction from env vars and params.
- **concat() for Bytes:** `concat(bytes_a, bytes_b)` works for constructing dynamic asset names from components (e.g., oracle token names like `concat(market_name, 0x5f4f5241434c45)`).

---

## What Was NOT Implemented

### 1. Batcher Operations (5 transactions)

The core batch processing transactions could not be implemented:

| Tx | Aiken Redeemer | Why Not |
|----|---------------|---------|
| Process Buy (batch) | `PredictionRedeemer::Apply` | Dynamic list of position inputs/outputs |
| Process Reward (batch) | `PredictionRedeemer::Reward` | Dynamic list + burn + oracle reference |
| Process Refund (batch) | `PredictionRedeemer::Refund` | Dynamic list + burn |
| Withdraw Admin Fees | `PredictionRedeemer::WithdrawFee` | Multi-validator tx (needs reference inputs L4) |
| Close Market | `PredictionRedeemer::Close` | Multi-validator consume + NFT transfer (needs reference inputs L4) |

**Root cause:** These operations process **N position UTxOs** in a single transaction. The Aiken redeemer includes `pos_indices: List<(Int, Int)>` — a dynamic list of (input_idx, output_idx) pairs. Each pair maps to a position UTxO that gets consumed and a corresponding user payout output. tx3 has no way to express "for each item in a list, create an input and an output".

### 2. Create Project

| Tx | Aiken Validator | Why Not |
|----|----------------|---------|
| Create Market | `project_authtoken_mp` | Requires seed UTxO consumption + outputs to 3 different scripts + complex datum construction with List of candidates |

**Root cause:** The `PATMintRedeemer` includes a `seed: OutputReference` that must match a consumed input (one-shot minting pattern). The datum includes `candidates: List<ByteArray>` and `predictions: List<(ByteArray, Int)>` which require tuple-in-list construction. Additionally, the tx must output to project_info, project_prediction, AND protocol_treasury scripts simultaneously, each with specific datum/value requirements.

### 3. Update Oracle Info

| Tx | Aiken Validator | Why Not |
|----|----------------|---------|
| Set Oracle Result | `project_info::CreatorUpdate` + oracle mint | Multi-validator spending + reference inputs (L4) |

**Root cause:** Requires spending project_info + protocol_manager UTxOs simultaneously and reading protocol_settings as a reference input (L4). The oracle token's dynamic asset name can be constructed with `concat()` (e.g., `concat(market_name, 0x5f4f5241434c45)` for `{name}_ORACLE`), so naming is not a blocker.

---

## tx3 Language Limitations Encountered

### L1: No multiplication or division operators

**Impact: HIGH** — Forces `total_lovelace` to be pre-computed by the caller.

The grammar only supports `+` and `-` for data expressions (`data_infix = _{ data_add | data_sub }`). Bodega Market requires:

```
payment = buy_amount * 1_000_000  (shares to lovelace)
admin_fee = buy_amount * admin_fee_percent * 1_000_000 / 10_000
```

**Workaround used:** Added `total_lovelace` as a tx parameter. The API caller must pre-compute the payment + fee amount. This leaks protocol internals (fee calculation) to the caller.

**Suggestion:** Add `*` and `/` (integer division) to `data_infix`. Even just `*` would cover most DeFi use cases.

### L2: No dynamic-length input/output lists (batch patterns)

**Impact: CRITICAL** — Makes batcher/batch operations impossible to model.

Many Cardano protocols use a "batcher" pattern where a single transaction processes N user positions. The Aiken redeemer carries `pos_indices: List<(Int, Int)>` and the transaction has N script inputs + N user outputs.

tx3 requires each input/output to be declared statically. There's no way to say "for each position in the batch, include an input and output".

**Suggestion:** A `foreach` or `batch` construct:
```tx3
// Hypothetical syntax
foreach position in positions {
    input {
        ref: position.utxo_ref,
        datum_is: PositionDatum,
        redeemer: PositionRedeemer { pred_in_idx: prediction_input_idx, },
    }
    output {
        to: position.user_address,
        amount: Ada(position.payout),
    }
}
```

This is the single biggest gap for DeFi protocol coverage. Most DEXes (Minswap, SundaeSwap), lending protocols, and prediction markets use batch patterns.

### L3: No tuple types — `List<(ByteArray, Int)>` can't be represented

**Impact: MEDIUM** — Forced to simplify `PredictionDatum.predictions` type.

Aiken uses `List<(ByteArray, Int)>` for the predictions field (list of candidate/amount pairs). tx3 has `List<T>` and `Map<K,V>` but no tuple type `(A, B)`.

**Workaround used:** Modeled as `List<List<Int>>` which is semantically incorrect but compiles. In practice, the prediction datum is only READ by batcher operations (which we can't implement anyway).

**Suggestion:** Add tuple syntax: `Tuple<Bytes, Int>` or `(Bytes, Int)`. Alternatively, allow `List<MyRecordType>` where the record has named fields.

### L4: No reference inputs (read-only UTxO references)

**Impact: MEDIUM** — Can't explicitly declare read-only reference inputs.

Bodega's `buy_position` requires the project_info UTxO as a **reference input** (CIP-31) — it's read but not consumed. tx3's `reference` block only supports script references (`reference script { ref: ... }`), not arbitrary reference inputs for datum reading.

The `ShareRedeemer::Buy { project_info_ref_idx: 0 }` hardcodes index 0, assuming the project_info UTxO will be the first reference input. But tx3 has no way to declare "include this UTxO as a reference input at index 0".

**Workaround used:** Hardcoded `project_info_ref_idx: 0`. Since Cardano sorts transaction inputs deterministically by `TxOutRef` and tx3 controls the transaction construction, the index is predictable at build time — but tx3 lacks a way to explicitly declare and reference data-carrying UTxOs as reference inputs.

**Suggestion:** Allow `reference` blocks for data-carrying UTxOs, not just scripts:
```tx3
reference project_info {
    ref: project_info_utxo_ref,
    datum_is: ProjectInfoDatum,
}
// Then reference: project_info.deadline, project_info.candidates, etc.
```

### L5: No conditional logic / branching

**Impact: MEDIUM** — Can't handle ADA vs custom token payment paths.

Bodega supports markets where the payment token is either ADA or a custom token. The position UTxO value structure differs:
- ADA market: lovelace includes payment + fee + envelope + batcher_fee
- Token market: lovelace = envelope + batcher_fee, token = payment + fee

tx3 has no `if/else` or `when` construct to branch on `payment_policy_id`.

**Workaround used:** Implemented only the ADA payment path. A separate tx definition would be needed for token-payment markets.

**Suggestion:** Conditional blocks:
```tx3
if payment_policy_id == 0x {
    // ADA path
} else {
    // Token path
}
```


---

## Coverage Summary

| Category | Total Ops | Implemented | % |
|----------|-----------|-------------|---|
| User-facing txs | 3 | 3 | 100% |
| Batcher txs | 3 | 0 | 0% |
| Admin txs | 3 | 0 | 0% |
| Protocol setup | 2 | 0 | 0% |
| **Total** | **11** | **3** | **27%** |

The 3 implemented transactions cover the **user-facing surface** of the protocol — the operations an end-user would perform to participate in a prediction market. The remaining 8 operations are batcher/admin/setup operations that require features tx3 doesn't currently support.

## Mainnet Deployment — V3 (most active)

V3 has ~22k txs and ~60k ADA locked. Stake key: `stake17x50lxk2yne6cuzywprx9rwgkfsxjfkf2fnw9zv3u42t2mqcwnc7y`.

### Parties (fixed per instance)

| Party | Address | Role |
|-------|---------|------|
| PositionScript | `addr1xx25vyyteavkeddsueufzr4ahgsa987fafvhv032tnmvg0dgl7dv5f8n43cyguzxv2xu3vnqdynvj5nxu2yere25k4kqf0df9x` | Position + prediction UTxOs |
| PredictionScript | *(same as PositionScript)* | Same address in V3 |
| ProjectInfoScript | `addr1x8x7nn5lch2uawxct2hjr06kgsplxu9rpm8gg9tyffv4u8agl7dv5f8n43cyguzxv2xu3vnqdynvj5nxu2yere25k4kqt4qep6` | Market config + BODEGA pledge |
| ProtocolTreasury | `addr1x8ru0tc0tsy23wfe34u02zazflmkat2ar8rzk8qf93f5r94gl7dv5f8n43cyguzxv2xu3vnqdynvj5nxu2yere25k4kqpuwsp0` | Open fees + admin fee withdrawals |
| Batcher | `addr1w9puyxdnzn3upaj73mk8lq0cq85jwrlu0kt62dw8dhka4lgjvdkjs` | Licensed batch processor |
| ProtocolSettings | `addr1w8ru0tc0tsy23wfe34u02zazflmkat2ar8rzk8qf93f5r9szeleuc` | PSettingDatum (read-only) |
| ProtocolManager | `addr1wx50lxk2yne6cuzywprx9rwgkfsxjfkf2fnw9zv3u42t2mqyyn7zk` | PROTOCOL_MANAGER_NFT |
| RefScriptLock | `addr1w8uxazmlktyy4lauq9nnlmgg08zz9mf4xvyf8sg8wg7pp7cyf6ysp` | Reference scripts storage |

**Finding:** PositionScript and PredictionScript are the **same address**. Both position UTxOs (user orders) and prediction UTxOs (PROJECT_PREDICTION_NFT + market state) live at `addr1xx25vyy...`. This is because the parameterized validators compile to the same script hash in V3.

### Environment — Instance Level (shared across all markets)

| Variable | Value | Source |
|----------|-------|--------|
| `project_authtoken_policy_id` | `08a8c0fbe85823132cb14a3767d2e114c8c85f58153b072f8c9e3633` | All PROJECT_INFO_NFTs at project_info address carry this policy |
| `psettings_nft_policy_id` | `d2e8f0bab4bc0e62ebe9abf79b7a13280229ad60ca8e329892cb71a7` | Token at protocol_settings UTxO |
| `psettings_nft_tn` | `50524f544f434f4c5f53455454494e47535f4e4654` | "PROTOCOL_SETTINGS_NFT" in hex |
| `batcher_policy_id` | `c7c7af0f5c08a8b9398d78f50ba24ff76ead5d19c62b1c092c534196` | PSettingDatum field[3] |
| `prediction_script_ref` | `73cc84de3e1b056abb3e004bdc25dd2ee49f4435ba4319f1faefe2ced843e894#2` | REF_P_PREDICTION at ref_script_lock |
| `position_script_ref` | `73cc84de3e1b056abb3e004bdc25dd2ee49f4435ba4319f1faefe2ced843e894#2` | Same ref (same validator) |
| `protocol_settings_utxo` | `73cc84de3e1b056abb3e004bdc25dd2ee49f4435ba4319f1faefe2ced843e894#0` | PROTOCOL_SETTINGS_NFT UTxO |

### Environment — Per-Market (varies for each prediction market)

| Variable | Description | How to obtain |
|----------|-------------|---------------|
| `share_policy_id` | Share token policy (unique per market) | From minting tx of that market's share tokens |
| `share_script_ref` | Reference script for share policy | May not exist — per-market policies may be inline |
| `payment_policy_id` | Payment asset policy (`""` = ADA) | From ProjectInfoDatum |
| `payment_token_name` | Payment asset name (`""` = ADA) | From ProjectInfoDatum |
| `project_outref_tx` | OutputReference tx_hash linking positions to market | From ProjectInfoDatum.outref_id |
| `project_outref_idx` | OutputReference output_index | From ProjectInfoDatum.outref_id |
| `envelope_amount` | Min ADA per position UTxO (typically 2 ADA) | From ProjectInfoDatum.envelope_amount |
| `oracle_policy_id` | Oracle token policy | From ProjectInfoDatum.oracle_policy_id |
| `oracle_token_name` | Oracle token name | From ProjectInfoDatum.oracle_token_name |

### PSettingDatum (decoded from protocol_settings UTxO)

Source: `73cc84de3e1b056abb3e004bdc25dd2ee49f4435ba4319f1faefe2ced843e894#0`

| Field | Name | Value | Notes |
|-------|------|-------|-------|
| 0 | pledge | 50,000,000,000 | 50k BODEGA (6 decimals) |
| 1 | pledge_policy_id | `5deab590a137066fef0e56f06ef1b830f21bc5d544661ba570bdd2ae` | BODEGA token policy |
| 2 | pledge_token_name | `424f44454741` | "BODEGA" |
| 3 | protocol_treasury_script_hash | `c7c7af0f5c08a8b9398d78f50ba24ff76ead5d19c62b1c092c534196` | |
| 4 | share_ratio | 5,000 | 50% (base 10,000) |
| 5 | open_fee | 2,000,000 | 2 ADA |
| 6 | open_fee_policy_id | `""` | ADA |
| 7 | open_fee_token_name | `""` | ADA |
| 8 | project_authtoken_policy_id | `08a8c0fbe85823132cb14a3767d2e114c8c85f58153b072f8c9e3633` | |
| 9 | project_info_token_name | `50524f4a4543545f494e464f5f4e4654` | "PROJECT_INFO_NFT" |
| 10 | project_prediction_token_name | `50524f4a4543545f50524544494354494f4e5f4e4654` | "PROJECT_PREDICTION_NFT" |
| 11 | protocol_stake_key_hash | `a8ff9aca24f3ac70447046628dc8b2606926c95266e28991e554b56c` | |

### Token Name Hex Reference

| Name | Hex |
|------|-----|
| PROTOCOL_SETTINGS_NFT | `50524f544f434f4c5f53455454494e47535f4e4654` |
| PROTOCOL_MANAGER_NFT | `50524f544f434f4c5f4d414e414745525f4e4654` |
| PROJECT_INFO_NFT | `50524f4a4543545f494e464f5f4e4654` |
| PROJECT_PREDICTION_NFT | `50524f4a4543545f50524544494354494f4e5f4e4654` |
| BODEGA | `424f44454741` |

---

## On-chain Datum Analysis (2026-03-13)

### Main finding

The `PositionDatum` deployed on mainnet has **9 fields**, not the **7** defined by the public repo `bodega-market-smart-contracts-v2`. This was verified across multiple txs and instances.

### Comparison: GitHub V2 vs Actual On-chain

**GitHub V2 `PositionDatum` (7 fields):**
```
[0] outref_id: OutputReference
[1] pos_user_pkh: PubKeyHash
[2] pos_user_stake_key: Option<PubKeyHash>
[3] pos_type: PositionType (BuyPos=Constr0, RewardPos=Constr1, RefundPos=Constr2)
[4] pos_amount: Int
[5] pos_batcher_fee: Int
[6] pos_candidate: ByteArray
```

**Actual on-chain (9 fields):**
```
[0] outref_id: OutputReference              ✓ matches
[1] pos_user_pkh: PubKeyHash               ✓ matches
[2] pos_user_stake_key: Option<PubKeyHash>  ✓ matches
[3] pos_type: PositionType                  ✓ matches
[4] pos_amount: Int                         ✓ matches
[5] pos_batcher_fee: Int                    ✓ matches
[6] pos_admin_fee_percent: Int              ← NEW (not in GitHub)
[7] pos_extra: Int                          ← NEW (not in GitHub)
[8] pos_candidate: CandidateIdx (enum)      ← CHANGED from ByteArray to Constr(N,[])
```

### On-chain evidence

#### Tx `fc6556d9` — BuyPos (new instance, addr1w9t35fy...)
- **Datum hash:** `a0906c60f3b95a6858a582a6ca25d701fe08787c5ba3a6ea156df12a2575867a`
- **CBOR:** `d8799fd8799f5820dccb306a96261269ec2c8769d907d891095de9939197d8c39a1e0c1bf08b59e903ff581cb2d0834973ca3f26aee1f012c1b4f03cc0dcd96a5e299c62beb86237d8799f581cfc246784befd6b6e5b7a84ac1feb0aa9f2d587c8674a0f3cb950f063ffd8798019012f1a000aae6018c81a00076978d87980ff`

| Field | Value | Interpretation |
|-------|-------|----------------|
| [0] outref_id | `{dccb306a...#3}` | Project outref |
| [1] pos_user_pkh | `b2d0834973ca3f26...` | User payment key hash |
| [2] pos_user_stake_key | `Some(fc246784befd...)` | User stake key |
| [3] pos_type | `Constr(0,[])` | BuyPos |
| [4] pos_amount | `303` | Shares to buy |
| [5] pos_batcher_fee | `700000` | 0.7 ADA |
| [6] **pos_admin_fee_percent** | `200` | 2% base 10000 |
| [7] **pos_extra** | `485752` | Unknown purpose |
| [8] **pos_candidate** | `Constr(0,[])` | Candidate 0 (first candidate) |

#### Tx `c7f0692d` — BuyPos (same instance, same user)
- **Datum hash:** `a76402289c11d41ace56b23d61101b454f83003807c7e81c2be700afd486672c`

| Field | Value | Delta vs previous |
|-------|-------|-------------------|
| [4] pos_amount | `191` | Different (191 vs 303) |
| [5] pos_batcher_fee | `700000` | Same |
| [6] pos_admin_fee_percent | `200` | Same |
| [7] pos_extra | `507147` | Different (507147 vs 485752) |
| [8] pos_candidate | `Constr(0,[])` | Same |

#### V3 — RefundPos (datum from previous research)
- **CBOR:** `d8799fd8799f5820cb9485fd...d87b8018501a000aae600000d87980ff`

| Field | Value | Notes |
|-------|-------|-------|
| [3] pos_type | `Constr(2,[])` | RefundPos |
| [4] pos_amount | `80` | |
| [5] pos_batcher_fee | `700000` | |
| [6] pos_admin_fee_percent | `0` | **0 for refund** |
| [7] pos_extra | `0` | **0 for refund** |
| [8] pos_candidate | `Constr(0,[])` | |

### Interpretation of new fields

**Field [6] — `pos_admin_fee_percent`** (high confidence)
- Constant value `200` in all observed BuyPos → 2% base 10000
- Value `0` in RefundPos → no fee charged on refund
- Matches `admin_fee_percent` from `ProjectInfoDatum` (validated in V2 source)
- Hypothesis: the deployed contract includes the fee in the datum so the batcher can calculate the fee without reading the ProjectInfoDatum

**Field [7] — `pos_unit_price`** (high confidence, verified with 5 trades + 2 markets)
- Varies between BuyPos txs: 485752, 507147, 507451, 506235, 493631
- Is `0` in RefundPos and RewardPos
- **Confirmed: average LMSR price per share in lovelace, computed off-chain by the frontend**
- Formula: `pos_unit_price = LMSR_cost_total / pos_amount`
  where LMSR_cost_total = `b * ln(e^(q_new/b) + e^(q_other/b)) - b * ln(e^(q_old/b) + e^(q_other/b))`
  and `b` = liquidity parameter from market info datum field[14] (e.g.: 2885)
- On-chain verification (see `bodega-market-onchain-analysis.md` section 2 for full detail):
  - 5/5 trades from market 9EE9: `actual_LMSR_cost / pos_amount ≈ pos_unit_price` (±2 lovelace)
  - 3/3 trades from market 6672 (V3): same formula verified
- Total payment is: `LMSR_cost + admin_fee + batcher_fee(700,000) + min_utxo(2,000,000)`
  where `admin_fee = LMSR_cost * (admin_fee_percent + field[6]) / 10,000`
- See `bodega-market-costs.md` section "pos_unit_price" for full formula

**Field [8] — `pos_candidate`** (high confidence)
- Changed from `ByteArray` (candidate name) to enum `Constr(N,[])`
- `Constr(0,[])` = first candidate from `ProjectInfoDatum.candidates`
- `Constr(1,[])` = second candidate, etc.
- Optimization: avoids storing the full string in each position datum

### New instance discovered

Tx `fc6556d9` targets a previously unknown address:

| Data | Value |
|------|-------|
| Address | `addr1w9t35fy2qu9xpn3e2pc6w3zwsje795kaeu38eqw9xyltjsg5ldlnp` |
| Script hash | `571a248a070a60ce395071a7444e84b3e2d2ddcf227c81c5313eb941` |
| Type | Enterprise (no stake key) |
| Current balance | 0 ADA (UTxOs consumed by batcher) |
| Recent activity | Multiple txs from the same user, processed quickly |

This is a **third instance** (neither A nor B from previous research). The script hash does not match any known validator from the V2 repo, confirming that the deployed code differs from the published GitHub code.

### Market Create Tx — 9EE9 example

**Tx:** `8797e92d475ad66d208ebe8cb929c556fbd703509b5c86b1ea4ee8ce2b921893`

**Pattern:** No per-market minting — the NFTs (PROJECT_INFO_NFT + PROJECT_PREDICTION_NFT) are pre-minted in batch and then deployed by moving from wallet to script addresses. This differs from the V2 source which uses one-shot minting.

**Outputs:**
- `#0` → project_info script: 1 PROJECT_INFO_NFT + 50,000 BODEGA (pledge) + datum
- `#1` → prediction script: 1 PROJECT_PREDICTION_NFT + ~2,001 ADA + datum
- `#2` → treasury: 2 ADA (open_fee)
- `#3` → wallet (change)

### ProjectInfoDatum (on-chain: 17 fields, GitHub V2: 15)

Datum from market `9EE9_MENS_BIG_12_QF_I`:

| Field | Type | Value | V2 Name |
|-------|------|-------|---------|
| [0] | OutputRef | `{dccb306a...#3}` | outref_id ✓ |
| [1] | Bytes | `326faac2b159...` | owner_pkh ✓ |
| [2] | Option | `Some(400b2325...)` | owner_stake_key ✓ |
| [3] | Bytes | `"9EE9_MENS_BIG_12_QF_I"` | project_name ✓ |
| [4] | Int | `1773333000000` (2026-03-12T16:30Z) | deadline ✓ |
| [5] | Bytes | `""` (ADA) | payment_policy_id ✓ |
| [6] | Bytes | `""` (ADA) | payment_token_name ✓ |
| [7] | Bytes | `d978b820644a33...` | batcher_policy_id ✓ |
| [8] | Bytes | `571a248a070a60...` | position_script_hash ✓ |
| [9] | Bytes | `ea69dcc8b82182...` | share_policy_id ✓ |
| [10] | Bytes | `bcd499d9373834...` | oracle_policy_id ✓ |
| [11] | Bytes | `"9EE9_..._ORACLE"` | oracle_token_name ✓ |
| [12] | Int | `200` (2%) | admin_fee_percent ✓ |
| [13] | Int | `2000000` (2 ADA) | envelope_amount ✓ |
| **[14]** | **Int** | **`2885`** | **NEW — purpose TBD** |
| [15] | Bytes | `"B_9EE9_YES"` | candidates[0] (formerly: List) |
| [16] | Bytes | `"B_9EE9_NO"` | candidates[1] (formerly: List) |

**Differences vs V2 GitHub:**
1. Field [14] is new (int: 2885, unknown purpose)
2. `candidates` changed from `List<ByteArray>` to individual fields [15] and [16]
3. `position_script_hash` [8] confirms each market has its own script address

### PredictionDatum (on-chain: 7 fields, GitHub V2: 3)

| Field | Type | Value | Interpretation |
|-------|------|-------|----------------|
| [0] | OutputRef | `{dccb306a...#3}` | outref_id ✓ |
| [1] | Int | `38,598,977` | total_fee (38.6 ADA accumulated) |
| [2] | Int | `2,675,245,167` | **total_pool** (2,675 ADA in pool) |
| [3] | Int | `666` | **YES shares sold** |
| [4] | Int | `685` | **NO shares sold** |
| [5] | Int | `498,353` | **YES price** (0.498 = 49.8%) |
| [6] | Int | `501,646` | **NO price** (0.502 = 50.2%) |

**Observations:**
- Prices sum to `999,999 ≈ 1,000,000` → AMM model with normalized prices
- Each price = implied probability × 1,000,000
- V2 GitHub only had `predictions: List<(ByteArray, Int)>` — the AMM model is completely different
- Total shares: 666 + 685 = 1,351
- Total pool / total shares ≈ 1,980,566 lovelace/share average

### Architecture: per-market position script addresses

Key discovery: each market has its **own position script address**, parameterized by the market's `prediction_nft`. The `position_script_hash` in ProjectInfoDatum determines the address.

| Market | Position Script Hash | Address |
|--------|---------------------|---------|
| 9EE9_MENS_BIG_12_QF_I | `571a248a070a60ce...` | `addr1w9t35fy2qu...` |
| (V3 shared) | `9546108bcf596cb5...` | `addr1xx25vyytea...` |

### Impact on tx3

The `main.tx3` was updated to reflect the actual 9-field structure:

1. `PositionDatum` expanded with `pos_admin_fee_percent`, `pos_unit_price`, and `pos_candidate` as `CandidateIdx`
2. New type `CandidateIdx` (enum: `Candidate0`, `Candidate1`)
3. Txs updated: `buy_position` accepts `admin_fee_percent`, `unit_price`, and `candidate: CandidateIdx`
4. `submit_reward` and `submit_refund` use `0` for the extra fields (based on on-chain evidence)
5. total_lovelace formula documented and verified

---

## Priority for tx3 Improvements

Ranked by impact on DeFi protocol coverage:

1. **Dynamic input/output lists (L2)** — Unlocks batcher patterns used by ~80% of Cardano DeFi
2. **Multiplication/division (L1)** — Needed for fee/price calculations in every DeFi protocol
3. **Reference inputs with datum access (L4)** — CIP-31 is fundamental to modern Cardano contracts
4. **Tuple types (L3)** — Common in Aiken type definitions
5. **Conditional logic (L5)** — Needed for multi-asset protocols
