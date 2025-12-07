# Pact4 Directory - StoaChain Modifications

> **📖 Main Documentation**: See the [main README](../../../README.md) for an overview of StoaChain.

## Overview

StoaChain uses **Pact 5 exclusively** from genesis. This directory contains only the **shared type definitions** that are still required for Pact 5 to function properly.

This document provides a comprehensive record of all modifications made to remove Pact 4 execution code while retaining necessary shared types.

---

## Summary of Changes

### Files Deleted (Pact 4 Execution Code)

| File | Lines | Purpose |
|------|-------|---------|
| `src/Chainweb/Pact4/Backend/ChainwebPactDb.hs` | ~500 | Pact 4 database backend implementation |
| `src/Chainweb/Pact4/ModuleCache.hs` | ~75 | Pact 4 module caching system |
| `src/Chainweb/Pact4/NoCoinbase.hs` | ~25 | Pact 4 no-coinbase handling |
| `src/Chainweb/Pact4/SPV.hs` | ~590 | Pact 4 SPV (Simple Payment Verification) |
| `src/Chainweb/Pact4/Templates.hs` | ~208 | Pact 4 transaction templates |
| `src/Chainweb/Pact4/TransactionExec.hs` | ~1,548 | Pact 4 transaction execution engine |
| `src/Chainweb/Pact/PactService/Pact4/ExecBlock.hs` | ~400 | Pact 4 block execution |
| **Total** | **~3,346** | |

### Files Kept (Shared Types)

| File | Purpose |
|------|---------|
| `Transaction.hs` | Transaction parsing types (`UnparsedTransaction`, `Command`, etc.) |
| `Types.hs` | Core types (`TxContext`, `GasSupply`, `PactBlockM`, gas models) |
| `Validations.hs` | Validation utilities (`defaultMaxTTL`, gas limits, TTL checks) |
| `README.md` | This documentation file |

---

## Detailed Modifications by File

### 1. `src/Chainweb/Pact/Types.hs`

**Change:** Simplified `RunnableBlock` type to Pact5-only

```haskell
-- BEFORE (dual Pact4/Pact5 support)
data RunnableBlock logger a
  = Pact4RunnableBlock (PactDbFor logger Pact4 -> Maybe ParentHeader -> IO (a, BlockHeader))
  | Pact5RunnableBlock (PactDbFor logger Pact5 -> Maybe ParentHeader -> BlockHandle Pact5 -> IO ((a, BlockHeader), BlockHandle Pact5))

-- AFTER (Pact5 only)
newtype RunnableBlock logger a = RunnableBlock
  { runBlock :: PactDbFor logger Pact5 -> Maybe ParentHeader -> BlockHandle Pact5 -> IO ((a, BlockHeader), BlockHandle Pact5)
  }
```

### 2. `src/Chainweb/Pact/PactService.hs`

**Changes:**
- Removed import: `Chainweb.Pact.PactService.Pact4.ExecBlock`
- Removed import: `Chainweb.Pact4.Types` (qualified as Pact4)
- Added import: `Data.Functor.Const`
- Changed all `SomeBlockM $ Pair (Pact4 action) (Pact5 action)` to `SomeBlockM $ Pair (Const ()) (Pact5 action)`
- Removed `Pact4._cpLookupProcessedTx` calls

**Affected functions:**
- `execValidateBlock` - Fork block execution
- `execLookupPactTxs` - Transaction lookup

### 3. `src/Chainweb/Pact/PactService/Checkpointer.hs`

**Changes:**
- Removed import: `Chainweb.Pact.PactService.Pact4.ExecBlock`
- Removed import: `Chainweb.Pact4.Types`
- Added import: `Data.Functor.Const`
- Removed import: `Chainweb.Version.Guards (pact5)` - no longer needed

**Type changes:**
```haskell
-- BEFORE
newtype SomeBlockM logger tbl a = SomeBlockM (Product (Pact4.PactBlockM logger tbl) (Pact5.PactBlockM logger tbl) a)

-- AFTER
newtype SomeBlockM logger tbl a = SomeBlockM (Product (Const ()) (Pact5.PactBlockM logger tbl) a)
```

**Function changes:**
- `readFrom` - Simplified to only execute Pact5 path
- `restoreAndSave` - Removed height-based Pact4/Pact5 branching

### 4. `src/Chainweb/Pact/PactService/Checkpointer/Internal.hs`

**Changes:**
- Removed import: `Chainweb.Pact4.Backend.ChainwebPactDb`

**Function changes:**
- `readFrom` - Pact4T case now returns `internalError "Pact4 not supported on StoaChain"`
- `restoreAndSave/extend` - Removed entire Pact4RunnableBlock case (~60 lines)

### 5. `src/Chainweb/Pact4/Types.hs`

**Changes:**
- Removed import: `Chainweb.Pact4.ModuleCache`
- Added type alias: `type ModuleCache = DbCache PersistModuleData`
- Added stub function: `cleanModuleCache :: ChainwebVersion -> ChainId -> BlockHeight -> Bool`

### 6. `src/Chainweb/Pact/Backend/Compaction.hs`

**Changes:**
- Removed import: `Chainweb.Pact4.Backend.ChainwebPactDb ()`

### 7. `chainweb.cabal`

**Removed module references:**
```
-- Library modules removed:
Chainweb.Pact4.Backend.ChainwebPactDb
Chainweb.Pact4.ModuleCache
Chainweb.Pact4.NoCoinbase
Chainweb.Pact4.SPV
Chainweb.Pact4.Templates
Chainweb.Pact4.TransactionExec
Chainweb.Pact.PactService.Pact4.ExecBlock

-- Test modules commented out (22 modules):
Chainweb.Test.Pact4.* (all test modules)
```

**Other changes:**
- Added `GHC == 9.6` to `tested-with`

### 8. `node/chainweb-node.cabal`

**Changes:**
- Changed `Cabal >= 3.14` to `Cabal >= 3.8` in custom-setup

---

## Test File Modifications

### `test/unit/ChainwebTests.hs`

**Commented out imports (16 modules):**
```haskell
-- import qualified Chainweb.Test.Pact4.Checkpointer (tests)
-- import qualified Chainweb.Test.Pact4.DbCacheTest (tests)
-- import qualified Chainweb.Test.Pact4.GrandHash (tests)
-- import qualified Chainweb.Test.Pact4.ModuleCacheOnRestart (tests)
-- import qualified Chainweb.Test.Pact4.NoCoinbase (tests)
-- import qualified Chainweb.Test.Pact4.PactExec (tests)
-- import qualified Chainweb.Test.Pact4.PactMultiChainTest (tests)
-- import qualified Chainweb.Test.Pact4.PactReplay (tests)
-- import qualified Chainweb.Test.Pact4.PactSingleChainTest (tests)
-- import qualified Chainweb.Test.Pact4.RemotePactTest (tests)
-- import qualified Chainweb.Test.Pact4.RewardsTest (tests)
-- import qualified Chainweb.Test.Pact4.SPV (tests)
-- import qualified Chainweb.Test.Pact4.SQLite (tests)
-- import qualified Chainweb.Test.Pact4.TTL (tests)
-- import qualified Chainweb.Test.Pact4.TransactionTests (tests)
-- import qualified Chainweb.Test.Pact4.VerifierPluginTest (tests)
```

**Test suite changes:**
- `pactTestSuite` - Emptied (all Pact4 tests removed)
- `nodeTestSuite` - Changed from Pact4 to Pact5 RemotePactTest
- `suite` - Removed Pact4.SQLite, Pact4.TransactionTests, Pact4.SPV

### Pact5 Test Files Modified

| File | Change |
|------|--------|
| `test/unit/Chainweb/Test/Pact5/PactServiceTest.hs` | Removed `Pact4.ExecBlock` import |
| `test/unit/Chainweb/Test/Pact5/CutFixture.hs` | Removed `Pact4.ExecBlock` import |
| `test/unit/Chainweb/Test/Pact5/SPVTest.hs` | Removed `Pact4.ExecBlock`, `Pact4.TransactionExec` imports |
| `test/lib/Chainweb/Test/Pact5/Utils.hs` | Removed `Pact4.ExecBlock` import |

---

## Benchmark Modifications

### `bench/Chainweb/Pact/Backend/Bench.hs`

**Changes:**
- Removed import: `Chainweb.Pact4.Backend.ChainwebPactDb`
- Removed: `cpRestoreAndSave` function
- Removed: `testVer`, `testChainId`, `childOf` definitions
- Removed: `cpWithBench` function
- Disabled benchmarks: `cpBenchNoRewindOverBlock`, `cpBenchOverBlock`, `cpBenchSampleKeys`, `cpBenchLookupProcessedTx`, `_cpBenchKeys`

### `bench/Chainweb/Pact/Backend/ApplyCmd.hs`

**Changes:**
- Removed imports: `Pact4.Backend.ChainwebPactDb`, `Pact4.TransactionExec`
- Added import: `Data.Functor.Const`
- Removed: `applyCmd4` function (~35 lines)
- Removed: `pact4Version` definition
- Changed benchmark: Now only runs Pact5 benchmark

### `bench/Chainweb/Pact/Backend/PactService.hs`

**Changes:**
- Removed import: `Chainweb.Pact.PactService.Pact4.ExecBlock`

---

## Architecture After Changes

```
┌─────────────────────────────────────────────────────────────┐
│                     StoaChain Node                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Pact 5 Execution Layer                   │  │
│  │  src/Chainweb/Pact5/                                  │  │
│  │  ├── TransactionExec.hs    (transaction execution)    │  │
│  │  ├── Templates.hs          (tx templates)             │  │
│  │  ├── SPV.hs                (SPV proofs)               │  │
│  │  ├── Types.hs              (Pact5 types)              │  │
│  │  ├── NoCoinbase.hs         (no-coinbase handling)     │  │
│  │  └── Backend/ChainwebPactDb.hs (database)             │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                │
│                            ▼                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           Shared Types (src/Chainweb/Pact4/)          │  │
│  │  ├── Transaction.hs   (parsing, Command types)        │  │
│  │  ├── Types.hs         (TxContext, GasSupply, etc.)    │  │
│  │  └── Validations.hs   (TTL, gas validation)           │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                │
│                            ▼                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Core Pact Service                        │  │
│  │  src/Chainweb/Pact/                                   │  │
│  │  ├── PactService.hs        (main service)             │  │
│  │  ├── Types.hs              (RunnableBlock, etc.)      │  │
│  │  └── PactService/Checkpointer/ (state management)     │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Type System Notes

The `PactVersionT` GADT still exists with both `Pact4T` and `Pact5T` constructors:

```haskell
data PactVersion = Pact4 | Pact5
data PactVersionT (v :: PactVersion) where
    Pact4T :: PactVersionT Pact4
    Pact5T :: PactVersionT Pact5
```

This is intentional - the type system needs these definitions for:
1. Type-safe version discrimination
2. Existing code that pattern matches on versions
3. `ForSomePactVersion` existential wrapper

**Runtime behavior:** Any code path that receives `Pact4T` will now error with:
```
"Pact4 not supported on StoaChain"
```

---

## Build Requirements

To build StoaChain, you need these system packages:

```bash
# Ubuntu/Debian
sudo apt-get install -y \
  libsnappy-dev \
  libgflags-dev \
  zlib1g-dev \
  libbz2-dev \
  liblz4-dev \
  libzstd-dev \
  librocksdb-dev

# Then build
cabal build lib:chainweb
```

---

## Migration Checklist

- [x] Delete Pact4 execution modules (7 files)
- [x] Update `RunnableBlock` type to Pact5-only
- [x] Update `SomeBlockM` type to use `Const ()` for Pact4 slot
- [x] Remove Pact4 code paths in `Checkpointer/Internal.hs`
- [x] Remove Pact4 imports from `PactService.hs`
- [x] Update `Pact4/Types.hs` to not depend on deleted modules
- [x] Update `chainweb.cabal` to remove deleted modules
- [x] Disable Pact4 tests in `ChainwebTests.hs`
- [x] Comment out Pact4 test modules in `chainweb.cabal`
- [x] Remove Pact4 imports from Pact5 test files
- [x] Update benchmarks to Pact5-only
- [x] Fix Cabal version constraint
- [x] Create documentation

---

# CSV-Based Rewards & Allocations Removal

> **Date**: December 7, 2025

StoaChain uses **dynamic emission** calculated in Pact via `global-supply-register`, not a static CSV reward table or allocation distribution system. This section documents the complete removal of these legacy Kadena systems.

## Why Remove?

| System | Kadena (Before) | StoaChain (After) |
|--------|-----------------|-------------------|
| **Miner Rewards** | Static 120-year CSV schedule | Dynamic: `(CEILING - supply) / (BPD × EMISSION × chains)` |
| **Allocations** | Pre-genesis token distributions | No allocations - all STOA minted at genesis to foundation |
| **Reward Source** | `rewards/miner_rewards.csv` embedded at compile time | `global-supply-register` injected by node at runtime |

## Deleted Files

### Folders Removed
```
allocations/                    # Kadena allocation CSVs
├── Mainnet-Keysets.csv        # ~500 entries
├── Testnet-Keysets.csv        # ~200 entries  
└── token_payments.csv         # ~1320 lines of payment schedule

rewards/                        # Static reward table
└── miner_rewards.csv          # 1438 lines (120-year schedule)

rosetta/                        # Empty folder (API removed)
└── README.md

changes/                        # Empty changelog folder
└── .gitignore
```

### Source Files Removed
```
src/Chainweb/MinerReward.hs    # 280 lines - CSV parsing & reward lookup
```

### Test Files Removed
```
test/pact/coin-and-devaccts.repl           # Kadena coin tests
test/pact/twenty-chain-upgrades.repl       # Allocation tests
test/pact/twenty-chain-insufficient-bal.repl
test/unit/Chainweb/Test/MinerReward.hs     # Reward unit tests
```

### Genesis Files Removed
```
pact/coin-contract/initial-mint.yaml       # Replaced by stoa-initialise.yaml
pact/coin-contract/initial-mint-urstoa.yaml
```

## Code Modifications

### 1. `src/Chainweb/Pact5/Templates.hs`

**Critical Bug Fix**: The template was passing 3 arguments but `XM_StoaCoinbase` only takes 2!

```haskell
-- BEFORE (Broken - passing unused reward)
coinbaseTemplate mid =
  let varApp = qn "XM_StoaCoinbase" "STOA"
      rks = app (bn "read-keyset") [strLit "miner-keyset"]
      rds = app (bn "read-decimal") [strLit "reward"]  -- ❌ Not needed!
  in app varApp [midTerm, rks, rds]  -- 3 args but Pact takes 2

mkCoinbaseTerm :: MinerId -> MinerKeys -> Decimal -> (Expr (), PactValue)

-- AFTER (Fixed - matches Pact signature)
coinbaseTemplate mid =
  let varApp = qn "XM_StoaCoinbase" "STOA"
      rks = app (bn "read-keyset") [strLit "miner-keyset"]
  in app varApp [midTerm, rks]  -- ✅ 2 args: account, guard

mkCoinbaseTerm :: MinerId -> MinerKeys -> (Expr (), PactValue)  -- No Decimal
```

### 2. `src/Chainweb/Pact/PactService/Pact5/ExecBlock.hs`

Removed the `minerReward` function that looked up rewards from CSV:

```haskell
-- REMOVED
minerReward :: ChainwebVersion -> BlockHeight -> Decimal
minerReward v = _kda . minerRewardKda . blockMinerReward v

-- runCoinbase no longer computes or passes reward
runCoinbase miner = do
    ...
    -- let reward = minerReward v bh  -- ❌ REMOVED
    pactTransaction Nothing $ \db ->
        applyCoinbase logger db txCtx  -- No reward parameter
```

### 3. `src/Chainweb/Pact5/TransactionExec.hs`

```haskell
-- BEFORE
applyCoinbase :: logger -> PactDb -> Decimal -> TxContext -> IO (...)
applyCoinbase logger db reward txCtx = do
    let (coinbaseTerm, coinbaseData) = mkCoinbaseTerm mid mks reward

-- AFTER  
applyCoinbase :: logger -> PactDb -> TxContext -> IO (...)
applyCoinbase logger db txCtx = do
    let (coinbaseTerm, coinbaseData) = mkCoinbaseTerm mid mks  -- No reward
```

### 4. `src/Chainweb/Miner/Pact.hs`

Removed entire rewards system:

```haskell
-- REMOVED exports
, MinerRewards(..)
, readRewards  
, rawMinerRewards

-- REMOVED types
newtype MinerRewards = MinerRewards
    { _minerRewards :: Map BlockHeight Decimal }

-- REMOVED functions  
readRewards :: MinerRewards
rawMinerRewards :: ByteString
rawMinerRewards = $(embedFile "rewards/miner_rewards.csv")
```

### 5. `src/Chainweb/Pact/Types.hs`

Removed `_psMinerRewards` field from `PactServiceEnv`:

```haskell
-- REMOVED from exports
, psMinerRewards

-- REMOVED from type
data PactServiceEnv logger tbl = PactServiceEnv
    { ...
    , _psMinerRewards :: !MinerRewards  -- ❌ REMOVED
    , ...
    }
```

### 6. `src/Chainweb/Pact/PactService.hs`

```haskell
-- REMOVED
let !rs = readRewards
let !pse = PactServiceEnv
    { ...
    , _psMinerRewards = rs  -- ❌ REMOVED
    }
```

### 7. `chainweb.cabal`

```cabal
-- REMOVED from extra-source-files
rewards/miner_rewards.csv
allocations/token_payments.csv
allocations/Mainnet-Keysets.csv
allocations/Testnet-Keysets.csv

-- REMOVED from library modules
Chainweb.MinerReward

-- REMOVED from test modules
Chainweb.Test.MinerReward
```

### 8. Test Files

**`test/unit/Chainweb/Test/Pact5/RemotePactTest.hs`**:
- Removed `allocationTest` function (~125 lines)
- Removed `allocation01KeyPair`, `allocation02KeyPair`, `allocation02KeyPair'`
- Removed test case from test suite

**`test/lib/Chainweb/Test/Pact5/CmdBuilder.hs`**:
- Removed `allocation00KeyPair`

## How StoaChain Emission Works Now

```
┌─────────────────────────────────────────────────────────────┐
│                    COINBASE EXECUTION                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Node computes global-supply-register:                    │
│     SUM(LocalSupply) from all 10 chains                     │
│                                                              │
│  2. Node injects into chain-data before coinbase            │
│                                                              │
│  3. Pact XM_StoaCoinbase calculates reward:                 │
│     remaining = CEILING - global_supply                      │
│     yang = remaining / (BPD × EMISSION_SPEED × chains)      │
│                                                              │
│  4. Split: 90% to miner, 10% to URSTOA-Vault                │
│                                                              │
└─────────────────────────────────────────────────────────────┘

Benefits:
✅ Self-adjusting emission (approaches CEILING asymptotically)
✅ No hardcoded 120-year schedule
✅ Transparent - calculation visible in Pact code
✅ Simpler codebase - no CSV parsing/embedding
```

## Statistics

| Metric | Value |
|--------|-------|
| Files Changed | 22 |
| Lines Deleted | 3,681 |
| Lines Added | 19 |
| Modules Removed | 1 (MinerReward.hs) |
| Test Files Removed | 4 |
| Data Files Removed | 5 |

---

## Date of Modification

December 2, 2025 (Pact4 Removal)
December 7, 2025 (Rewards & Allocations Removal)

## Modified By

AI Assistant (Claude) during StoaChain implementation

## Related Documentation

- StoaChain Implementation Plan: `cursor-plan://StoaChain Implementation Plan.plan.md`
- StoaChain GitBook: https://demiourgos-holdings-tm.gitbook.io/kadena-evolution
- Emission System: `docs/EMISSION_SYSTEM.md`
- Genesis System: `cwtools/ea/README.md`
