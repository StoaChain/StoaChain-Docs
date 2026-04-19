# StoaChain Node Launch Checklist

This document is a **corrected checklist** for launching a StoaChain node.

> ⚠️ **Supersedes earlier drafts.** Earlier versions of this file described a "centralized configuration" flow built around `stoachain-config.yaml` and `scripts/apply-config.sh`, and a two-file genesis-time sync between `pact/coin-contract/stoa.pact` and `src/Chainweb/GasPrice.hs`. **None of that matches the node repo today.** The central-config script does not exist, and the gas-price sync requirement is obsolete because the dynamic gas-price ramp has not shipped. Operators should use the real deploy tooling (`deploy.sh`, `run-stoa.sh`, `stoa-node.service`) in the `github.com/StoaChain/stoa-chain` repository.

---

## What "launching a StoaChain node" actually looks like

StoaChain is a single network (not three). Running a node means:

1. Build the node from `github.com/StoaChain/stoa-chain` with GHC 9.10.1 / Cabal 3.14.1.1.
2. Start it with `--chainweb-version=stoa` (version code `0x0000000A`, 10 chains, Petersen graph, 30 s block delay).
3. Let it sync against the existing bootstrap (`node1.stoachain.com:1789`).

There is no separate `stoatestnet02` / `stoadevnet03` to configure. There is no mandatory pre-genesis configuration step that individual node operators must perform — the chain's genesis already happened at `2026-02-23T18:00:00.000000` UTC.

The checklist below is relevant only if you are **re-generating genesis** (i.e., bringing up a *new* StoaChain-style chain from scratch, or regenerating the shipped chain's genesis to change governance, supply, etc.). It is not required for operating a node on the live chain.

---

## Quick reference

| Priority | Item | When it applies | Status |
|----------|------|-----------------|--------|
| 🔴 Critical | Genesis time in `src/Chainweb/Version/Stoa.hs` | Only if re-generating genesis | ☐ |
| 🔴 Critical | Coin module (`pact/stoa-coin/new-coin.pact`) reviewed | Only if re-generating genesis | ☐ |
| 🔴 Critical | Genesis YAMLs (`pact/genesis/stoa/`) reviewed | Only if re-generating genesis | ☐ |
| 🟡 Important | Bootstrap list in `_versionBootstraps` | Any node launch | ☐ |
| 🟡 Important | Foundation keyset + 7 Stoa Masters keysets in genesis | Only if re-generating genesis | ☐ |
| 🟢 Verify | `cabal run ea` generates payload cleanly | Only if re-generating genesis | ☐ |
| 🟢 Verify | `cabal build chainweb-node` succeeds | Any node launch | ☐ |

---

## 🔴 CRITICAL: Genesis time (only if re-generating genesis)

The authoritative genesis time lives in **one** place:

```haskell
-- src/Chainweb/Version/Stoa.hs
_genesisTime = 2026-02-23T18:00:00.000000 UTC
```

There is **no** separate `stoaGenesisTime` constant in `src/Chainweb/GasPrice.hs` in the shipped code. Earlier drafts that required syncing `GENESIS-TIME` (Pact) with `stoaGenesisTime` (Haskell) described a design that was not landed. Do not try to create the sync — the Haskell file does not exist.

If the dynamic gas-price ramp ships in the future, the Pact-side `GENESIS-TIME` constant would live in `pact/stoa-coin/new-coin.pact` and the ramp-computation module would need the same time. Today, neither constant exists; both are roadmap-only. See [`GAS_PRICE_SYSTEM.md`](GAS_PRICE_SYSTEM.md).

---

## 🔴 CRITICAL: Coin module & genesis payloads (only if re-generating)

| Path | Purpose |
|------|---------|
| `pact/stoa-coin/new-coin.pact` | Single STOA coin module used at genesis |
| `pact/genesis/stoa/` | Per-chain genesis YAML/pact payloads for the `stoa` network |

The `pact/coin-contract/stoa.pact` / `pact/coin-contract/v1/stoa.pact` paths that appeared in earlier drafts do not match the live tree — those referred to a planned rewrite that was not landed.

---

## 🔴 CRITICAL: Governance keysets (only if re-generating genesis)

The coin module is governed by **7 Stoa Masters keysets** (`stoa-ns.stoa_master_one` … `stoa-ns.stoa_master_seven`) with `enforce-one` — any 1-of-7 can authorise. To re-generate genesis you need 7 unique ED25519 keypairs; wire them into the genesis YAMLs under `pact/genesis/stoa/`.

Namespace administration uses a separate `ns-admin-keyset` / `ns-operate-keyset` pair — those gate namespace operations (creating new namespaces, administering `stoa-ns`) and are not the coin-module governance.

A **foundation keyset** controls the `stoa-foundation` account that holds the foundation's genesis STOA allocations — including the 10M held until the ICO finalises. Configure this keyset in the genesis YAMLs as well.

Inspect the current `pact/genesis/stoa/` layout before copying key structures from older docs; filenames and exact schema have evolved across commits.

---

## 🟡 IMPORTANT: Bootstrap

The live chain's `_versionBootstraps` in `src/Chainweb/Version/Stoa.hs` contains only:

```
node1.stoachain.com:1789
```

`node2.stoachain.com` is a public HTTPS endpoint for `/info` queries, **not** a protocol-level bootstrap. Adding new bootstraps requires a code change in `Version/Stoa.hs`, not a config tweak.

Check liveness on public endpoints:

```bash
curl -s https://node1.stoachain.com/chainweb/0.0/stoa/cut | jq .
curl -s https://node2.stoachain.com/chainweb/0.0/stoa/cut | jq .
```

---

## 🟢 VERIFY: Build the node

### Toolchain

GHC 9.10.1 and Cabal 3.14.1.1 (as used in `deploy.sh`).

### System dependencies (Ubuntu/Debian)

```bash
sudo apt-get install -y \
    libsnappy-dev \
    libgflags-dev \
    zlib1g-dev \
    libbz2-dev \
    liblz4-dev \
    libzstd-dev \
    librocksdb-dev
```

### Build

```bash
cd stoa-chain

# Build the library
cabal build lib:chainweb

# Build the node executable
cabal build chainweb-node
```

### Regenerate genesis (only if needed)

```bash
cd cwtools
cabal run ea
# Payload is emitted under src/Chainweb/BlockHeader/Genesis/
```

Only one payload is produced for the single `stoa` network — not three. The `StoaMainnet0to9Payload.hs` / `StoaTestnet0to2Payload.hs` / `StoaDevnet0to2Payload.hs` triad from earlier drafts does not match the shipped tree.

---

## 🟢 VERIFY: Run the node

Preferred deployment tooling in the repo:

| Script / unit | Purpose |
|---------------|---------|
| `deploy.sh` | End-to-end setup (toolchain install, build, configure) |
| `run-stoa.sh` | Launch wrapper used by the systemd unit |
| `stoa-node.service` | systemd unit that runs `run-stoa.sh` as a service |

Minimum manual invocation:

```bash
chainweb-node --chainweb-version=stoa
```

---

## Pre-Launch Verification Checklist

### ☐ Only if re-generating genesis
- [ ] `_genesisTime` in `src/Chainweb/Version/Stoa.hs` reviewed
- [ ] `pact/stoa-coin/new-coin.pact` reviewed
- [ ] All files under `pact/genesis/stoa/` reviewed
- [ ] 7 Stoa Masters keysets configured (coin-module governance, 1-of-7 enforce-one)
- [ ] `ns-admin-keyset` / `ns-operate-keyset` configured (namespace governance)
- [ ] Foundation keyset configured (controls `stoa-foundation` account)
- [ ] `cabal run ea` succeeds
- [ ] Generated payload wired into `Version/Stoa.hs`

### ☐ Every node launch
- [ ] `cabal build chainweb-node` succeeds
- [ ] `_versionBootstraps` reachable (`node1.stoachain.com:1789`)
- [ ] Ports open: 1789 (P2P), 1848 (service API) unless overridden
- [ ] `curl /info` on the running node returns `"nodeVersion":"stoa"`, `"nodeNumberOfChains":10`, `"nodeBlockDelay":30000000`

---

## Obsolete sections from earlier drafts

The items below appeared in earlier versions of this checklist but **do not apply** and should not be reproduced:

- "Centralized configuration" via `stoachain-config.yaml` + `./scripts/apply-config.sh` — neither file exists.
- The `pact/coin-contract/stoa.pact` ↔ `src/Chainweb/GasPrice.hs` genesis-time sync — the Haskell file does not exist; the sync is not a requirement.
- Network-specific configs under `pact/genesis/stoachain/stoa-masters-testnet.yaml` and `-devnet.yaml` — there is only one network.
- "Generates `StoaMainnet0to9Payload.hs`, `StoaTestnet0to2Payload.hs`, `StoaDevnet0to2Payload.hs`" — there is one payload.

---

## Related Documentation

- **Emission System**: [`EMISSION_SYSTEM.md`](EMISSION_SYSTEM.md)
- **Gas Price System**: [`GAS_PRICE_SYSTEM.md`](GAS_PRICE_SYSTEM.md)
- **Genesis System**: [`GENESIS_SYSTEM.md`](GENESIS_SYSTEM.md)
- **Pact 4 retrospective**: [`PACT4_REMOVAL.md`](PACT4_REMOVAL.md)

---

*Last updated: 2026-04-19*
