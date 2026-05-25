# Bodega Market - Costs, Fees, and Price Formulas

Reference document with all fee calculation functions, unit price formulas,
reward distribution, and costs found in the Bodega Market smart contracts.

**Sources:**
- [bodega-market-smart-contracts](https://github.com/bodega-market/bodega-market-smart-contracts) (v1, Aiken)
- [bodega-market-smart-contracts-v2](https://github.com/bodega-market/bodega-market-smart-contracts-v2) (v2, Aiken)
- [bodega-market-docs](https://github.com/bodega-market/bodega-market-docs)
- [Staking docs](https://docs.bodegacardano.org/staking)

---

## Global Constants

```aiken
// lib/bodega/utils.ak
pub const decimals = 1_000_000    // 6 decimals (like lovelace)
pub const multiplier = 10_000     // base for percentages (basis points) -- v2 only
```

---

## 1. Unit Price (Price per Share)

**Fixed on-chain price: 1 share = 1,000,000 units of the payment token** (1 ADA or 1 USDM).

### V1 - mint_shares.ak (lines 74-83)

```aiken
let amount =
  when params.payment_asset.policy_id == assets.ada_policy_id is {
    True -> (out_pos_lovelace - required_lovelace) / decimals
    False ->
      assets.quantity_of(
        pos_output.value,
        params.payment_asset.policy_id,
        params.payment_asset.asset_name,
      ) / decimals
  }
```

- For ADA: `shares = (deposited_lovelace - admin_fee - envelope_amount - batcher_fee) / 1,000,000`
- For tokens: `shares = token_amount / 1,000,000`
- Example: 10 USDM (10 * 1,000,000 units) = 10 shares

### V2 - project_shares.ak (lines 76-78)

```aiken
let min_payment = pos_datum.pos_amount * decimals + admin_fee_amount
```

Expanded:
```
min_payment = pos_amount * 1,000,000 + pos_amount * admin_fee_percent * 1,000,000 / 10,000
```

Or equivalently (from test code, line 310):
```aiken
10 * (multiplier + project_info_datum.admin_fee_percent) * decimals / multiplier
```

### Off-Chain Pricing: LMSR (Logarithmic Market Scoring Rule)

The documentation mentions they use Robin Hanson's LMSR model for off-chain pricing:

**Cost function:**
```
C(q) = b * ln( sum_i( e^(q_i / b) ) )
```

**Price (probability) of outcome i:**
```
p_i = e^(q_i / b) / sum_j( e^(q_j / b) )
```

**Cost to buy shares:**
```
Cost = C(q_i + delta_q_i, q_{-i}) - C(q_i, q_{-i})
```

Where:
- `q_i` = total shares in outcome i
- `b` = liquidity parameter (controls sensitivity; higher b = less price impact per trade)
- `p_i` = current price/probability of outcome i

> **Important note:** LMSR pricing is computed off-chain by the frontend/SDK.
> On-chain, the validator verifies that the payment is consistent with the pool state.
> The contracts deployed on mainnet (unlike the open-source v2 repo)
> DO track the LMSR state in the pool datum (prices and shares).

### pos_unit_price: LMSR Price per Share (field[7] of Order Datum)

The `pos_unit_price` field (field[7] of the 9-field on-chain order datum) is the **average
LMSR price per share at the time of order creation**. It is computed off-chain by the frontend.

**Formula:**
```
pos_unit_price = LMSR_cost_total / pos_amount
```

Where `LMSR_cost_total` is obtained from the LMSR cost function:
```
LMSR_cost_total = b * ln(e^(q_new/b) + e^(q_other/b)) - b * ln(e^(q_old/b) + e^(q_other/b))
```

Parameters:
- `b` = liquidity parameter from market info datum field[14] (e.g.: 2885 for market 9EE9)
- `q_old` = current shares on the purchased side (YES or NO, from pool datum)
- `q_new` = q_old + pos_amount (shares after the purchase)
- `q_other` = shares on the opposite side (unchanged)

The result is expressed in **lovelace per share** (base 1,000,000 = 1 ADA).

**On-chain verification (market 9EE9, b=2885):**

| Trade | pos_amount | pos_unit_price (datum) | Actual LMSR cost | cost/amount |
|-------|-----------|------------------------|----------------|-------------|
| 172 YES | 172 | 507,451 | 87,281,612 | 507,451 ✅ |
| 488 NO | 488 | 506,235 | 247,042,743 | 506,235 ✅ |
| 303 YES | 303 | 485,752 | 147,182,965 | 485,754 ≈ ✅ (±2) |
| 191 YES | 191 | 507,147 | 96,865,238 | 507,147 ✅ |
| 197 NO | 197 | 493,631 | 97,142,992 | 493,065 ≈ ✅ (±566) |

**Relationship with the user's total payment:**

```
LMSR_cost    = pool_ada_tracked_after - pool_ada_tracked_before   (actual on-chain value)
admin_fee    = total_fee_after - total_fee_before                 (actual on-chain value)
batcher_fee  = 700,000                                            (fixed in order datum field[5])
min_utxo     = 2,000,000                                          (for token delivery)

total_order  = LMSR_cost + admin_fee + batcher_fee + min_utxo
```

Or equivalently, approximating with pos_unit_price:
```
LMSR_cost    ≈ pos_amount * pos_unit_price                        (approx, ±a few lovelace)
admin_fee    = LMSR_cost * (admin_fee_percent + field[6]) / 10,000
total_order  = LMSR_cost + admin_fee + 700,000 + 2,000,000
```

**Notes:**
- `pos_unit_price` is an **informational/reference** value pre-computed by the frontend
- The actual cost that enters the pool is the full LMSR cost (not exactly amount * price)
- There are ±2 lovelace differences between `amount * price` and the actual LMSR cost due to integer rounding
- For rewards/refunds, `pos_unit_price = 0` (not applicable, payout is fixed at 1 ADA/share)
- The frontend needs to read the current pool datum (yes_shares, no_shares) and `b` from market info
  to compute this value before building the order

---

## 2. Admin Fee

### V2 - Percentage-based fee (current mainnet contracts)

**Formula:**
```aiken
admin_fee_amount = pos_amount * admin_fee_percent * decimals / multiplier
```

Where `admin_fee_percent` is in base 10,000 (e.g.: 500 = 5%).

**Example:** If `admin_fee_percent = 400` (4%) and `pos_amount = 100`:
```
admin_fee = 100 * 400 * 1,000,000 / 10,000 = 4,000,000 lovelace = 4 ADA
```

**Locations in v2 code:**

| File | Line | Context |
|------|------|---------|
| `project_shares.ak` | 73-74 | Share minting (Buy redeemer) |
| `project_prediction.ak` | 187 | Apply redeemer (processing buys in batch) |
| `project_prediction.ak` | 368-369 | Reward redeemer |
| `project_prediction.ak` | 560-561 | Refund redeemer |

**Total purchase cost (v2):**
```aiken
min_payment = pos_amount * decimals + admin_fee_amount
// = pos_amount * 1,000,000 + pos_amount * admin_fee_percent * 1,000,000 / 10,000
// = pos_amount * (10,000 + admin_fee_percent) * 1,000,000 / 10,000
```

### V1 - Fixed fee per position

**Formula:**
```aiken
total_fee = batch_size * admin_fee
```

Where `admin_fee` is a fixed value in lovelace (test value: 200,000 = 0.2 ADA).

**Accumulation:**
```aiken
// predictions.ak line 174
cur_total_fee_output == cur_total_fee_input + batch_size * admin_fee
```

**Locations in v1 code:**

| File | Line | Context |
|------|------|---------|
| `predictions.ak` | 152 | PredApply - ADA payment |
| `predictions.ak` | 170 | PredApply - non-ADA payment |
| `predictions.ak` | 174 | total_fee accumulation |
| `predictions.ak` | 300-318 | PredReward |

---

## 3. Batcher Fee

### V1 - mint_shares.ak (lines 70-73)

```aiken
expect pos_datum.pos_batcher_fee > 0 && pos_datum.pos_amount > 0
let out_pos_lovelace = assets.lovelace_of(pos_output.value)
let required_lovelace =
  pred_datum.admin_fee + pred_datum.envelope_amount + pos_datum.pos_batcher_fee
```

- `pos_batcher_fee`: user-defined, must be > 0
- Test value: 300,000 lovelace (0.3 ADA)
- Compensates the off-chain batcher that processes transactions
- The position output must carry at least `admin_fee + envelope_amount + batcher_fee` in lovelace

---

## 4. Reward Distribution

Winners share the **entire pool** (winners + losers) proportionally to their shares.

### V1 - predictions.ak (line 269)

```aiken
let reward_amount =
  curr_shares * (total_winning + total_losing) * decimals / total_winning
```

Where:
- `curr_shares` = user's shares on the winning side
- `total_winning` = total shares on the winning side
- `total_losing` = total shares on the losing side
- `decimals` = 1,000,000

**Side determination (v1, binary):**
```aiken
let (total_winning, total_losing) =
  if oracle_datum.position_name == own_output_datum.true_position_name {
    (own_output_datum.true_position_amount, own_output_datum.false_position_amount)
  } else {
    (own_output_datum.false_position_amount, own_output_datum.true_position_amount)
  }
```

### V2 - project_prediction.ak (line 371)

```aiken
let reward =
  num_candidate_shares * total_shares * decimals / win_shares
```

Where:
- `num_candidate_shares` = user's shares in the winning candidate
- `total_shares` = sum of ALL shares from all candidates
- `win_shares` = shares only from the winning candidate
- `decimals` = 1,000,000

**Winner determination (v2, multi-candidate, lines 322-337):**
```aiken
let (total_shares, win_shares) =
  list.foldr(
    own_input_datum.predictions,
    (0, 0),
    fn((cand, amount), acc) {
      let (cur_total_shares, cur_win_shares) = acc
      (
        cur_total_shares + amount,
        if oracle_datum.candidate == cand {
          cur_win_shares + amount
        } else {
          cur_win_shares
        },
      )
    },
  )
```

### Conceptual equivalence

```
Payout = original_investment + (original_investment / winner_pool) * loser_pool
```

### Numerical example

- Binary market: YES vs NO
- Total YES shares: 500, Total NO shares: 300
- User holds 100 YES shares, YES wins
- Reward = 100 * (500 + 300) * 1,000,000 / 500 = 160,000,000 lovelace = 160 ADA
- User invested 100 ADA, wins 160 ADA (60 ADA profit)

### Balance equations during reward

**ADA (v2, lines 410-428):**
```aiken
input_lovelace + total_fee == output_lovelace + total_reward
```

**Token (v2):**
```aiken
input_payment + total_fee == output_payment + total_reward
```

### Rounding

Payouts are rounded down (integer truncation). The remainder stays in the contract
as profit for the protocol ("the house").

---

## 5. Refund (v2 only)

### project_prediction.ak (line 562)

```aiken
let refund = num_candidate_shares * decimals
```

- Returns exactly what was paid per share (1,000,000 per share)
- Does **not** return the admin fee
- Used when a market is cancelled before resolution
- Admin fee is also charged on refund:
  ```aiken
  let fee = pos_datum.pos_amount * project_info_datum.admin_fee_percent * decimals / multiplier
  ```

---

## 6. Fee Withdrawal

### V1 - predictions.ak (lines 329-396)

```aiken
// Treasury must receive at least the accumulated fees
assets.lovelace_of(treasury_value) >= own_input_datum.cur_total_fee

// Value conservation
input_lovelace <= output_lovelace + own_input_datum.cur_total_fee

// Counter reset
own_output_datum.cur_total_fee == 0
```

### V2 - project_prediction.ak (lines 679-780)

```aiken
// For ADA:
assets.lovelace_of(treasury_value) >= own_input_datum.total_fee
input_lovelace <= output_lovelace + own_input_datum.total_fee

// For tokens:
input_payment == output_payment + own_input_datum.total_fee

// Reset
own_output_datum.total_fee == 0
```

### Collected fee distribution (per staking docs)

| Destination | Percentage | Description |
|-------------|------------|-------------|
| BODEGA Stakers | 50% | Rewards in ADA |
| Protocol Treasury | 50% | Deployment costs, operations |

- Ratio adjustable by governance
- Total protocol fee is ~4% of total volume

---

## 7. Open Fee (Market Creation Cost)

Defined in `PSettingDatum` (v2):

```aiken
// types.ak
pub type PSettingDatum {
  pledge: Int,                          // required pledge (in BODEGA)
  pledge_policy_id: ByteArray,
  pledge_token_name: ByteArray,
  protocol_treasury_script_hash: ByteArray,
  share_ratio: Int,                     // base 10,000 - NOT USED on-chain
  open_fee: Int,                        // fee to create a market
  open_fee_policy_id: ByteArray,
  open_fee_token_name: ByteArray,
  // ...
}
```

When creating a project:
- `open_fee` is paid to the protocol treasury
- `pledge` in BODEGA tokens is deposited to the project_info script
- Observed on-chain: 50,000 BODEGA as pledge + ~1,002 ADA

---

## 8. Cost-Related Datum Fields

### ProjectInfoDatum (v2)

| Field | Type | Description |
|-------|------|-------------|
| `admin_fee_percent` | Int | Fee percentage, base 10,000 (e.g.: 400 = 4%) |
| `envelope_amount` | Int | Min lovelace per UTXO (~2,000,000 = 2 ADA) |

### PredictionDatum (v1) / ProjectPredictionDatum (v2)

| Field | Type | Description |
|-------|------|-------------|
| `admin_fee` (v1) | Int | Fixed fee per position in lovelace |
| `total_fee` / `cur_total_fee` | Int | Accumulated fees pending withdrawal |

### PositionDatum

| Field | Type | Description |
|-------|------|-------------|
| `pos_amount` | Int | Number of shares |
| `pos_batcher_fee` | Int | Fee for the batcher (user-defined, > 0) |

### PSettingDatum (v2)

| Field | Type | Description |
|-------|------|-------------|
| `share_ratio` | Int | Base 10,000 - defined but NOT used on-chain |
| `open_fee` | Int | Fee to create a market |
| `pledge` | Int | Pledge required to create a market |

---

## 9. Datum Validation (v2)

```aiken
// project_authtoken_mp.ak
pub fn is_project_info_datum_valid(d: ProjectInfoDatum) -> Bool {
  let correct_candidates =
    list.length(list.unique(d.candidates)) == list.length(d.candidates)
  d.deadline > 0 && d.admin_fee_percent > 0 && d.envelope_amount > 0 && correct_candidates
}
```

Constraints:
- `admin_fee_percent` must be > 0 (cannot be free)
- `envelope_amount` must be > 0 (min UTXO)
- Candidates must be unique

---

## 10. Summary of All Formulas

| Operation | Formula | File (v2) |
|-----------|---------|-----------|
| **Price per share** | `1,000,000` units of the payment token | `project_shares.ak` |
| **Admin fee** | `pos_amount * admin_fee_percent * 1,000,000 / 10,000` | `project_shares.ak:74`, `project_prediction.ak:187,369,561` |
| **Total purchase cost** | `pos_amount * 1,000,000 + admin_fee` | `project_shares.ak:78` |
| **Winner reward** | `user_shares * total_pool * 1,000,000 / winning_shares` | `project_prediction.ak:371` |
| **Refund** | `num_shares * 1,000,000` | `project_prediction.ak:562` |
| **Accumulated fee** | `total_fee += sum(batch_fees)` | `project_prediction.ak:221,405,597` |
| **Fee withdrawal** | `treasury receives total_fee; datum.total_fee = 0` | `project_prediction.ak:770-779` |

| Operation | Formula | File (v1) |
|-----------|---------|-----------|
| **Price per share** | `payment_amount / 1,000,000` | `mint_shares.ak:76-83` |
| **Admin fee** | `batch_size * admin_fee` (fixed) | `predictions.ak:152,170,174` |
| **Required lovelace** | `admin_fee + envelope_amount + batcher_fee` | `mint_shares.ak:72-73` |
| **Winner reward** | `user_shares * (winning + losing) * 1,000,000 / winning` | `predictions.ak:269` |
| **Fee withdrawal** | `treasury receives cur_total_fee; datum = 0` | `predictions.ak:387-395` |
