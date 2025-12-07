# StoaChain Emission System

This document describes the deterministic emission system used by StoaChain to mint new STOA tokens as block rewards, including the 90/10 Yang emission split and URSTOA distribution mechanism.

---

## Overview

Unlike traditional blockchain emission systems that use hardcoded CSV tables or simple halving schedules, StoaChain uses a **deterministic formula** that calculates block rewards based on the current total supply across all chains.

### Key Features

- **Supply-Aware**: Emission rate decreases as total supply approaches the ceiling
- **Cross-Chain Aggregation**: Global supply computed by summing all chains' local supplies
- **Deterministic**: Same inputs always produce same outputs, ensuring consensus
- **Self-Adjusting**: No need for hard forks to adjust emission schedule
- **90/10 Split**: 90% to miners, 10% to URSTOA (Stoa Shares) holders
- **Per-Chain Division**: Block emission divided by number of chains (10 for Mainnet)

---

## Emission Formula

The emission calculation happens in `STOA.XM_StoaCoinbase`:

```
daily_emission = (CEILING - global_supply) / EMISSION_SPEED
block_emission = daily_emission / BPD / CHAINS
```

### Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `GENESIS-CEILING` | 480,000,000 | Initial ceiling (increases 1M/year) |
| `EMISSION-SPEED` | 25,000 | Days to reach ceiling at current rate |
| `BPD` | 2,880 | Blocks per day (30-second blocks) |
| `VALID_CHAIN_IDS` | 10 (Mainnet) | Number of parallel chains |

### Per-Chain Division

Since StoaChain runs multiple parallel chains, the block emission is **divided by the number of chains**:

```pact
(divisor:decimal (fold (*) 1.0 [BPD EMISSION-SPEED chains]))
(yang-amount:decimal (floor (/ remaining divisor) STOA_PREC))
```

For Mainnet with 10 chains:
- If total daily emission would be 18,720 STOA
- Each chain contributes: 18,720 / 10 = 1,872 STOA/day
- Per block per chain: 1,872 / 2,880 = 0.65 STOA

### Dynamic Ceiling

The ceiling increases by 1 million STOA per year:

```pact
(defun UC_CurrentCeiling ()
    (let
        (
            (genesis-year:decimal (UC_GetYear GENESIS-TIME))
            (year-now:decimal (UC_GetYear (at "block-time" (chain-data))))
            (new-millions:decimal (* 1000000.0 (- year-now genesis-year)))
        )
        (+ new-millions GENESIS-CEILING)
    )
)
```

---

## 90/10 Yang Emission Split

Block rewards (Yang emission) are split between miners and the Stoa Foundation:

| Recipient | Share | Description |
|-----------|-------|-------------|
| Miner | 90% | Direct block reward to the mining account |
| Foundation | 10% | Distributed to URSTOA (Stoa Shares) holders |

### Split Calculation

```pact
(defun UC_FoundationSplit:object{FoundationSplit} (yang:decimal)
    @doc "Computes the Foundation Split for a given Yang Amount"
    (let*
        (
            (foundation:decimal (floor (* yang FOUNDER-CUT) STOA_PREC))
            (miner:decimal (floor (- yang foundation) STOA_PREC))
        )
        {"miner" : miner, "foundation" : foundation}
    )
)
```

Where `FOUNDER-CUT = 0.1` (10%).

---

## URSTOA (Stoa Shares) Token

URSTOA is a secondary token on StoaChain that represents ownership shares in the foundation's 10% of Yang emissions.

### Token Properties

| Property | Value |
|----------|-------|
| Total Supply | 1,000,000 URSTOA |
| Precision | 3 decimals |
| Chain | **Chain 0 only** |
| Purpose | Distributing 10% of Yang emission |

### Why Chain 0 Only?

URSTOA exists exclusively on Chain 0 because:
1. The distribution logic (RPS - Rewards Per Share) runs on Chain 0
2. It avoids fragmenting staking across multiple chains
3. Simplifies the cross-chain accumulation of foundation shares

All URSTOA functions enforce this with `UEV_MainChain`:
```pact
(defun UEV_MainChain ()
    @doc "Enforces that the function is called on Chain 0"
    (let
        (
            (chain-id:string (at "chain-id" (chain-data)))
        )
        (enforce (= chain-id "0") "UR-STOA Operations are restricted to ChainId 0")
    )
)
```

---

## URSTOA-Vault (Staking System)

The URSTOA-Vault is where URSTOA holders stake their tokens to receive a share of the 10% foundation allocation from Yang emissions. It uses a **Rewards Per Share (RPS)** model for efficient distribution.

### Critical Design: Bootstrap Stake

The vault is initialized with **1 URSTOA already staked** at genesis. This is a critical design decision:

```
┌─────────────────────────────────────────────────────────────────┐
│                    VAULT INITIALIZATION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  At Genesis:                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  URSTOA-Vault                                            │   │
│  │  ├── total-supply: 1.0 URSTOA (bootstrap stake)         │   │
│  │  ├── rps: 0.0                                            │   │
│  │  └── foundation stake: 1 URSTOA                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Block 1 Coinbase:                                               │
│  ├── Yang emission calculated ✅                                │
│  ├── 10% to foundation (0.065 STOA)                             │
│  ├── C_URV|Inject called → RPS increases ✅                     │
│  └── Vault has supply > 0, injection works! ✅                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Why the Bootstrap Stake?

**Problem**: If the vault had 0 URSTOA staked, the injection formula would break:

```pact
;; RPS update formula
(new-rps:decimal (+ rps (/ amount total-supply)))
;;                         ↑
;;        Division by zero if total-supply = 0!
```

**Solution**: Initialize the vault with 1 URSTOA from the foundation account:

```pact
(defun XM_InitialiseUrStoaVault (account:string)
    @doc "Initializes the URSTOA-Vault with foundation as first staker"
    (require-capability (GENESIS))
    (UEV_MainChain)
    (insert URV|UrStoaVault URSTOA-VAULT
        {"rps"          : 0.0
        ,"total-supply" : 1.0    ;; Bootstrap with 1 URSTOA
        }
    )
    (insert URV|UrStoaVaultUser account
        {"balance"  : 1.0        ;; Foundation stakes 1 URSTOA
        ,"rps-debt" : 0.0
        }
    )
)
```

### Unstake Constraint: Last Token Cannot Be Withdrawn

To maintain vault viability forever, the unstake function enforces that **at least 1 URSTOA must remain staked**:

```pact
(defun C_URV|Unstake (account:string amount:decimal)
    @doc "Unstakes URSTOA from the vault"
    ...
    ;; Critical check: ensure vault doesn't go to zero
    (let
        (
            (new-vault-supply (- total-supply amount))
        )
        (enforce (>= new-vault-supply 1.0) 
            "Cannot unstake - vault must maintain minimum 1 URSTOA supply")
        ...
    )
)
```

**Examples**:

| Vault Supply | Unstake Request | Result |
|--------------|-----------------|--------|
| 5.0 URSTOA | 4.0 URSTOA | ✅ Allowed (leaves 1.0) |
| 5.0 URSTOA | 5.0 URSTOA | ❌ Rejected (would leave 0) |
| 1.0 URSTOA | 0.5 URSTOA | ❌ Rejected (would leave 0.5 < 1.0) |
| 1.0 URSTOA | Any amount | ❌ Rejected (cannot go below 1.0) |

### Why This Matters

If the vault ever reached 0 URSTOA supply:
1. **Coinbase would break**: `C_URV|Inject` is called every block
2. **Division by zero**: RPS calculation would fail
3. **Chain halts**: No blocks could be mined

By ensuring minimum 1 URSTOA, the vault is **permanently viable** from genesis onwards.

---

## Foundation Pending Amount (FPA) System

Since Yang emission happens on random chains (whichever chain mines a block), we need a mechanism to collect the 10% foundation share from all chains and distribute it to URSTOA holders on Chain 0.

### The Challenge

```
Chain 0: Mines block → 10% should go to URSTOA-Vault
Chain 3: Mines block → 10% should go to URSTOA-Vault (but vault is on Chain 0!)
Chain 7: Mines block → 10% should go to URSTOA-Vault (but vault is on Chain 0!)
```

### The Solution: Delta-Based Tracking

Instead of complex cross-chain transfers, we use a **delta-based tracking** system at the protocol level:

#### Chains 1-9 Behavior:
1. When a block is mined, calculate 10% foundation share
2. **Don't mint** the foundation share
3. Increment local `FoundationPending` table (accumulation only)
4. Miner receives only 90% of the block reward

```pact
;; CHAINS 1-9: Just accumulate, don't mint foundation share
(do
    ;;Cumulate Foundation Share
    (with-capability (UPDATE-FPA)
        (X_IncrementFPASupply foundation-share))
    ;;Update Local Supply (only miner's 90%)
    (with-capability (UPDATE-LOCAL-SUPPLY)
        (X_UpdateLocalSupply miner-share true))
    ;;Miner only gets 90% share
    (with-capability (CREDIT account)
        (X_Credit account account-guard miner-share))
)
```

#### Chain 0 Behavior:
1. When a block is mined, read `external-fpa` from `chain-data`
   - This is the **sum** of `FoundationPending` from chains 1-9
2. Calculate `delta = external-fpa - LastProcessedAmount`
3. Mint: `(10% of this block) + delta` to URSTOA-Vault
4. Update `LastFPA` to current `external-fpa`
5. Miner receives 90% of block reward + triggers foundation injection

```pact
;; CHAIN 0: Mint foundation share + accumulated from other chains
(let
    (
        (external-fpa:decimal (at "external-fpa" (chain-data)))
        (last-fpa:decimal (UR_LastFPA))
        (delta:decimal (- external-fpa last-fpa))
        (total-foundation:decimal (+ delta foundation-share))
        (whole:decimal (+ miner-share total-foundation))
    )
    ;;Update Local Supply
    (with-capability (UPDATE-LOCAL-SUPPLY)
        (X_UpdateLocalSupply whole true))
    ;;Update Last FPA
    (with-capability (UPDATE-FPA)
        (X_UpdateLastFPA external-fpa))
    ;;Miner gets 90% plus total-foundation (to inject)
    (with-capability (CREDIT account)
        (X_Credit account account-guard whole))
    ;;Miner Injects total-foundation to URSTOA-Vault
    (C_URV|Inject account total-foundation)
)
```

### FPA Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Foundation Pending Amount Flow                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        ┌──────────┐               │
│  │ Chain 1  │  │ Chain 2  │  │ Chain 3  │  ...   │ Chain 9  │               │
│  │          │  │          │  │          │        │          │               │
│  │ FPA: 50  │  │ FPA: 30  │  │ FPA: 45  │        │ FPA: 25  │               │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘        └────┬─────┘               │
│       │             │             │                   │                     │
│       └─────────────┼─────────────┼───────────────────┘                     │
│                     │             │                                         │
│                     ▼             ▼                                         │
│              ┌─────────────────────────┐                                    │
│              │ computeFoundationPending│                                    │
│              │ Sum = 50+30+45+...+25   │                                    │
│              │ external-fpa = 350      │                                    │
│              └────────────┬────────────┘                                    │
│                           │                                                 │
│                           ▼                                                 │
│              ┌─────────────────────────┐                                    │
│              │       Chain 0           │                                    │
│              │                         │                                    │
│              │  last-fpa: 300          │                                    │
│              │  external-fpa: 350      │                                    │
│              │  delta: 350 - 300 = 50  │                                    │
│              │                         │                                    │
│              │  Mint to URSTOA-Vault:  │                                    │
│              │  10% of block + 50 = 56 │                                    │
│              │                         │                                    │
│              │  Update last-fpa: 350   │                                    │
│              └─────────────────────────┘                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why This Works

1. **No Cross-Chain Writes**: Chains 1-9 never write to Chain 0
2. **No Stranded Funds**: Foundation shares accumulate until Chain 0 processes them
3. **Protocol-Level**: No external robots or services required
4. **Deterministic**: Same inputs always produce same outputs

### Chain-Data Extension

Two new fields were added to `chain-data`:

| Field | Type | Description |
|-------|------|-------------|
| `global-supply-register` | Decimal | Sum of LocalSupply from all chains (emission calculation) |
| `external-fpa` | Decimal | Sum of FoundationPending from chains 1-9 (Chain 0 only) |

These are computed in Haskell before coinbase execution and injected into the Pact environment.

---

## Global Supply Registry

### The Challenge

StoaChain runs 10 parallel chains. Each chain maintains its own `LocalSupply` table tracking STOA minted on that chain. For the emission formula, we need the **total supply across all chains**.

### The Solution

We extended Pact's `chain-data` native function to include a `global-supply-register` field that contains the sum of all chains' local supplies.

### Implementation Components

#### 1. AncientPact (Pact-5 Fork)

**File**: `Pact/Core/Evaluate.hs`

Added new field to `PublicData`:
```haskell
data PublicData = PublicData
    { _pdPublicMeta :: PublicMeta
    , _pdBlockHeight :: Word64
    , _pdBlockTime :: Int64
    , _pdPrevBlockHash :: Text
    , _pdGlobalSupplyRegister :: Maybe Decimal  -- NEW
    }
```

Added field constant:
```haskell
cdGlobalSupplyRegister :: Field
cdGlobalSupplyRegister = Field "global-supply-register"
```

Updated `coreChainData` to return the new field.

#### 2. AncientStoa (Chainweb-node)

**File**: `src/Chainweb/Pact/GlobalSupply.hs`

```haskell
-- | Compute total STOA supply across all chains
computeGlobalSupply :: Logger logger 
    => logger -> ChainwebVersion -> FilePath -> IO Decimal
computeGlobalSupply logger v pactDbDir = do
    let chainIds = allChains v  -- [0..9] for mainnet
    supplies <- forM chainIds $ \cid ->
        queryLocalSupplyFromChain logger cid pactDbDir
    return $ sum supplies

-- | Query LocalSupply from a single chain's SQLite database
queryLocalSupplyFromChain :: Logger logger 
    => logger -> ChainId -> FilePath -> IO Decimal
queryLocalSupplyFromChain logger cid pactDbDir = do
    result <- tryAny $ withSqliteDb cid logger pactDbDir False $ \sqlEnv ->
        queryLocalSupplyTable sqlEnv
    case result of
        Left _err -> return 0.0  -- Table doesn't exist yet (genesis)
        Right supply -> return supply

-- | Direct SQLite query for LocalSupply
queryLocalSupplyTable :: SQLiteEnv -> IO Decimal
queryLocalSupplyTable sqlEnv = do
    result <- Pact.qry sqlEnv 
        "SELECT rowdata FROM \"STOA_LocalSupply\" WHERE rowkey = 'supply'" 
        [] [RBlob]
    case result of
        [[SBlob rowData]] -> 
            case Aeson.decodeStrict rowData of
                Just (LocalSupplyRow supply) -> return supply
                Nothing -> return 0.0
        _ -> return 0.0
```

**File**: `src/Chainweb/Pact5/Types.hs`

Extended `TxContext` to carry global supply:
```haskell
data TxContext = TxContext
    { _tcParentHeader :: !ParentHeader
    , _tcMiner :: !Miner
    , _tcGlobalSupply :: !(Maybe Decimal)  -- NEW
    }
```

**File**: `src/Chainweb/Pact5/TransactionExec.hs`

Updated `ctxToPublicData` to inject global supply:
```haskell
ctxToPublicData :: PublicMeta -> TxContext -> PublicData
ctxToPublicData pm (TxContext ph _ globalSupply) = PublicData
    { _pdPublicMeta = pm
    , _pdBlockHeight = bh
    , _pdBlockTime = bt
    , _pdPrevBlockHash = toText h
    , _pdGlobalSupplyRegister = globalSupply  -- NEW
    }
```

**File**: `src/Chainweb/Pact/PactService/Pact5/ExecBlock.hs`

Compute global supply before coinbase:
```haskell
runCoinbase miner = do
    ...
    pactDbDir <- view (psServiceEnv . psPactDbDir)
    
    -- Compute global supply BEFORE coinbase execution
    globalSupply <- liftIO $ Just <$> computeGlobalSupply logger v pactDbDir
    let txCtx = TxContext ph miner globalSupply
    
    pactTransaction Nothing $ \db ->
        applyCoinbase logger db reward txCtx
```

---

## Coinbase Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Block N Execution on Chain X                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. runCoinbase starts                                                      │
│     │                                                                       │
│     ▼                                                                       │
│  2. computeGlobalSupply() queries all 10 chains:                            │
│     ┌───────────────────────────────────────────────────────────┐           │
│     │ Chain 0: SELECT rowdata FROM STOA_LocalSupply → 1,200,000 │           │
│     │ Chain 1: SELECT rowdata FROM STOA_LocalSupply → 1,300,000 │           │
│     │ ...                                                       │           │
│     │ Chain 9: SELECT rowdata FROM STOA_LocalSupply → 1,100,000 │           │
│     │ ──────────────────────────────────────────────────────────│           │
│     │ TOTAL = 12,000,000                                        │           │
│     └───────────────────────────────────────────────────────────┘           │
│     │                                                                       │
│     ▼                                                                       │
│  3. IF Chain 0: computeFoundationPending() queries chains 1-9:              │
│     ┌───────────────────────────────────────────────────────────┐           │
│     │ Chain 1: SELECT rowdata FROM STOA_FoundationPending → 50  │           │
│     │ Chain 2: SELECT rowdata FROM STOA_FoundationPending → 30  │           │
│     │ ...                                                       │           │
│     │ Chain 9: SELECT rowdata FROM STOA_FoundationPending → 25  │           │
│     │ ──────────────────────────────────────────────────────────│           │
│     │ TOTAL FPA = 350                                           │           │
│     └───────────────────────────────────────────────────────────┘           │
│     │                                                                       │
│     ▼                                                                       │
│  4. TxContext created:                                                      │
│     { globalSupply = 12,000,000, externalFpa = 350 (Chain 0) or Nothing }   │
│     │                                                                       │
│     ▼                                                                       │
│  5. Pact's (chain-data) returns:                                            │
│     { "global-supply-register": 12000000.0,                                 │
│       "external-fpa": 350.0,          (Chain 0 only)                        │
│       "chain-id": "X",                                                      │
│       ... }                                                                 │
│     │                                                                       │
│     ▼                                                                       │
│  6. STOA.XM_StoaCoinbase executes emission calculation:                     │
│     ┌──────────────────────────────────────────────────────────┐            │
│     │ global-supply = 12,000,000                               │            │
│     │ ceiling = 480,000,000                                    │            │
│     │ remaining = 468,000,000                                  │            │
│     │ divisor = BPD × EMISSION-SPEED × CHAINS                  │            │
│     │         = 2880 × 25000 × 10 = 720,000,000                │            │
│     │ yang-amount = 468,000,000 / 720,000,000 = 0.65 STOA      │            │
│     │                                                          │            │
│     │ 90/10 Split:                                             │            │
│     │   miner-share = 0.65 × 0.9 = 0.585 STOA                  │            │
│     │   foundation-share = 0.65 × 0.1 = 0.065 STOA             │            │
│     └──────────────────────────────────────────────────────────┘            │
│     │                                                                       │
│     ├── IF Chain 1-9:                                                       │
│     │   • Miner credited: 0.585 STOA                                        │
│     │   • FoundationPending += 0.065                                        │
│     │   • LocalSupply += 0.585                                              │
│     │                                                                       │
│     └── IF Chain 0:                                                         │
│         • delta = external-fpa - last-fpa = 350 - 300 = 50                  │
│         • total-foundation = 0.065 + 50 = 50.065 STOA                       │
│         • Miner credited: 0.585 + 50.065 = 50.65 STOA                       │
│         • Miner injects 50.065 STOA to URSTOA-Vault                         │
│         • LocalSupply += 50.65                                              │
│         • LastFPA = 350                                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Supply-Modifying Functions

### STOA Supply

Only three capabilities can modify the total STOA supply:

| Capability | Function | Effect on Supply |
|------------|----------|------------------|
| `GENESIS` | `XM_InitialMint` | **Increase** - Mints initial supply at genesis |
| `COINBASE` | `XM_StoaCoinbase` | **Increase** - Mints block rewards (miner 90% + foundation on Chain 0) |
| `REMEDIATE` | `XM_Remediate` | **Decrease** - Emergency supply correction |

All three functions update the `LocalSupply` table after modifying balances:

```pact
(with-capability (UPDATE-LOCAL-SUPPLY)
    (X_UpdateLocalSupply amount true))   ; true = increase
    ; or
    (X_UpdateLocalSupply amount false))  ; false = decrease
```

### URSTOA Supply

URSTOA has a fixed supply of 1,000,000 minted at genesis:

| Capability | Function | Effect on Supply |
|------------|----------|------------------|
| `GENESIS` | `XM_UR\|InitialMint` | **Increase** - Mints 1M URSTOA at genesis (Chain 0 only) |

Note: URSTOA has no coinbase - its value comes from receiving STOA from the 10% foundation share.

---

## Tables

### LocalSupply Table (STOA)

Each chain maintains its own `LocalSupply` table:

```pact
(defschema LocalSupplySchema
    supply:decimal
)

(deftable LocalSupply:{LocalSupplySchema})

(defconst CSK "supply")  ; Constant Supply Key
```

#### Reading Local Supply (Pact)

```pact
(defun UR_LocalStoaSupply:decimal ()
    @doc "Returns the total STOA supply on this chain"
    (with-default-read LocalSupply CSK 
        {"supply" : 0.0} 
        {"supply" := s} 
        s
    )
)
```

#### Reading Local Supply (Haskell/SQLite)

```sql
SELECT rowdata FROM "STOA_LocalSupply" WHERE rowkey = 'supply'
```

Returns JSON blob: `{"supply": 1200000.0}`

### FoundationPending Table

Chains 1-9 accumulate the 10% foundation share in this table:

```pact
(defschema FoundationPendingSchema
    fpa:decimal         ;; Foundation Pending Amount (chains 1-9)
    last-fpa:decimal    ;; Last processed FPA (chain 0 only)
)

(deftable FoundationPending:{FoundationPendingSchema})

(defconst FPA "foundation-pending-amount")
```

#### Reading FPA (Haskell/SQLite)

```sql
SELECT rowdata FROM "STOA_FoundationPending" WHERE rowkey = 'foundation-pending-amount'
```

Returns JSON blob: `{"fpa": 50.0, "last-fpa": 0.0}`

### UR|LocalSupply Table (URSTOA)

Chain 0 maintains the URSTOA supply (always 1,000,000):

```pact
(deftable UR|LocalSupply:{LocalSupplySchema})
```

Note: This table exists on all chains but is only populated on Chain 0.

---

## Edge Cases

### Genesis Block

At genesis, the `LocalSupply` table doesn't exist yet. The `computeGlobalSupply` function handles this:

```haskell
queryLocalSupplyFromChain logger cid pactDbDir = do
    result <- tryAny $ withSqliteDb cid logger pactDbDir False $ \sqlEnv ->
        queryLocalSupplyTable sqlEnv
    case result of
        Left _err -> return 0.0  -- Table doesn't exist
        Right supply -> return supply
```

### Cross-Chain Transfers

Cross-chain transfers (`C_TransferAcross`) do NOT change total supply:
- Source chain: Balance decreases, LocalSupply decreases
- Target chain: Balance increases, LocalSupply increases
- Net effect: Zero change to global supply

---

## Comparison with Kadena

| Aspect | Kadena | StoaChain |
|--------|--------|-----------|
| Emission Source | CSV file (`miner_rewards.csv`) | Deterministic formula |
| Supply Tracking | None (implicit) | Explicit `LocalSupply` table per chain |
| Global Supply | Not tracked | Computed via `global-supply-register` |
| Flexibility | Hard fork required to change | Self-adjusting based on supply |
| Ceiling | None | Dynamic (increases 1M/year) |

---

## Removed Kadena Legacy Systems

StoaChain's dynamic emission system means several Kadena legacy components were completely removed from the codebase.

### CSV-Based Miner Rewards (Removed)

Kadena used a static 120-year reward schedule stored in `rewards/miner_rewards.csv`:

```csv
# Kadena's miner_rewards.csv (REMOVED)
87600,23.04523    # Block 87600: 23.04 KDA per block
175200,22.97878   # Block 175200: 22.97 KDA per block
262800,22.91249   # ...decreasing every ~3 months
...               # 1438 entries over 120 years
```

**Why removed?** StoaChain calculates rewards dynamically:

```
┌─────────────────────────────────────────────────────────────────┐
│  KADENA (Removed)                 │  STOACHAIN (Current)         │
├───────────────────────────────────┼──────────────────────────────┤
│  reward = CSV[blockHeight]        │  yang = (CEILING - supply)   │
│          / numChains              │        / (BPD × SPEED × N)   │
├───────────────────────────────────┼──────────────────────────────┤
│  ❌ Static 120-year schedule      │  ✅ Self-adjusting           │
│  ❌ Embedded at compile time      │  ✅ Computed at runtime      │
│  ❌ Hard fork to change           │  ✅ Formula-based            │
└───────────────────────────────────┴──────────────────────────────┘
```

### Allocation System (Removed)

Kadena had a pre-genesis token allocation system for investors, team, etc:

```
allocations/                    # REMOVED
├── Mainnet-Keysets.csv        # ~500 investor keysets
├── Testnet-Keysets.csv        # ~200 testnet allocations  
└── token_payments.csv         # 1320 vesting schedule entries
```

**Why removed?** StoaChain has no pre-allocation:
- All 12M STOA minted at genesis to foundation account
- No investor vesting schedules
- No `release-allocation` mechanism needed

### Removed Files Summary

| Category | Files Removed | Lines |
|----------|---------------|-------|
| **Reward CSV** | `rewards/miner_rewards.csv` | 1,438 |
| **Allocations** | `allocations/*.csv` (3 files) | ~1,320 |
| **Haskell Module** | `src/Chainweb/MinerReward.hs` | 280 |
| **Test Files** | `test/unit/Chainweb/Test/MinerReward.hs` | ~100 |
| **Pact Tests** | `test/pact/allocation*.repl` | ~50 |
| **Total** | 8+ files | **~3,200** |

### Code Changes

The removal required updates to several Haskell modules:

**`src/Chainweb/Pact5/Templates.hs`** - Fixed coinbase template:
```haskell
-- BEFORE (Kadena): Passed 3 arguments including CSV reward
mkCoinbaseTerm :: MinerId -> MinerKeys -> Decimal -> (Expr (), PactValue)
mkCoinbaseTerm mid mks reward = ...

-- AFTER (StoaChain): Only 2 arguments, reward calculated in Pact
mkCoinbaseTerm :: MinerId -> MinerKeys -> (Expr (), PactValue)
mkCoinbaseTerm mid mks = ...
```

**`src/Chainweb/Pact/PactService/Pact5/ExecBlock.hs`** - Removed reward lookup:
```haskell
-- REMOVED: Static CSV lookup
minerReward :: ChainwebVersion -> BlockHeight -> Decimal
minerReward v = _kda . minerRewardKda . blockMinerReward v

-- Coinbase now only passes miner info, not reward
applyCoinbase logger db txCtx  -- No Decimal parameter
```

**`src/Chainweb/Miner/Pact.hs`** - Removed:
```haskell
-- REMOVED types and functions
newtype MinerRewards = MinerRewards { _minerRewards :: Map BlockHeight Decimal }
readRewards :: MinerRewards
rawMinerRewards :: ByteString  -- $(embedFile "rewards/miner_rewards.csv")
```

---

## Files Reference

| File | Purpose |
|------|---------|
| `pact/coin-contract/stoa.pact` | STOA + URSTOA contracts with emission logic |
| `pact/coin-contract/stoa-initialise.yaml` | Genesis initialization (accounts, mints, vault) |
| `src/Chainweb/Pact/GlobalSupply.hs` | Global supply + FPA computation |
| `src/Chainweb/Pact5/Types.hs` | TxContext with global supply and FPA fields |
| `src/Chainweb/Pact5/TransactionExec.hs` | PublicData construction |
| `src/Chainweb/Pact/PactService/Pact5/ExecBlock.hs` | Coinbase execution with FPA logic |

---

## Repository References

- **AncientPact** (Pact-5 fork): https://github.com/StoaChain/AncientPact
  - Tag: `5.4.1-stoachain`
  - Changes: `_pdGlobalSupplyRegister` field in `PublicData`

- **AncientStoa** (Chainweb-node fork): https://github.com/StoaChain/AncientStoa
  - Branch: `master`
  - New module: `Chainweb.Pact.GlobalSupply`

