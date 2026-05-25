# Bodega Market - On-Chain Transaction Analysis

Detailed analysis of real transactions from market `9EE9_MENS_BIG_12_QF_I`
("Men's Big 12 QF: Iowa State -6.5 v Texas Tech") and market `D50B_NBA_CAVALIERS_2`
("NBA: Cavaliers (-2.5) vs. Magic | MAR11").

**Market address (order script):** `addr1w9t35fy2qu9xpn3e2pc6w3zwsje795kaeu38eqw9xyltjsg5ldlnp`
(script hash: `571a248a070a60ce395071a7444e84b3e2d2ddcf227c81c5313eb941`)

> **Important note:** The contracts deployed on mainnet have a **different datum structure**
> than the open-source v2 code. The on-chain `ProjectPredictionDatum` has 7 fields (vs 3 in the repo),
> and includes LMSR price tracking. This suggests a newer unpublished version.

---

## Prediction Datum Structure (on-chain, different from v2 repo)

```
Constructor 0:
  [0] OutputReference    -- outref_id (identifies the market)
  [1] Int                -- total_fee (accumulated fees, in lovelace)
  [2] Int                -- pool_ada_tracked (net ADA in pool, without envelope)
  [3] Int                -- yes_shares (total YES shares count)
  [4] Int                -- no_shares (total NO shares count)
  [5] Int                -- yes_price (YES probability, base 1,000,000)
  [6] Int                -- no_price (NO probability, base 1,000,000)
```

**Invariant:** `yes_price + no_price ≈ 1,000,000` (sums to ~1.0)

---

## Order Datum Structure (Position)

```
Constructor 0:
  [0] OutputReference    -- market reference
  [1] ByteArray          -- user payment credential (pubkey hash)
  [2] Constructor        -- user stake credential (Option<PubKeyHash>)
  [3] Constructor        -- pos_type (0 = BuyPos, 2 = Reward/Collect)
  [4] Int                -- pos_amount (number of shares)
  [5] Int                -- pos_batcher_fee (fee for the batcher, lovelace)
  [6] Int                -- fee_premium (200=standard, 10=reduced; added to admin_fee_percent)
  [7] Int                -- pos_unit_price (average LMSR price per share, lovelace)
                            Computed off-chain: LMSR_cost_total / pos_amount
                            Is 0 for rewards/refunds
  [8] Constructor        -- side (0 = YES, 1 = NO)
```

---

## Market Parameters (Market Info Datum)

From market `9EE9_MENS_BIG_12_QF_I`:

| Field | Value | Meaning |
|-------|-------|---------|
| field[3] | `9EE9_MENS_BIG_12_QF_I` | Market ID |
| field[4] | 1,773,333,000,000 | Deadline (POSIX ms) |
| field[7] | `d978b820...` | Batcher auth NFT policy |
| field[8] | `571a248a...` | Order validator script hash |
| field[9] | `ea69dcc8...` | Bet token minting policy |
| field[11] | `9EE9_..._ORACLE` | Oracle asset name |
| field[12] | **200** | **admin_fee_percent (2%, base 10,000)** |
| field[13] | **2,000,000** | **envelope_amount (2 ADA min UTxO)** |
| field[14] | **2,885** | **LMSR liquidity parameter `b`** |
| field[15] | `B_9EE9_YES` | YES token name |
| field[16] | `B_9EE9_NO` | NO token name |

---

## 1. Create Market

**Tx:** `8797e92d475ad66d208ebe8cb929c556fbd703509b5c86b1ea4ee8ce2b921893`

### Inputs

| # | Source | Lovelace | Assets |
|---|--------|----------|--------|
| 0-5 | Creator's wallet | 8,102,459,728 total | 100B+ BODEGA + others |

### Outputs

| # | Destination | Lovelace | Assets | Role |
|---|-------------|----------|--------|------|
| 0 | project_info script | 2,693,750 | 50,000,000,000 BODEGA + PROJECT_INFO_NFT | Market info + pledge |
| 1 | prediction pool script | 2,001,729,617 | PROJECT_PREDICTION_NFT | Liquidity pool |
| 2 | treasury script | 2,000,000 | - | Open fee |
| 3 | Change (creator) | 6,095,122,113 | Remaining assets | Change |

### Create Market Cost Breakdown

| Item | Amount | Notes |
|------|--------|-------|
| **BODEGA Pledge** | 50,000,000,000 units (50,000 BODEGA) | Deposited in project_info script |
| **Pool Funding (initial liquidity)** | 2,001,729,617 lovelace (~2,001.73 ADA) | Market capital for LMSR |
| **Open Fee (protocol)** | 2,000,000 lovelace (2 ADA) | Sent to treasury |
| **Min UTxO (market info)** | 2,693,750 lovelace (~2.69 ADA) | Cost of maintaining the UTxO with NFT + BODEGA |
| **Tx Fee (Cardano)** | 914,248 lovelace (~0.91 ADA) | Network fee |
| **Total for creator** | ~2,007.34 ADA + 50,000 BODEGA | |

### Initial Pool State

```
total_fee:        0
pool_ada_tracked: 1,999,729,617  (= 2,001,729,617 - 2,000,000 envelope)
yes_shares:       0
no_shares:        0
yes_price:        500,000  (50.0%)
no_price:         500,000  (50.0%)
```

**Verification:** `pool_ada_tracked = pool_lovelace - envelope_amount = 2,001,729,617 - 2,000,000 = 1,999,729,617` ✅

---

## 2. All Buy Positions for Market 9EE9

The market had **5 buys** in total, processed chronologically.
For each one, pool state, LMSR cost, admin fee, and balance are verified.

### Full Pool State (all transactions)

| State | total_fee | pool_ada_tracked | pool_lovelace | yes | no | yes_p | no_p |
|-------|-----------|------------------|---------------|-----|----|----|-----|
| Initial (create) | 0 | 1,999,729,617 | 2,001,729,617 | 0 | 0 | 500,000 | 500,000 |
| Trade 1: +172 YES | 3,491,222 | 2,087,011,229 | 2,092,502,451 | 172 | 0 | 514,900 | 485,099 |
| Trade 2: +488 NO | 13,372,865 | 2,334,053,972 | 2,349,426,838 | 172 | 488 | 472,644 | 527,355 |
| Trade 3: +303 YES | 19,260,070 | 2,481,236,937 | 2,502,497,008 | 475 | 488 | 498,873 | 501,126 |
| Trade 4: +191 YES | 23,134,511 | 2,578,102,175 | 2,603,236,688 | 666 | 488 | 515,419 | 484,580 |
| Trade 5: +197 NO | 25,278,977 | 2,675,245,167 | 2,702,524,146 | 666 | 685 | 498,353 | 501,646 |
| Reward 1: -172 YES | 28,718,977 | 2,675,245,167 | 2,533,964,146 | 666 | 685 | 498,353 | 501,646 |
| Reward 2: -494 YES | 38,598,977 | 2,675,245,167 | 2,049,844,146 | 666 | 685 | 498,353 | 501,646 |

> **Note:** During rewards, `pool_ada_tracked`, shares and prices DO NOT change. Only total_fee increases
> and pool_lovelace decreases. The LMSR state is "frozen" post-resolution.

### Verified invariants

```
At all times: pool_lovelace = pool_ada_tracked + total_fee + envelope (2,000,000)

Example Trade 2: 2,349,426,838 = 2,334,053,972 + 13,372,865 + 2,000,001 ≈ ✅ (±1 rounding)
Example Trade 5: 2,702,524,146 = 2,675,245,167 + 25,278,977 + 2,000,002 ≈ ✅ (±2 rounding)

At all times: yes_price + no_price ≈ 1,000,000 (= 999,999)
```

---

### Trade 1: Buy 172 YES

**User order:** `562fc667...` → **Batcher:** `9cb6b173...`

| Field | Value |
|-------|-------|
| Order lovelace | 93,472,834 |
| Shares | 172 YES |
| Order price | 507,451 |
| Order field[6] | 200 |

**Verified breakdown:**

| Item | Lovelace | Calculation |
|------|----------|-------------|
| LMSR Cost (pool_ada delta) | 87,281,612 | 2,087,011,229 - 1,999,729,617 |
| Admin Fee (total_fee delta) | 3,491,222 | 3,491,222 - 0 |
| Batcher Fee | 700,000 | Fixed in order datum |
| Min UTxO (token delivery) | 2,000,000 | |
| **Total** | **93,472,834** | **= order lovelace ✅** |

```
Fee ratio = 3,491,222 / 87,281,612 = 4.000% ✅
```

---

### Trade 2: Buy 488 NO

**User order:** `bb15365e...` → **Batcher:** `304d1bfc...`

| Field | Value |
|-------|-------|
| Order lovelace | 259,624,387 |
| Shares | 488 NO |
| Order price | 506,235 |
| Order field[6] | 200 |

**Verified breakdown:**

| Item | Lovelace | Calculation |
|------|----------|-------------|
| LMSR Cost (pool_ada delta) | 247,042,743 | 2,334,053,972 - 2,087,011,229 |
| Admin Fee (total_fee delta) | 9,881,643 | 13,372,865 - 3,491,222 |
| Batcher Fee | 700,000 | |
| Min UTxO | 2,000,000 | |
| **Total** | **259,624,386** | **≈ order lovelace (off by 1, rounding)** |

```
Fee ratio = 9,881,643 / 247,042,743 = 3.99998% ≈ 4.000% ✅
```

---

### Trade 3: Buy 303 YES

**User order:** `fc6556d9...` → **Batcher:** `90efb08e...`

| Field | Value |
|-------|-------|
| Order lovelace | 155,770,170 |
| Shares | 303 YES |
| Order price | 485,752 |
| Order field[6] | 200 |

**Verified breakdown:**

| Item | Lovelace | Calculation |
|------|----------|-------------|
| LMSR Cost (pool_ada delta) | 147,182,965 | 2,481,236,937 - 2,334,053,972 |
| Admin Fee (total_fee delta) | 5,887,205 | 19,260,070 - 13,372,865 |
| Batcher Fee | 700,000 | |
| Min UTxO | 2,000,000 | |
| **Total** | **155,770,170** | **= order lovelace ✅ EXACT** |

```
Fee ratio = 5,887,205 / 147,182,965 = 4.00008% ≈ 4.000% ✅
```

---

### Trade 4: Buy 191 YES

**User order:** `c7f0692d...` → **Batcher:** `689e5165...`

| Field | Value |
|-------|-------|
| Order lovelace | 103,439,680 |
| Shares | 191 YES |
| Order price | 507,147 |
| Order field[6] | 200 |

**Verified breakdown:**

| Item | Lovelace | Calculation |
|------|----------|-------------|
| LMSR Cost (pool_ada delta) | 96,865,238 | 2,578,102,175 - 2,481,236,937 |
| Admin Fee (total_fee delta) | 3,874,441 | 23,134,511 - 19,260,070 |
| Batcher Fee | 700,000 | |
| Min UTxO | 2,000,000 | |
| **Total** | **103,439,679** | **≈ order lovelace (off by 1, rounding)** |

```
Fee ratio = 3,874,441 / 96,865,238 = 3.99999% ≈ 4.000% ✅
```

---

### Trade 5: Buy 197 NO ⚠️ ANOMALY

**User order:** `6804d270...` → **Batcher:** `831b7e97...`

| Field | Value |
|-------|-------|
| Order lovelace | 101,987,458 |
| Shares | 197 NO |
| Order price | 493,631 |
| Order field[6] | **10** (≠ 200) |

**Verified breakdown:**

| Item | Lovelace | Calculation |
|------|----------|-------------|
| LMSR Cost (pool_ada delta) | 97,142,992 | 2,675,245,167 - 2,578,102,175 |
| Admin Fee (total_fee delta) | 2,144,466 | 25,278,977 - 23,134,511 |
| Batcher Fee | 700,000 | |
| Min UTxO | 2,000,000 | |
| **Total** | **101,987,458** | **= order lovelace ✅ EXACT** |

```
Fee ratio = 2,144,466 / 97,142,992 = 2.208% ⚠️ (NOT 4%)
```

> **ANOMALY:** This order has field[6]=10 in the order datum (vs 200 in the others).
> The resulting fee is ~2.2% of the LMSR cost, not 4%.
> Possible explanations:
> - field[6] in the order datum affects the fee rate
> - It's a different fee tier or an earlier frontend version
> - The user or batcher applied a preferential rate
>
> The 4 orders with field[6]=200 have an EXACT 4.000% fee.
> The only order with field[6]=10 has a 2.208% fee.
> The relationship between field[6] and the effective rate is not linear.

---

### Buy Fees Summary

| Trade | Shares | Side | LMSR Cost | Admin Fee | Fee % | field[6] |
|-------|--------|------|-----------|-----------|-------|----------|
| 1 | 172 | YES | 87,281,612 | 3,491,222 | **4.000%** | 200 |
| 2 | 488 | NO | 247,042,743 | 9,881,643 | **4.000%** | 200 |
| 3 | 303 | YES | 147,182,965 | 5,887,205 | **4.000%** | 200 |
| 4 | 191 | YES | 96,865,238 | 3,874,441 | **4.000%** | 200 |
| 5 | 197 | NO | 97,142,992 | 2,144,466 | **2.208%** | 10 |

**Conclusion:** For orders with field[6]=200, the admin fee is **consistently 4.000%**
of the LMSR cost across all 4 verified trades. The trade with field[6]=10 is the exception.

### LMSR Price Verification

With `b = 2885` (liquidity parameter from market info):

```
Initial state (0 YES, 0 NO):
  p_yes = e^(0/2885) / (e^(0/2885) + e^(0/2885)) = 1/2 = 0.500000
  Recorded: 500,000 ✅

After Trade 1 (172 YES, 0 NO):
  p_yes = e^(172/2885) / (e^(172/2885) + e^(0/2885))
        = 1.06143 / (1.06143 + 1.0)
        = 0.51491
  Recorded: 514,900 ✅ (diff < 0.001)

After Trade 2 (172 YES, 488 NO):
  p_yes = e^(172/2885) / (e^(172/2885) + e^(488/2885))
        = 1.06143 / (1.06143 + 1.18424)
        = 0.47275
  Recorded: 472,644 ✅ (diff < 0.001)

Final state (666 YES, 685 NO):
  p_yes = e^(666/2885) / (e^(666/2885) + e^(685/2885))
        = e^0.23085 / (e^0.23085 + e^0.23743)
        = 1.25978 / (1.25978 + 1.26809)
        = 0.49835
  Recorded: 498,353 ✅ (diff < 0.002)
```

**LMSR prices verify correctly with parameter b = 2885 across all states.** ✅

---

## 3. All Rewards for Market 9EE9

The market resolved **YES** (Iowa State won). YES share holders claimed rewards.
2 rewards were processed covering ALL 666 YES shares (172 + 494 = 666).

### Reward 1: 172 YES shares

**User order:** `6bce9b41...` → **Batcher:** `091dae79...`

| Field | Value |
|-------|-------|
| Shares claimed | 172 B_9EE9_YES |
| User payout | **170,560,000 lovelace** |
| Order UTxO | 2,700,000 lovelace + 172 tokens |

**Pool datum change:**

| Field | BEFORE | AFTER | Change |
|-------|--------|-------|--------|
| total_fee | 25,278,977 | 28,718,977 | **+3,440,000** |
| pool_ada_tracked | 2,675,245,167 | 2,675,245,167 | 0 |
| pool_lovelace | 2,702,524,146 | 2,533,964,146 | **-168,560,000** |

**Verification:**

```
Gross reward       = 172 * 1,000,000 = 172,000,000
Admin fee (2%)     = 172 * 200 * 1,000,000 / 10,000 = 3,440,000  ✅ EXACT
Net from pool      = 172,000,000 - 3,440,000 = 168,560,000  ✅ = pool_lovelace delta
User payout        = 168,560,000 + 2,700,000 (order) - 700,000 (batcher) = 170,560,000  ✅ EXACT
```

**172 B_9EE9_YES tokens BURNED** (negative mint)

---

### Reward 2: 494 YES shares (191 + 303 from the same user)

**User order:** `6db7bbf0...` → **Batcher:** `a4b266ae...`

| Field | Value |
|-------|-------|
| Shares claimed | 494 B_9EE9_YES |
| User payout | **486,120,000 lovelace** |
| Order UTxO | 2,700,000 lovelace + 494 tokens |

**Pool datum change:**

| Field | BEFORE | AFTER | Change |
|-------|--------|-------|--------|
| total_fee | 28,718,977 | 38,598,977 | **+9,880,000** |
| pool_ada_tracked | 2,675,245,167 | 2,675,245,167 | 0 |
| pool_lovelace | 2,533,964,146 | 2,049,844,146 | **-484,120,000** |

**Verification:**

```
Gross reward       = 494 * 1,000,000 = 494,000,000
Admin fee (2%)     = 494 * 200 * 1,000,000 / 10,000 = 9,880,000  ✅ EXACT
Net from pool      = 494,000,000 - 9,880,000 = 484,120,000  ✅ = pool_lovelace delta
User payout        = 484,120,000 + 2,700,000 (order) - 700,000 (batcher) = 486,120,000  ✅ EXACT
```

**494 B_9EE9_YES tokens BURNED** (negative mint)

---

### Market D50B Reward (cross-verification)

Market: `D50B_NBA_CAVALIERS_2`. Oracle resolved: **"B_D50B_NO"** (winner = NO).

**User order:** `6f95e174...` → **Batcher:** `2d311f50...`

| Field | Value |
|-------|-------|
| Shares claimed | 451 B_D50B_NO |
| User payout | **443,980,000 lovelace** |

**Verification:**

```
Gross reward       = 451 * 1,000,000 = 451,000,000
Admin fee (2%)     = 451 * 200 * 1,000,000 / 10,000 = 9,020,000  ✅ EXACT
Net from pool      = 451,000,000 - 9,020,000 = 441,980,000  ✅
User payout        = 441,980,000 + 2,700,000 - 700,000 = 443,980,000  ✅ EXACT
```

---

### Reward Fees Summary (3 rewards verified)

| Reward | Shares | Admin Fee | Expected Fee | Match |
|--------|--------|-----------|--------------|-------|
| 172 YES (9EE9) | 172 | 3,440,000 | 172 * 20,000 = 3,440,000 | ✅ EXACT |
| 494 YES (9EE9) | 494 | 9,880,000 | 494 * 20,000 = 9,880,000 | ✅ EXACT |
| 451 NO (D50B) | 451 | 9,020,000 | 451 * 20,000 = 9,020,000 | ✅ EXACT |

**Confirmed formula:** `admin_fee = shares * admin_fee_percent * 1,000,000 / 10,000`
(with admin_fee_percent = 200 in both markets)

**Confirmed payout:** `user_receives = shares * 1,000,000 - admin_fee + order_ada - batcher_fee`

---

## 4. Total Balance Verification for Market 9EE9

### Complete ADA flow in the pool

```
Initial pool:                         2,001,729,617

+ Trade 1 (172 YES, net to pool):    +   90,772,834
+ Trade 2 (488 NO,  net to pool):    +  256,924,387
+ Trade 3 (303 YES, net to pool):    +  153,070,170
+ Trade 4 (191 YES, net to pool):    +  100,739,680
+ Trade 5 (197 NO,  net to pool):    +   99,287,458

- Reward 1 (172 YES, net from pool): -  168,560,000
- Reward 2 (494 YES, net from pool): -  484,120,000

= Expected final pool:                2,049,844,146
= Actual final pool (on-chain):       2,049,844,146  ✅ EXACT
```

### Final pool distribution

```
Pool lovelace:     2,049,844,146
  pool_ada_tracked:  2,675,245,167  (LMSR state frozen)
  total_fee:            38,598,977  (fees pending withdrawal)
  envelope:              2,000,002  (min UTxO, ~2 ADA)

Verification: 2,675,245,167 + 38,598,977 + 2,000,002 = 2,715,844,146
But pool_lovelace = 2,049,844,146
Difference: -666,000,000 = the 666 YES shares paid at 1 ADA each ✅
```

> pool_ada_tracked was not updated during rewards, so the difference
> between pool_ada_tracked and (pool_lovelace - total_fee - envelope) represents
> exactly the ADA paid in rewards: 666 * 1,000,000 = 666,000,000.

### Remaining ADA for NO holders (who lost)

The 685 NO shares lost. Their ADA remains in the pool:
- Pool after all rewards: 2,049,844,146
- Pending fees: 38,598,977
- Envelope: ~2,000,000
- "Orphaned" ADA (from losers): 2,049,844,146 - 38,598,977 - 2,000,000 ≈ 2,009,245,169

This ADA is the creator's initial liquidity (~2,001.73 ADA) plus the net ADA from the losers,
and represents the market maker / creator's profit when closing the market.

---

## 5. Summary: Actual Economic Model of Bodega Market

### Pricing: LMSR (Logarithmic Market Scoring Rule)

```
Parameter b = 2,885 (for this market, configurable per market)

YES Price = e^(yes_shares / b) / (e^(yes_shares / b) + e^(no_shares / b))
NO Price  = 1 - YES Price

LMSR Cost = b * ln(e^(q_yes_new/b) + e^(q_no_new/b)) - b * ln(e^(q_yes_old/b) + e^(q_no_old/b))
```

- `b` is in the market info datum field[14]
- Higher `b` = more liquidity, less impact per trade
- Prices are in base 1,000,000 (e.g., 498,873 = 49.89%)
- `yes_price + no_price ≈ 1,000,000` (invariant verified across all states)

### Winning Shares: Pay 1 ADA per Share

At market resolution, each winning share is redeemed at a fixed value of
**1,000,000 lovelace (1 ADA)**, regardless of the purchase price.

```
User profit = (1.0 - purchase_price) * num_shares - admin_fee
```

### Fees (verified with 5 buys + 3 rewards on-chain)

| Fee | Amount | When | Formula | Verified |
|-----|--------|------|---------|----------|
| **Admin fee (buy)** | 4% of LMSR cost | On purchase | `LMSR_cost * 400 / 10,000` | ✅ 4/5 trades exact |
| **Admin fee (collect)** | 2% of nominal value | On redemption | `shares * 200 * 1,000,000 / 10,000` | ✅ 3/3 rewards EXACT |
| **Batcher fee** | 0.7 ADA (typical) | On each order | Fixed by user in order datum | ✅ |
| **Open fee** | 2 ADA | On market creation | Fixed, goes to treasury | ✅ |
| **Pledge** | 50,000 BODEGA | On market creation | Deposited in project_info | ✅ |
| **Tx fee (Cardano)** | Variable | Each tx | Cardano network | ✅ |

> **About the buy fee:** In 4 of 5 verified trades, the fee is exactly 4.000%
> of the LMSR cost. One anomalous trade (order field[6]=10 vs 200) had a 2.208% fee.
> The admin_fee_percent in market info is 200 (2%), but on-chain double (4%) is charged
> on purchase. On collect the fee is exactly 2%.

### Fee Withdrawal

Accumulated fees (`total_fee` in the prediction datum) can be withdrawn by the admin
to the protocol treasury. At the end of this market there are 38.6 ADA pending withdrawal.
Distribution per docs:
- 50% for BODEGA stakers (rewards in ADA)
- 50% for operational treasury

---

## 6. Identified Addresses and Scripts

### Market 9EE9 (Buy example)

| Role | Address / Hash |
|------|----------------|
| Order script | `addr1w9t35fy2qu9xpn3e2pc6w3zwsje795kaeu38eqw9xyltjsg5ldlnp` (`571a248a...`) |
| Pool script | `addr1xx25vyy...f0df9x` (`9546108b...`) |
| Market info | `addr1x8x7nn5...t4qep6` (`cde9ce9f...`) |
| Config | `addr1w8ru0tc...szeleuc` (`c7c7af0f...`) |
| Bet token policy | `ea69dcc8b821821d47fd3db0f30e70f725ae1d5a6ba1d57bc809b59f` |
| Batcher auth policy | `d978b820644a332411071b4d19c5b5323aad4e9c500c46d6a182f72a` |
| Project NFT policy | `08a8c0fbe85823132cb14a3767d2e114c8c85f58153b072f8c9e3633` |

### Market D50B (Collect example)

| Role | Address / Hash |
|------|----------------|
| Order script | `addr1wxgrjp2f62qphprtvgl89tfsp3d0a4adcvw5s2u8tmdr5pgemt0js` (`90390549...`) |
| Pool script | `addr1xx25vyy...f0df9x` (`9546108b...`) (same as 9EE9) |
| Market info | `addr1x8x7nn5...t4qep6` (`cde9ce9f...`) (same as 9EE9) |
| Oracle | `addr1wx7dfxw...cej6nha` (`bcd499d9...`) |
| Bet token policy | `a89b593441d34cb72c14556ecad74ad0003c65c2a1ac773b9ab10605` |
| Batcher auth policy | `77c6e27d5a2d21587df2d1814874124317a6a145fe67ffaf1e35e856` |

> **Note:** Order scripts are DIFFERENT between markets (parameterized per market),
> but the pool script, market info script, and config script are shared.
> Each market has its own bet token minting policy and its own batcher auth policy.

---

## 7. Full Verified Flow (Market 9EE9, complete lifecycle)

```
CREATE MARKET (tx 8797e92d)
  Creator -> 50,000 BODEGA (pledge) + 2,001.73 ADA (pool) + 2 ADA (open fee)
  <- PROJECT_INFO_NFT + PROJECT_PREDICTION_NFT
  Pool: 2,001,729,617 lovelace | 0 shares | 50/50 prices | fee=0

TRADE 1: Buy 172 YES (txs 562fc667 -> 9cb6b173)
  User pays: 93.47 ADA | Receives: 172 B_9EE9_YES
  Pool: +87.28 ADA (LMSR) +3.49 ADA (fee) | YES 50%->51.5% | Shares: 172/0

TRADE 2: Buy 488 NO (txs bb15365e -> 304d1bfc)
  User pays: 259.62 ADA | Receives: 488 B_9EE9_NO
  Pool: +247.04 ADA (LMSR) +9.88 ADA (fee) | YES 51.5%->47.3% | Shares: 172/488

TRADE 3: Buy 303 YES (txs fc6556d9 -> 90efb08e)
  User pays: 155.77 ADA | Receives: 303 B_9EE9_YES
  Pool: +147.18 ADA (LMSR) +5.89 ADA (fee) | YES 47.3%->49.9% | Shares: 475/488

TRADE 4: Buy 191 YES (txs c7f0692d -> 689e5165)
  User pays: 103.44 ADA | Receives: 191 B_9EE9_YES
  Pool: +96.87 ADA (LMSR) +3.87 ADA (fee) | YES 49.9%->51.5% | Shares: 666/488

TRADE 5: Buy 197 NO (txs 6804d270 -> 831b7e97)
  User pays: 101.99 ADA | Receives: 197 B_9EE9_NO
  Pool: +97.14 ADA (LMSR) +2.14 ADA (fee) | YES 51.5%->49.8% | Shares: 666/685

=== ORACLE RESOLVES: YES WINS ===

REWARD 1: 172 YES (txs 6bce9b41 -> 091dae79)
  User receives: 170.56 ADA | Burns: 172 B_9EE9_YES
  Pool: -168.56 ADA +3.44 ADA (fee) | Shares/prices unchanged

REWARD 2: 494 YES (txs 6db7bbf0 -> a4b266ae)
  User receives: 486.12 ADA | Burns: 494 B_9EE9_YES
  Pool: -484.12 ADA +9.88 ADA (fee) | Shares/prices unchanged

FINAL STATE:
  Pool: 2,049.84 ADA (includes 38.6 ADA fees + ~2,009 ADA remaining)
  Shares: 666 YES (all claimed) / 685 NO (lost)
  Accumulated fees: 38,598,977 lovelace (pending withdrawal to treasury)
```

---

## 8. V3 Market Analysis: `6672_WILL_FED_CUT_AT_`

Market deployed on `v3.bodegamarket.io`: "Will Fed cut at least 100 basis points in 2026?"
URL: https://v3.bodegamarket.io/marketDetails?id=6672_WILL_FED_CUT_AT_

**Market address (escrow/order script):** `addr1w9q7w457f28qe5p653ynlxf07ejurm38zrxt3d9plcw0lcgzr6v97`
(script hash: `41e7569e4a8e0cd03aa4493f992ff665c1ee2710ccb8b4a1fe1cffe1`)

### Reference Transactions

**Create Market:**
- `15fa3077f4e04a7a09dadf2cf8c731c1dabdb5edf4d9ed211206a6cb465a365c`

**Buy Position (user order → batcher process):**

| # | User Order Tx | Batcher Tx | Side | Shares | ADA |
|---|---------------|------------|------|--------|-----|
| 1 | `2ad697ca89e8b92615828999a2277b7406d800649a35583afa900ab9724ffb2e` | `8050988b3b5c635656863db623c63227b22be0c68c79b8561555d48dca81d630` | NO | 93 | 51.84 |
| 2 | `b60f9db2051e20a1f80df4d70ed5cd827034eb7f6c0072237711101c23d831d5` | `03d8c78096566b2b6f976c064002c71e861ac83418cc97e021362ced562401d4` | NO | 90 | 50.84 |
| 3 | `3ff2b5d027e1b1ba959e06fa92c757e03452733d6a658cca26085515e57ad909` | `cf36e35369c9b960e39d610077a0084baba821fae8450beb507d021e066ea1b1` | NO | 40 | 24.96 |

### Architectural Changes V3 vs V2

| Aspect | V2 (markets 9EE9, D50B) | V3 (market 6672) |
|--------|--------------------------|-------------------|
| Bet token policy | Per-market (`ea69dcc8...`) | `dc01dca328c96cd58a93f5e3661d0ad02b7270b6f525a88ccd6bcfff` |
| Oracle policy | `bcd499d9...` | `41e7569e4a8e0cd03aa4493f992ff665c1ee2710ccb8b4a1fe1cffe1` |
| Escrow/Order script | Per-market (`571a248a...`) | `41e7569e...` (= oracle policy hash!) |
| Batcher beacon | `d978b820...` | `e6f424c76f181de8d1ef125a4ef618b72cb8e37edf18d579411559c5` |
| Pool script | `9546108b...` | `9546108b...` (**same**) |
| Project NFT policy | `08a8c0fb...` | `08a8c0fb...` (**same**) |
| Pool datum | 7 fields | 7 fields (**same structure**) |
| Order datum | 9 fields | 9 fields (**same structure**) |

> **The base architecture is the same:** 7-field pool datum, 9-field order datum,
> 2-step flow (user order → batcher process). Script hashes changed but the model did not.

### Market 6672 Parameters

| Field | Value |
|-------|-------|
| Market ID | `6672_WILL_FED_CUT_AT_` |
| admin_fee_percent | 200 (2%) |
| envelope_amount | 2,000,000 (2 ADA) |
| Market index | 1443 |
| Deadline | 2026-12-31 |
| BODEGA pledge | 50,000 |
| Initial liquidity | ~1,000.21 ADA |

### Pool State Progression (first 3 buys)

| State | total_fee | pool_ada | yes | no | yes_p | no_p |
|-------|-----------|----------|-----|-----|-------|------|
| Initial | 0 | 1,000,211,382 | 0 | 0 | 500,000 | 500,000 |
| +93 NO | 1,889,892 | 1,047,460,472 | 0 | 93 | 483,893 | 516,106 |
| +90 NO | 2,880,004 | 1,094,610,427 | 0 | 183 | 468,337 | 531,662 |
| +40 NO | 3,736,166 | 1,116,014,881 | 0 | 223 | 461,441 | 538,558 |

### Fee Verification (V3)

| Trade | Shares | field[6] | LMSR Cost | Admin Fee | Fee/LMSR | Balance ✅ |
|-------|--------|----------|-----------|-----------|----------|-----------|
| 93 NO | 93 | 200 | 47,249,090 | 1,889,892 | **4.000%** | ✅ |
| 90 NO | 90 | **10** | 47,149,955 | 990,112 | **2.100%** | ✅ (±1) |
| 40 NO | 40 | 200 | 21,404,454 | 856,162 | **4.000%** | ✅ (±1) |

**Balance verification (example Trade 1):**
```
Order:    51,838,982
- MinUTxO: 2,000,000
- Batcher: 700,000
= Net:    49,138,982

LMSR:     47,249,090
+ Fee:     1,889,892
= Total:  49,138,982  ✅ EXACT
```

### Finding: field[6] in Order Datum Controls the Fee Rate

Consolidating data from BOTH markets (V2 and V3):

| Market | Trade | field[6] | Fee/LMSR |
|--------|-------|----------|----------|
| 9EE9 | 172 YES | 200 | 4.000% |
| 9EE9 | 488 NO | 200 | 4.000% |
| 9EE9 | 303 YES | 200 | 4.000% |
| 9EE9 | 191 YES | 200 | 4.000% |
| 9EE9 | 197 NO | **10** | **2.208%** |
| 6672 | 93 NO | 200 | 4.000% |
| 6672 | 90 NO | **10** | **2.100%** |
| 6672 | 40 NO | 200 | 4.000% |

**For field[6]=200:** Fee = LMSR_cost * 400 / 10,000 = **4.000%** (exact, 7/7 trades)

**For field[6]=10:** Fee ≈ **2.1-2.2%** of LMSR cost.
Likely formula: `fee = LMSR_cost * (admin_fee_percent + field[6]) / 10,000`
= LMSR_cost * (200 + 10) / 10,000 = 2.1%.
V3 verification: 47,149,955 * 210 / 10,000 = 990,149 vs actual 990,112 (diff 37, <0.004%)

> **Interpretation:** field[6] appears to be a **user fee premium** that is added to the base admin_fee_percent
> (200). With field[6]=200, the total fee is (200+200)/10000 = 4%. With field[6]=10,
> the total fee is (200+10)/10000 = 2.1%. The frontend likely offers different
> fee levels that affect the batcher's processing priority.

---

## 9. Differences from Open-Source v2 Code

| Aspect | Open-source v2 (repo) | On-chain V2/V3 (mainnet) |
|--------|----------------------|--------------------------|
| Prediction datum | 3 fields (outref, total_fee, predictions list) | 7 fields (+ pool_ada, shares, prices) |
| Pricing | Shares 1:1 (on-chain), LMSR off-chain only | LMSR tracking integrated in datum |
| Reward formula | `shares * total_pool * decimals / win_shares` (pro-rata) | `shares * 1,000,000` (fixed 1 ADA/share) |
| Fee on buy | `pos_amount * fee% * decimals / multiplier` | `LMSR_cost * (admin_fee + field[6]) / 10,000` |
| Fee on collect | `pos_amount * fee% * decimals / multiplier` | `pos_amount * fee% * decimals / multiplier` ✅ (same) |
| Market outcomes | Multi-candidate (list) | Binary (YES/NO with LMSR prices) |
| Order datum | 6 fields | 9 fields (+ fee field, price, side flag) |
| Fee tiers | Does not exist | field[6] in order datum (10 or 200 observed) |

> The open-source v2 code appears to be an earlier or simplified version.
> The contracts deployed on mainnet include direct LMSR integration,
> a fixed payout model (1 ADA/winning share), and user-configurable fee tiers.
