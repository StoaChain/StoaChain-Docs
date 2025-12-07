# StoaChain Node Launch Checklist

This document provides a **comprehensive checklist** of all configurations that must be set **before** launching a StoaChain node. Failure to properly configure these items will result in a non-functional or insecure network.

---

## Quick Reference

| Priority | Item | Location | Status |
|----------|------|----------|--------|
| 🔴 Critical | Genesis Time | 2 files | ☐ |
| 🔴 Critical | Foundation Keyset | YAML | ☐ |
| 🔴 Critical | Stoa Masters Keysets | YAML | ☐ |
| 🟡 Important | Namespace Keysets | YAML | ☐ |
| 🟡 Important | Bootstrap Nodes | Config | ☐ |
| 🟢 Verify | Genesis Payloads | Generate | ☐ |
| 🟢 Verify | Build Node | Compile | ☐ |

---

## 🔴 CRITICAL: Genesis Time Configuration

The genesis time **MUST** be set identically in **TWO locations**. A mismatch will cause gas price validation failures and broken consensus.

### 1. Pact Contract: `pact/coin-contract/stoa.pact`

```pact
;; Line ~23 in stoa.pact
(defconst GENESIS-TIME (time "2026-01-01T00:00:00Z"))
```

**Also in:** `pact/coin-contract/v1/stoa.pact` (same line)

### 2. Haskell Module: `src/Chainweb/GasPrice.hs`

```haskell
-- Line ~35 in GasPrice.hs
stoaGenesisTime :: UTCTime
stoaGenesisTime = parseTimeOrError True defaultTimeLocale "%Y-%m-%dT%H:%M:%SZ" 
    "2026-01-01T00:00:00Z"  -- MUST MATCH stoa.pact!
```

### ⚠️ Important Notes

- Set genesis time to a date **BEFORE** your planned launch
- Use UTC timezone (Z suffix)
- Format: `YYYY-MM-DDTHH:MM:SSZ`
- If blockchain starts before genesis time, gas price = minimum (10,000 ANU)

### Verification

```bash
# Check Pact constant
grep "GENESIS-TIME" pact/coin-contract/stoa.pact

# Check Haskell constant  
grep "stoaGenesisTime" src/Chainweb/GasPrice.hs
```

---

## 🔴 CRITICAL: Foundation Keyset

The foundation account receives the initial 12M STOA and 1M URSTOA at genesis.

### File: `pact/coin-contract/stoa-initialise.yaml`

```yaml
data:
  stoa-foundation-keyset:
    keys:
      - YOUR_FOUNDATION_PUBLIC_KEY_HERE  # Replace!
    pred: keys-all
```

### Requirements

- Generate a secure ED25519 keypair
- Store private key securely (hardware wallet recommended)
- This key controls all genesis funds

### Key Generation

```bash
# Using Pact CLI
pact -g

# Output example:
# public: a241430816f0112add2179e3e3939ebab5336ab024be990960e684e38eb87d80
# secret: [KEEP THIS SECURE]
```

---

## 🔴 CRITICAL: Stoa Masters Keysets (Governance)

Stoa Masters control governance functions. Configure 7 master keys for multi-sig.

### File: `pact/genesis/stoachain/stoa-masters.yaml` (Mainnet)

```yaml
data:
  stoa_master_one:
    keys: ["PUBLIC_KEY_1"]
    pred: "keys-all"
  stoa_master_two:
    keys: ["PUBLIC_KEY_2"]
    pred: "keys-all"
  # ... through stoa_master_seven
```

### Also Configure For:
- `pact/genesis/stoachain/stoa-masters-testnet.yaml`
- `pact/genesis/stoachain/stoa-masters-devnet.yaml`

### Requirements

- Generate 7 unique keypairs
- Distribute to different trusted parties
- Consider hardware security modules (HSM)

---

## 🟡 IMPORTANT: Namespace Keysets

Namespace administration keys for the `free` and `user` namespaces.

### File: `pact/genesis/stoachain/ns.yaml` (Mainnet)

```yaml
data:
  ns-admin-keyset:
    keys: ["ADMIN_PUBLIC_KEY"]
    pred: "keys-all"
  ns-operate-keyset:
    keys: ["OPERATE_PUBLIC_KEY"]
    pred: "keys-all"
```

### Also Configure For:
- `pact/genesis/stoachain/ns-testnet.yaml`
- `pact/genesis/stoachain/ns-devnet.yaml`

---

## 🟡 IMPORTANT: Bootstrap Nodes

Configure initial peer nodes for network discovery.

### File: Node configuration (varies by deployment)

```yaml
# Example chainweb-node config
p2p:
  bootstrapNodes:
    - address: "node1.stoachain.io"
      port: 1789
    - address: "node2.stoachain.io"
      port: 1789
```

### For Development

Use the provided test configuration:
```
cwtools/run-nodes/test-bootstrap-node.config
```

---

## 🟢 VERIFY: Generate Genesis Payloads

After all configurations are set, generate the genesis payloads.

### Steps

```bash
# 1. Navigate to cwtools
cd chainweb-node/cwtools

# 2. Run the Ea tool
cabal run ea

# 3. Verify output files
ls ../src/Chainweb/BlockHeader/Genesis/
# Expected: StoaMainnet0to9Payload.hs, StoaTestnet0to2Payload.hs, StoaDevnet0to2Payload.hs
```

### What Gets Generated

| Network | Chains | Output File |
|---------|--------|-------------|
| Mainnet | 0-9 (10) | `StoaMainnet0to9Payload.hs` |
| Testnet | 0-2 (3) | `StoaTestnet0to2Payload.hs` |
| Devnet | 0-2 (3) | `StoaDevnet0to2Payload.hs` |

---

## 🟢 VERIFY: Build the Node

Compile the chainweb-node with all changes.

### Prerequisites

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
```

### Build Commands

```bash
cd chainweb-node

# Build the library
cabal build lib:chainweb

# Build the node executable
cabal build chainweb-node

# Run tests (optional but recommended)
cabal test chainweb-tests
```

---

## Pre-Launch Verification Checklist

### ☐ Genesis Time
- [ ] `stoa.pact` GENESIS-TIME set
- [ ] `v1/stoa.pact` GENESIS-TIME set (same value)
- [ ] `GasPrice.hs` stoaGenesisTime set (same value)
- [ ] All three values are **IDENTICAL**

### ☐ Keysets
- [ ] Foundation keyset configured (stoa-initialise.yaml)
- [ ] Foundation private key securely stored
- [ ] 7 Stoa Master keysets configured
- [ ] Stoa Master private keys distributed to trustees
- [ ] Namespace admin/operate keysets configured

### ☐ Genesis Payloads
- [ ] `cabal run ea` completed successfully
- [ ] Genesis payload files generated
- [ ] No errors in payload generation

### ☐ Build
- [ ] `cabal build lib:chainweb` succeeds
- [ ] `cabal build chainweb-node` succeeds
- [ ] Node binary runs without errors

### ☐ Network
- [ ] Bootstrap nodes configured
- [ ] Ports open (default: 1789 for P2P, 1848 for API)
- [ ] DNS/IP addresses assigned

---

## Files Summary

| File | Purpose | Must Configure |
|------|---------|----------------|
| `pact/coin-contract/stoa.pact` | STOA contract | GENESIS-TIME |
| `pact/coin-contract/v1/stoa.pact` | Versioned contract | GENESIS-TIME |
| `src/Chainweb/GasPrice.hs` | Gas price module | stoaGenesisTime |
| `pact/coin-contract/stoa-initialise.yaml` | Genesis init | Foundation keyset |
| `pact/genesis/stoachain/stoa-masters.yaml` | Governance | 7 master keysets |
| `pact/genesis/stoachain/ns.yaml` | Namespaces | Admin keysets |

---

## Troubleshooting

### Gas Price Validation Failures

**Symptom**: Transactions rejected with "Gas price below minimum"

**Cause**: Genesis time mismatch between Pact and Haskell

**Fix**: Ensure `GENESIS-TIME` in stoa.pact equals `stoaGenesisTime` in GasPrice.hs

### Genesis Payload Generation Fails

**Symptom**: `cabal run ea` produces errors

**Cause**: Missing or malformed YAML files

**Fix**: Check all keysets are valid JSON/YAML format

### Node Won't Start

**Symptom**: Node crashes on startup

**Cause**: Missing genesis payloads or build errors

**Fix**: 
1. Re-run `cabal run ea`
2. Re-build with `cabal build chainweb-node`
3. Check for compilation errors

---

## Related Documentation

- **Yang Emission System**: `docs/EMISSION_SYSTEM.md`
- **Yin Earnings (Gas)**: `docs/GAS_PRICE_SYSTEM.md`
- **Genesis System**: `cwtools/ea/README.md`
- **Pact4 Removal**: `src/Chainweb/Pact4/README.md`

---

## Date

Last updated: December 7, 2025

## Attribution

StoaChain Node Launch Checklist - Pre-launch configuration guide

