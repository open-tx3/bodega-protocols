# Bodega Market - Research Report

## Overview

Bodega Market is an open-source prediction market protocol built on the Cardano blockchain using Aiken smart contracts (PlutusV3). Users can create and participate in prediction markets with flexible payment tokens and volume-based fee sharing.

- Website: https://www.bodegamarket.xyz/
- Docs: https://docs.bodegacardano.org
- Twitter: https://x.com/BodegaCardano

## GitHub Organization: bodega-market

6 public repositories:

| Repo | Language | Description |
|------|----------|-------------|
| bodega-market-smart-contracts | Aiken | V1 contracts (alpha, 0.1.0) - 3 validators |
| bodega-market-smart-contracts-v2 | Aiken | V2 contracts (beta) - 11 validators |
| bodega-market-docs | Markdown | Documentation |
| market-cap | TypeScript | Fork - Cardano native token market cap |
| minswap-tokens | TypeScript | Fork - Token list |
| cardano-token-registry | Nix | Fork - Cardano token registry |

---

## V1 Contracts (bodega-market-smart-contracts)

- **Compiler:** Aiken v1.1.9+2217206
- **Plutus version:** v3
- **aiken.toml name:** `aiken-lang/bodega-market`
- **Dependencies:** aiken-lang/stdlib v2.1.0
- **plutus.json:** YES

### Validators (3)

| Validator | Hash | Type |
|-----------|------|------|
| mint_shares.shares | `00dca98d8920f9a935b18885ded0fd5ec1bf2e671c8f7b1898005de1` | mint |
| positions.positions | `68a9548220273050e0d74b710bd6f2b85c9c84dfbcb4d70debc3df3f` | spend |
| predictions.predictions | `7ba49d96cb70adc007e48a9a6ea681f6977a01cea85a6d29050cd300` | spend |

### V1 Type Definitions

**PositionDatum:**
- pos_user_pkh: PubKeyHash
- pos_user_stake_key: Option<PubKeyHash>
- pos_amount: Int
- pos_batcher_fee: Int
- pos_type: PositionType (BuyPos | RewardPos)
- pos_side: PositionSide (SideTrue | SideFalse)

**PositionRedeemer:**
- pred_in_idx: Int

**PredictionDatum:**
- true_position_name: ByteArray
- false_position_name: ByteArray
- dead_line: Int
- true_position_amount: Int
- false_position_amount: Int
- position_script_hash: ScriptHash
- admin_fee: Int
- cur_total_fee: Int
- envelope_amount: Int

**PredictionParams:**
- prediction_nft: Asset
- oracle_nft: Asset
- share_policy_id: PolicyId
- payment_asset: Asset
- license_symbol: PolicyId
- maximum_deadline_range: Int
- treasury_script_hash: ScriptHash

**PredictionRedeemer:** PredApply | PredReward | PredWithdrawAdminFee

**ShareRedeemer:** Buy | Reward

**ShareParams:**
- (not detailed in V1 types)

**OracleDatum:**
- position_name: ByteArray

---

## V2 Contracts (bodega-market-smart-contracts-v2) -- CURRENT/ACTIVE

- **Compiler:** Aiken v1.1.12+0da4f70
- **Plutus version:** v3
- **aiken.toml name:** `aiken-lang/market-contracts-beta`
- **Dependencies:** aiken-lang/stdlib v2.1.0
- **plutus.json:** YES

### File Structure

```
validators/
  batcher_mp.ak
  project_authtoken_mp.ak
  project_info.ak
  project_position.ak
  project_prediction.ak
  project_shares.ak
  protocol_authtoken_mp.ak
  protocol_manager.ak
  protocol_settings.ak
  protocol_treasury.ak
  ref_script_lock.ak
lib/bodega/
  constants.ak
  test_constants.ak
  types.ak
  utils.ak
```

### Constants (lib/bodega/constants.ak)

```
psetting_nft_tn = "PROTOCOL_SETTINGS_NFT"
pmanager_nft_tn = "PROTOCOL_MANAGER_NFT"
project_info_nft_tn = "PROJECT_INFO_NFT"
project_prediction_nft_tn = "PROJECT_PREDICTION_NFT"
```

### Utility Constants (lib/bodega/utils.ak)

```
decimals = 1_000_000
multiplier = 10_000
```

### Validators (11 validators, 18 endpoints)

| # | Validator | Hash | Endpoints |
|---|-----------|------|-----------|
| 1 | batcher_mp.batcher_mp | `8ded994ef2379595cf0559758508670af5aa0ba9a3e198bb4002924a` | mint, else |
| 2 | project_authtoken_mp.project_authtoken_mp | `5a7f5f9a82c983960ba4adb711b537755af56d7a925f58ace6b68d16` | mint, else |
| 3 | project_info.project_info | `5aac1d4092fa314c8bac768202e161802efaa535f7a2d327f2a24302` | spend, else |
| 4 | project_position.positions | `319c02f3da1c83263b29a3bd6fd201f08c90b79cdddd90012b5285f3` | spend, else |
| 5 | project_prediction.project_prediction | `385d559ccf306bfcedc2644ec6574ebf63e59442a6bba0571d2a93e9` | spend, else |
| 6 | project_shares.project_shares | `5dc32a01eff1e7c55b3c53caa7f7c20eeb6be3f3c0ab50c22ed9618d` | mint, else |
| 7 | protocol_authtoken_mp.protocol_authtoken_mp | `8fc7ff697d18a246fa496044f7048426cc5b9bad1cddfeceabdb45ad` | mint, else |
| 8 | protocol_manager.protocol_manager | `075677d86b7534e26c93989e36d0fbf2876eb3ca13aa5665183a4ae7` | spend, withdraw, else |
| 9 | protocol_settings.protocol_settings | `319c02f3da1c83263b29a3bd6fd201f08c90b79cdddd90012b5285f3` | spend, else |
| 10 | protocol_treasury.protocol_treasury | `319c02f3da1c83263b29a3bd6fd201f08c90b79cdddd90012b5285f3` | spend, else |
| 11 | ref_script_lock.ref_script_lock | `3dc3c0deca6f832bd41dcf76dc3d366737726242ec8420f83c4a139a` | spend, mint, else |

**NOTE:** Validators #4, #9, #10 share the same hash `319c02f3...` - they are parameterized validators with the same compiled code but different runtime parameters.

### V2 Type Definitions

**ProjectInfoDatum:**
- outref_id: OutputReference
- owner_pkh: ByteArray
- owner_stake_key: Option<PubKeyHash>
- project_name: ByteArray
- deadline: Int
- payment_policy_id: ByteArray
- payment_token_name: ByteArray
- batcher_policy_id: ByteArray
- position_script_hash: ByteArray
- share_policy_id: ByteArray
- oracle_policy_id: ByteArray
- oracle_token_name: ByteArray
- admin_fee_percent: Int (base 10,000)
- envelope_amount: Int
- candidates: List<ByteArray>

**ProjectPredictionDatum:**
- outref_id: OutputReference
- total_fee: Int
- predictions: List<(ByteArray, Int)>

**PSettingDatum:**
- pledge: Int
- pledge_policy_id: ByteArray
- pledge_token_name: ByteArray
- protocol_treasury_script_hash: ByteArray
- share_ratio: Int (base 10,000)
- open_fee: Int
- open_fee_policy_id: ByteArray
- open_fee_token_name: ByteArray
- project_authtoken_policy_id: ByteArray
- project_info_token_name: ByteArray
- project_prediction_token_name: ByteArray
- protocol_stake_key_hash: ByteArray

**PositionDatum:**
- outref_id: OutputReference
- pos_user_pkh: PubKeyHash
- pos_user_stake_key: Option<PubKeyHash>
- pos_type: PositionType (BuyPos | RewardPos | RefundPos)
- pos_amount: Int
- pos_batcher_fee: Int
- pos_candidate: ByteArray

**OracleDatum:**
- candidate: ByteArray

**ShareRedeemer:** Buy { project_info_ref_idx } | Reward { pred_in_idx } | Refund { pred_in_idx }

**ShareParams:**
- project_info_nft: Asset
- project_prediction_nft: Asset

**ProjectPredictionRedeemer:** Apply | Reward | Refund | WithdrawFee | Close

**PMSpendRedeemer (project_info):** CreatorUpdate { own_in_idx, own_out_idx, protocol_manager_in_idx } | AdminUpdate { protocol_manager_in_idx }

**LicenseRedeemer (batcher_mp):** { protocol_manager_in_idx, dead_line }

**PATMintRedeemer (project_authtoken):** { seed, protocol_setting_ref_idx, project_info_out_idx, project_prediction_out_idx, protocol_treasury_out_idx }

### Validator Parameters

| Validator | Parameters |
|-----------|-----------|
| batcher_mp | protocol_manager_nft: Asset, _project_name: ByteArray |
| project_authtoken_mp | project_info_script_hash, project_prediction_script_hash, psettings_nft_policy_id, psettings_nft_tn (all ByteArray) |
| project_info | protocol_manager_nft: Asset |
| project_position (positions) | prediction_nft: Asset |
| project_prediction | psettings_nft_policy_id, psettings_nft_tn (ByteArray) |
| project_shares | params: ShareParams (project_info_nft + project_prediction_nft) |
| protocol_authtoken_mp | seed: OutputReference |
| protocol_manager | admin_pkh: ByteArray |
| protocol_settings | pmanager_nft: Asset |
| protocol_treasury | pmanager_nft: Asset |
| ref_script_lock | protocol_manager_nft: Asset |

---

## Deployed Contracts - Mainnet

### BODEGA Token
- **Policy ID:** `5deab590a137066fef0e56f06ef1b830f21bc5d544661ba570bdd2ae`
- **Asset Name:** BODEGA (hex: `424f44454741`)
- **Fingerprint:** asset1f8paxp0vuytlw37aqpcnufx4uge8y3kakvqqkk
- **Decimals:** 6
- **Total Supply:** 25,000,000
- **Mint Tx:** `793c4cde84ef33aed66ad76822585d11d068ef4b670c762363ece53731b05912`

### Note on V2 / V3 naming

DefiLlama labels both sets of addresses as using the same contracts, but on-chain analysis
(see `bodega-market-onchain-analysis.md`) reveals **structural differences** in the deployed
datum formats (7-field vs 3-field PredictionDatum, 9-field vs 7-field PositionDatum, LMSR
tracking integrated on-chain). The open-source `bodega-market-smart-contracts-v2` repo only
covers V2. V3 is a newer, unpublished version with different validators and datum structures,
parameterized with a different stake key.

### V2 (stake key: `stake1786p7hr...`)

**Market Creation (project_info):**
- Address: `addr1xy397mvr7dcm9a0jlevdh78d2vxse5upewend0m76rkw6ch5rawx2k37hx9mjk6pr0n0fg4rp7sswpv7pywfpvvuj6ks3rknx4`
- Script Hash (HEX): `31225f6d83f371b2f5f2fe58dbf8ed530d0cd381cbb336bf7ed0eced`
- Balance: ~391 ADA
- Transactions: 3,350
- Stake key: `stake1786p7hr9tgltnzaetdq3heh5523slgg8qk0qj8yskxwfdtgxgayap`

**Market Positions (project_position):**
- Address: `addr1x99yh3eglqg320ee4yeefvafc7h9fk7khk8xwp5hqcq524l5rawx2k37hx9mjk6pr0n0fg4rp7sswpv7pywfpvvuj6ksj58rfr`
- Script Hash (HEX): `314a4bc728f811153f39a93394b3a9c7ae54dbd6bd8e67069706014557f41f5c655a3eb98bb95b411be6f4a2a30fa107059e091c90b19c96ad`
- Balance: ~1,533 ADA
- Transactions: 12,071
- Stake key: `stake1786p7hr9tgltnzaetdq3heh5523slgg8qk0qj8yskxwfdtgxgayap`

### V3 — most active (stake key: `stake17x50lxk2...`)

**Market Creation (project_info):**
- Address: `addr1x8x7nn5lch2uawxct2hjr06kgsplxu9rpm8gg9tyffv4u8agl7dv5f8n43cyguzxv2xu3vnqdynvj5nxu2yere25k4kqt4qep6`
- Script Hash (HEX): `31cde9ce9fc5d5ceb8d85aaf21bf564403f370a30ece8415644a595e1f`
- Balance: ~75 ADA
- Transactions: 3,857
- Stake key: `stake17x50lxk2yne6cuzywprx9rwgkfsxjfkf2fnw9zv3u42t2mqcwnc7y`

**Market Positions (project_position):**
- Address: `addr1xx25vyyteavkeddsueufzr4ahgsa987fafvhv032tnmvg0dgl7dv5f8n43cyguzxv2xu3vnqdynvj5nxu2yere25k4kqf0df9x`
- Script Hash (HEX): `319546108bcf596cb5b0e678910ebdba21d29fc9ea59763e2a5cf6c43d`
- Balance: ~58,443 ADA
- Transactions: 18,318
- Stake key: `stake17x50lxk2yne6cuzywprx9rwgkfsxjfkf2fnw9zv3u42t2mqcwnc7y`

### Activity Summary

| Version | Role | Txs | ADA Balance |
|---------|------|-----|-------------|
| V2 - Market Creation | project_info | 3,350 | 391 |
| V2 - Market Positions | positions | 12,071 | 1,533 |
| V3 - Market Creation | project_info | 3,857 | 75 |
| V3 - Market Positions | positions | 18,318 | 58,443 |

**V3 is the most active** with 22,175 combined txs and ~58,518 ADA locked.

---

## Transaction Flows

1. **Protocol Initialization** - Mint protocol_authtoken (PROTOCOL_MANAGER_NFT + PROTOCOL_SETTINGS_NFT), set up protocol_manager and protocol_settings UTxOs
2. **Project Creation** - Mint project_authtoken (PROJECT_INFO_NFT + PROJECT_PREDICTION_NFT), create project_info and project_prediction UTxOs, pay open_fee to treasury
3. **Buy Positions** - User creates position UTxO at positions script, mints share tokens via project_shares
4. **Batch Buy Processing** - Batcher (licensed via batcher_mp) processes position UTxOs, updates project_prediction amounts
5. **Reward Positions** - After oracle resolution, burn winning shares and distribute rewards proportionally
6. **Batch Reward Processing** - Batcher processes reward distributions
7. **Fee Withdrawal** - Admin withdraws accumulated fees from project_prediction to protocol_treasury
8. **Project Closure** - Close market, burn remaining tokens

---

## Testnet

- **Testnet URL:** https://beta.bodegacardano.org (redirects to https://beta.bodegamarket.xyz/)
- According to the [testnet guide](https://docs.bodegacardano.org/testnet-guide), beta.bodegacardano.org corresponds to the testnet environment
- **Status:** unused for ~1 year, possibly abandoned or outdated
- No testnet contract addresses found in public repos or DefiLlama

---

## Fee Structure

- Admin fee: configurable per project (base 10,000 = 100%)
- Revenue split: 75% to BODEGA holders (before Jun 2025), 100% after
- Protocol fee: 4% of total volume
- Open fee: configurable in PSettingDatum
- Pledge required to create markets

---

## Security

- V1 mainnet was unaudited, intentionally centralized for safety
- V2/V3 audit status: audit partner announcements pending
- Protocol manager is admin-controlled (admin_pkh signature required)
- Oracle resolution is centralized (admin sets oracle datum)

---

## On-Chain Verification (2026-03-11)

### V3 - Positions Address - Confirmed Active
- Address: `addr1xx25vyyteavkeddsueufzr4ahgsa987fafvhv032tnmvg0dgl7dv5f8n43cyguzxv2xu3vnqdynvj5nxu2yere25k4kqf0df9x`
- Verified balance: **58,443.060727 ADA**
- Verified txs: **18,318**
- Stake key: `stake17x50lxk2yne6cuzywprx9rwgkfsxjfkf2fnw9zv3u42t2mqcwnc7y` (Not Active)
- Token holdings: None (positions are created/consumed per market)

### V3 - Market Creation Address - Confirmed Active
- Address: `addr1x8x7nn5lch2uawxct2hjr06kgsplxu9rpm8gg9tyffv4u8agl7dv5f8n43cyguzxv2xu3vnqdynvj5nxu2yere25k4kqt4qep6`
- Verified balance: **75.78704 ADA**
- Verified txs: **3,857**
- Holds: BODEGA tokens + PROJECT_INFO_NFT
- Recent activity: systematic transfers of 50,000 BODEGA tokens

### Example Transaction Analysis

**Tx:** `9045a7a4d701cfaef19f5bbc7f517009946115c6a1fe34b358f96b7b1aced481`
**Type:** Process Reward Positions
**Market:** "WCC Championship: #12 Gonzaga -6.5 vs Santa Clara"

**Inputs:**
- V3 positions script: 78.4 ADA (with datum)
- Batcher wallet `addr1w9puyxdnzn3upaj73mk8lq0cq85jwrlu0kt62dw8dhka4lgjvdkjs`: 2.7 ADA + 80 B_FB01_YES tokens

**Outputs:**
- User reward payout: 80.4 ADA
- V3 positions script: updated datum (PROJECT_PREDICTION_NFT + tokens)
- Batcher fee output: 23.5 ADA

**Minting/Burning:**
- Policy `3428b8fcc0bdbbbf037c0f4ca97e88a846ec024de09ed07933789c05`: burned 80 B_FB01_YES share tokens

**Datum CBOR (positions output):**
```
d8799fd8799f5820cb9485fd19d817e10f3e7b55bdbe904d9dd05e16e0ffefefe5d3ccf6b1e0761603ff1a020c98b51a98708def190401001a0008f87a1a000649c5ff
```

**Datum CBOR (batcher output):**
```
d8799fd8799f5820cb9485fd19d817e10f3e7b55bdbe904d9dd05e16e0ffefefe5d3ccf6b1e0761603ff581c8ed579d16fbdf94543bacf7e44380799f2c097b61ceaed9544901a84d8799f581c89ff300b867176dc343ddfb677fcde5cd6c8abd1253ecd563e9c594affd87b8018501a000aae600000d87980ff
```

**Observations:**
- Share tokens follow naming convention `B_{MARKET_ID}_{CANDIDATE}` (e.g. `B_FB01_YES`)
- Batcher processes reward distributions, burning winner share tokens
- Two PlutusV3 SPEND redeemers executed in the tx
- The share token policy `3428b8fcc0bdbbbf...` is a per-project instantiation of `project_shares`

---

## Testnet Deployments

- **Testnet URL:** https://beta.bodegacardano.org (redirects to https://beta.bodegamarket.xyz/)
- According to the [testnet guide](https://docs.bodegacardano.org/testnet-guide), this site corresponds to the testnet environment
- **Status:** unused for ~1 year, possibly abandoned or outdated
- No testnet contract addresses found in public repos, docs, or DefiLlama
- The V2 repo does not include deployment scripts, .env files, or testnet configurations
- Testing on testnet would require deploying from plutus.json with test parameters

---

## Key Observations for tx3 Implementation

1. The protocol is heavily parameterized - most validators take NFT assets or script hashes as parameters
2. The protocol_authtoken_mp uses a seed OutputReference (one-shot minting pattern)
3. Share tokens represent prediction positions - minted on buy, burned on reward/refund
4. The batcher pattern uses license NFTs with deadline-based expiry
5. V2 adds RefundPos position type and multi-candidate support (vs V1 binary true/false)
6. All script addresses use the same stake key per version (V2 and V3 have different stake keys and different deployed code)
7. The positions and predictions validators are the core - most complex logic is in project_prediction
8. **plutus.json is available** in both V1 and V2 repos - types can be mapped directly to tx3 records
9. Share tokens are per-project (each market mints its own share token with unique policy)
10. V3 is the most active on mainnet (22k+ txs, 58k+ ADA locked)

## Reference Transactions (Mainnet)

Example transactions by operation type, for use as reference when implementing in tx3.
All correspond to V3 (the most active) unless marked as V2.
V2 matches the open-source `smart-contracts-v2` repo; V3 uses a newer unpublished version.

### 1. Process Trade Positions (Batch Apply + Buy)

Batcher processes buy positions: consumes position UTxOs, updates prediction datum, mints share tokens.

| Tx Hash | Market | Share Token | Action |
|---------|--------|-------------|--------|
| `90985801cf0588d5ea96bcc6232c043def95497e7a3aeffd2bff05bafd2b3119` | Will ADA close above $0.27 on March 20th 2026? | Mint 93 B_9737_YES | Buy YES |
| `56ef22422ffb76f8fae0a254718575b04a1209c14e0f16a0b200f539b06c9b10` | Will ADA close above $0.27 on March 20th 2026? | Mint 401 B_9737_NO | Buy NO |
| `013981d8e3c3bbe6de4a26ebd0e18c7e5fe4d26602d9514827ede16c014ee233` | NBA: Cavaliers (-2.5) vs. Magic \| MAR11 | Mint 185 B_D50B_YES | Buy YES |
| `a104187545a60d8ffffd179f389f4e09406ae5e169a8f376048c82e09e98e15a` | NBA: Cavaliers (-2.5) vs. Magic \| MAR11 | Mint 451 B_D50B_NO | Buy NO |
| `e84e15f2a8ae53c40fa703667b487b9b16f64bdd3744bd255a4c57cd9febfa69` | Real Madrid win or draw vs Man City & goals U 3.5 | Mint 342 B_33B7_YES | Buy YES |

**Pattern:** 3 redeemers (2x SPEND on position + prediction, 1x MINT on shares). Metadata label 674 = "Bodega Market - Process Trade Positions". Label 721 = NFT metadata with IPFS images.

### 2. Process Reward Positions

Batcher processes post-resolution rewards: burns winner share tokens, distributes ADA proportionally.

| Tx Hash | Market | Share Token | Action |
|---------|--------|-------------|--------|
| `9045a7a4d701cfaef19f5bbc7f517009946115c6a1fe34b358f96b7b1aced481` | WCC Championship: #12 Gonzaga -6.5 vs Santa Clara | Burn 80 B_FB01_YES | Reward |
| `c1ef588e04a8266cfb0dd6ff28dbc8690c0ffc49152cfeb90d430610f91aa9c2` | WCC Championship: #12 Gonzaga -6.5 vs Santa Clara | Burn 820 B_FB01_YES | Reward |
| `8a85f14748c8c164783ec0b3e9a031d5acb3c9a87ebfeefcd0a55ca99cd4c7aa` | NBA: Timberwolves (-1.5) vs. Lakers \| MAR10 | Burn 479 B_4BE9_NO | Reward |
| `26d8146483ea389d90abd3c1b3484a9603582dc630a02cb476aa6bfb5600e80a` | CLARITY ACT clears US Senate by Q1 2026! | Burn 1500 B_C1EA_YES | Reward |

**Pattern:** 2-3 redeemers (SPEND on position + prediction, negative MINT on shares). Metadata = "Bodega Market - Process Reward Positions".

### 3. Create Project

Mint PROJECT_INFO_NFT + PROJECT_PREDICTION_NFT. Creates UTxOs at project_info and project_prediction scripts. Requires BODEGA pledge.

| Tx Hash | Instance | Notes |
|---------|----------|-------|
| `316ed9a33365ec8df38918eb13ac470e65abf88e86d4c1ed950208d07363f7bb` | V3 | Mint 1 PROJECT_INFO_NFT + 1 PROJECT_PREDICTION_NFT. Policy `08a8c0fbe858...`. Output: 50,000 BODEGA pledge + 1,002 ADA |

**Pattern:** 1 MINT redeemer (project_authtoken_mp). Sends BODEGA pledge to project_info script, ADA to prediction script, open_fee to treasury.

### 4. Update Project Oracle Info

Admin updates the oracle with the market result. Mints an oracle token with the winning candidate.

| Tx Hash | Market | Oracle Token |
|---------|--------|-------------|
| `469af3055261a6ee6aab21e36cf6a7bde67ef00521614cf88f07fc0cfa3b3919` | NBA: Nuggets (-5.5) vs. Rockets \| MAR11 | Mint 1 `3E36_NBA_NUGGETS_55__ORACLE` |

**Pattern:** 3 redeemers (SPEND on project_info, SPEND on protocol_manager, MINT oracle token). Metadata = "Bodega Market - Update Project Oracle Info".

### 5. Close Market

Closes a resolved market. Transfers PROJECT_INFO_NFT and PROJECT_PREDICTION_NFT out of the script to regular wallets.

| Tx Hash | Market | Version | Date |
|---------|--------|---------|------|
| `fbc4286e5320f0c4eace318933c62ec407607f0c3ded9d1b2b7c7c790e540adf` | How many Charizards will Logan Paul pull on 02/15? | V2 | 2025-12-24 |

**Note:** Only found an example in V2. The NFTs go to regular wallets (not migration between versions). No Close found in V3 — probably because the markets are still active (~58k ADA locked).

**Pattern:** 3 SPEND redeemers (positions, protocol_manager, project_info). Consumes PROTOCOL_MANAGER_NFT as reference (re-created in output). Metadata = "Bodega Market - Close Market".

### 6. Admin Fees Withdrawal

Admin withdraws accumulated fees from the prediction UTxO.

| Tx Hash | Market | Amount | Version | Date |
|---------|--------|--------|---------|------|
| `84be78e8aa53c747c5e24aac3a63903693673b7167cc4af11ebea0c7deb17015` | NFL MVP | 4.1 ADA | V2 | ~2026-01 |

**Note:** Only found an example in V2. It's a normal fee withdrawal (outputs return to V2 addresses). No WithdrawFee found in V3 — possibly the fees have not been withdrawn yet.

**Pattern:** 1 SPEND redeemer (prediction). Metadata = "Bodega Market - Admin Fees Withdrawal". Fee sent to designated address.

### Operations not found on mainnet

- **Refund Positions** — No on-chain example found. Requires a market to be cancelled before resolution, which has not occurred in recent markets.
- **User Buy Position (without batch)** — All observed buys go through the batcher. The user creates a position UTxO that the batcher processes in the next batch.
- **Close Market (V3)** and **Admin Fees Withdrawal (V3)** — Not found. V3 markets appear to still be active. The V2 examples serve as reference since the transaction patterns are similar.

---

## Recommended tx3 Implementation Order

1. **Buy Position** - User creates a position UTxO (simplest user-facing tx)
2. **Reward Position** - User claims rewards after market resolution
3. **Refund Position** - User gets refund if market is cancelled
4. **Create Project** - Market creation (more complex, requires protocol settings reference)
5. **Batch Apply** - Batcher processes positions (operator-only, most complex)
