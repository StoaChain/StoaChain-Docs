# StoaChain Yin Earnings System (Dynamic Gas Price)

This document describes the **Yin Earnings** mechanism - the gas-based income that miners earn from transaction execution, powered by StoaChain's dynamic minimum gas price system.

---

## Yin vs Yang: Miner Income Sources

StoaChain miners earn STOA from two distinct sources:

| Source | Name | Description | Document |
|--------|------|-------------|----------|
| **Block Rewards** | **Yang Emission** | Newly minted STOA (90% to miner) | `EMISSION_SYSTEM.md` |
| **Transaction Fees** | **Yin Earnings** | Gas fees from transactions (100% to miner) | **This document** |

### Key Difference

```
YANG EMISSION                        YIN EARNINGS
─────────────────────────────────    ─────────────────────────────────
• Newly minted STOA                  • Transfer of existing STOA
• Decreases over time                • Increases over time (min price)
• 90% miner / 10% foundation         • 100% to miner
• Supply-based formula               • Activity-based (tx volume)
• Inflationary                       • Non-inflationary
```

Unlike Yang Emission (which mints new tokens), **Yin Earnings are not emissions** - they represent the transfer of existing STOA from transaction senders to miners as payment for block space.

---

## Overview

StoaChain uses a **time-based dynamic minimum gas price** that:
- Starts at **10,000 ANU** at genesis
- Increases by **1 ANU** every **3 hours**
- Caps at **400,000 ANU** (reached after ~133 years)

### ANU (Atomic Unit)

**ANU** is the smallest unit of STOA:
```
1 STOA = 10^12 ANU = 1,000,000,000,000 ANU
```

STOA has 12 decimal places of precision, matching Kadena's KDA.

### Gas Price Conversion

| ANU | STOA |
|-----|------|
| 10,000 | 0.00000001 (1e-8) |
| 100,000 | 0.0000001 (1e-7) |
| 400,000 | 0.0000004 (4e-7) |

---

## Formula

```
If time <= genesis_time:
    minimum_gas_price = 10,000 ANU

Else:
    seconds_elapsed = current_time - genesis_time
    intervals = floor(seconds_elapsed / 10,800)  # 10,800 = 3 hours
    minimum_gas_price = min(10,000 + intervals, 400,000)
```

### Timeline

| Time After Genesis | Minimum Gas Price (ANU) | Minimum Gas Price (STOA) |
|--------------------|-------------------------|--------------------------|
| 0 hours | 10,000 | 0.00000001 |
| 3 hours | 10,001 | 0.000000010001 |
| 24 hours | 10,008 | 0.000000010008 |
| 1 week | 10,056 | 0.000000010056 |
| 1 month | 10,240 | 0.00000001024 |
| 1 year | 12,920 | 0.00000001292 |
| 10 years | 39,200 | 0.0000000392 |
| 133.5 years | 400,000 (cap) | 0.0000004 |

---

## Configuration

### Genesis Time

The genesis time must be configured in **TWO places** and **MUST MATCH**:

| Location | File | Constant |
|----------|------|----------|
| **Pact** | `pact/coin-contract/stoa.pact` | `GENESIS-TIME` |
| **Haskell** | `src/Chainweb/GasPrice.hs` | `stoaGenesisTime` |

**Current placeholder value**: `2026-01-01T00:00:00Z`

**Important Notes**:
- Set this to a time **before** your planned chain launch
- If the blockchain starts before genesis time, gas price = 10,000 ANU (minimum)
- Update both files before chain launch!

### Pact Constants (stoa.pact)

```pact
(defconst GENESIS-TIME              (time "2026-01-01T00:00:00Z"))
(defconst GENESIS-MIN-GAS-PRICE     10000)      ; 10,000 ANU
(defconst MAX-GAS-PRICE             400000)     ; 400,000 ANU
(defconst GAS-PRICE-INTERVAL        10800.0)    ; 3 hours in seconds
```

### Haskell Constants (GasPrice.hs)

```haskell
stoaGenesisTime :: UTCTime
stoaGenesisTime = parseTimeOrError True defaultTimeLocale "%Y-%m-%dT%H:%M:%SZ" 
    "2026-01-01T00:00:00Z"

genesisMinGasPriceANU :: Integer
genesisMinGasPriceANU = 10_000

maxGasPriceANU :: Integer  
maxGasPriceANU = 400_000

gasPriceIntervalSeconds :: Integer
gasPriceIntervalSeconds = 10_800  -- 3 hours
```

---

## Implementation Details

### Enforcement Points

The minimum gas price is enforced at **two points**:

#### 1. Mempool Insertion (`InMem.hs`)

When a transaction is submitted to the mempool:

```haskell
gasPriceMinCheck :: Either InsertError ()
gasPriceMinCheck = ebool_ (InsertErrorUndersized txPrice minPrice) (txPrice >= minPrice)
  where
    -- Use TX creation time for minimum calculation
    Time (TimeSpan (Micros creationTimeMicros)) = txMetaCreationTime $ txMetadata txcfg t
    minPrice = computeMinGasPriceFromCreationTime creationTimeMicros
```

#### 2. Block Validation (`Validations.hs`)

When validating transactions in a block:

```haskell
assertMinGasPrice :: P.GasPrice -> P.TxCreationTime -> Bool
assertMinGasPrice txGasPrice (P.TxCreationTime creationTimeSecs) =
    txGasPrice >= computeMinGasPriceFromCreationTime (creationTimeSecs * 1_000_000)
```

### Why TX Creation Time?

Both enforcement points use the **transaction's creation time** (not current time) to compute the minimum. This ensures:

1. **Consistency**: Mempool and block validation always agree
2. **Lenient Policy**: A TX valid at submission stays valid until expiry
3. **No Manipulation**: TTL check prevents fake creation times

### Validation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Transaction Lifecycle                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. User creates TX                                             │
│     ├── creationTime = T                                        │
│     └── gasPrice = X                                            │
│                                                                 │
│  2. TX submitted to mempool                                     │
│     ├── TTL check: Is T recent? (within ~2 minutes)            │
│     ├── Gas price check: Is X >= computeMinGasPrice(T)?        │
│     │   ├── YES → Accept into mempool                          │
│     │   └── NO  → Reject: InsertErrorUndersized                │
│     │                                                          │
│  3. TX included in block (possibly hours later)                │
│     ├── Block validation uses same check                       │
│     ├── computeMinGasPrice(T) → same result                    │
│     └── TX still valid!                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Pact Query Functions

Users can query the current minimum gas price via Pact:

### Get Minimum in ANU

```pact
(STOA.UC_MinimumGasPriceANU)
;; Returns: 10042 (example, depends on current time)
```

### Get Minimum in STOA

```pact
(STOA.UC_MinimumGasPriceSTOA)
;; Returns: 0.000000010042 (example)
```

---

## Removed Components

The following static gas price components were **removed** from Kadena's original code:

### Configuration.hs
- Removed: `_configMinGasPrice` field
- Removed: `configMinGasPrice` lens export
- Removed: Default value `1e-8`
- Removed: JSON encoding/parsing for `minGasPrice`
- Removed: CLI option `--min-gas-price`

### InMemTypes.hs
- Removed: `_inmemTxMinGasPrice` field from `InMemConfig`

### Chainweb.hs
- Removed: `GasPrice` parameter from `validatingMempoolConfig`
- Removed: `_configMinGasPrice conf` from mempool config creation

---

## Files Modified

| File | Changes |
|------|---------|
| **NEW** `src/Chainweb/GasPrice.hs` | Dynamic gas price computation module |
| `src/Chainweb/Mempool/InMem.hs` | Uses dynamic minimum based on TX creation time |
| `src/Chainweb/Mempool/InMemTypes.hs` | Removed `_inmemTxMinGasPrice` field |
| `src/Chainweb/Pact5/Validations.hs` | Added `assertMinGasPrice` validation |
| `src/Chainweb/Chainweb.hs` | Removed static gas price from mempool config |
| `src/Chainweb/Chainweb/Configuration.hs` | Removed all static gas price config |
| `pact/coin-contract/stoa.pact` | Added gas price constants and functions |
| `chainweb.cabal` | Added `Chainweb.GasPrice` module |

---

## Troubleshooting

### Error: "Gas price below minimum for transaction creation time"

**Cause**: The transaction's gas price is below the minimum for its creation time.

**Solution**: 
1. Query the current minimum: `(STOA.UC_MinimumGasPriceSTOA)`
2. Set your transaction's gas price to at least this value
3. Add a small buffer to account for time passing

### Error: "InsertErrorUndersized"

**Cause**: Mempool rejected the transaction due to low gas price.

**Solution**: Same as above - use the current minimum gas price or higher.

### Genesis Time Mismatch

If Pact and Haskell have different genesis times, gas price calculations will differ, causing transactions to fail validation unexpectedly.

**Solution**: Ensure both constants match exactly:
- `stoa.pact`: `GENESIS-TIME`
- `GasPrice.hs`: `stoaGenesisTime`

---

## Comparison with Kadena

| Aspect | Kadena | StoaChain |
|--------|--------|-----------|
| Minimum Gas Price | Static (`1e-8`) | Dynamic (time-based) |
| Configuration | CLI/config file | Hardcoded + Pact |
| Starting Price | Fixed | 10,000 ANU |
| Price Increase | None | +1 ANU every 3 hours |
| Maximum Price | None | 400,000 ANU |
| Query Function | N/A | `UC_MinimumGasPriceSTOA` |

---

---

## Related Documentation

- **Yang Emission**: `EMISSION_SYSTEM.md` - Deterministic block rewards (90/10 split)
- **Genesis System**: `cwtools/ea/README.md` - Genesis payload generation

---

## Date

Last updated: December 2024

## Attribution

StoaChain Yin Earnings System - Dynamic gas pricing for miner income from transaction fees.

