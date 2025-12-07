# StoaChain Documentation

<p align="center">
<img src="assets/StoaLogo.png" width="200" height="200" alt="StoaChain Logo" title="StoaChain">
</p>

<h3 align="center">A Proof-of-Work Parallel-Chain Protocol</h3>

> **StoaChain** is a next-generation blockchain forked from Kadena's Chainweb protocol, featuring a custom economic model, streamlined architecture, and Pact 5 from genesis.

---

## Documentation Index

### AncientStoa (Chainweb Node)

| Document | Description |
|----------|-------------|
| [**Main README**](docs/chainweb-node/README.md) | Complete overview of StoaChain, STOA/URSTOA tokens, architecture, and configuration |
| [**Emission System**](docs/chainweb-node/EMISSION_SYSTEM.md) | Deterministic emission formula, global supply registry, foundation share distribution |
| [**Gas Price System**](docs/chainweb-node/GAS_PRICE_SYSTEM.md) | Dynamic minimum gas price, time-based increases |
| [**Genesis System**](docs/chainweb-node/GENESIS_SYSTEM.md) | Genesis payload generation, transaction order, keysets |
| [**Pact 4 Removal**](docs/chainweb-node/PACT4_REMOVAL.md) | Detailed log of Pact 4 code removal |
| [**CW Tools**](docs/chainweb-node/CWTOOLS.md) | Command-line tools for Chainweb |
| [**Store**](docs/chainweb-node/STORE.md) | Database storage layer |

### AncientPact (Pact 5.4.1 Fork)

| Document | Description |
|----------|-------------|
| [**Main README**](docs/pact-5/README.md) | AncientPact overview and StoaChain extensions |
| [**chain-data Extensions**](docs/pact-5/chain-data.md) | Documentation for `global-supply-register` and `external-fpa` fields |

---

## Quick Links

### Token Economics

- **STOA Token**: Native currency with deterministic emission
  - Genesis Supply: 12M STOA (on Chain 0)
  - Emission: `(Ceiling - GlobalSupply) / (EMISSION_SPEED × BPD)`
  - 90% to miners, 10% to URSTOA-Vault stakers

- **URSTOA Token**: Virtual mining token (Chain 0 only)
  - Fixed Supply: 1M URSTOA
  - Stake in URSTOA-Vault to earn 10% of all block emissions
  - RPS (Reward Per Share) mechanism for gas-efficient distribution

### Chain-Data Extensions

StoaChain extends Pact's `chain-data` with:

| Field | Type | Description |
|-------|------|-------------|
| `global-supply-register` | Decimal | Total STOA supply across all chains |
| `external-fpa` | Decimal | Foundation Pending Amount from chains 1-9 (Chain 0 only) |

### Networks

| Network | Chains | Graph | Purpose |
|---------|--------|-------|---------|
| StoaMainnet01 | 10 | Petersen | Production |
| StoaTestnet02 | 3 | Triangle | Testing |
| StoaDevnet03 | 3 | Triangle | Development |

---

## Source Repositories

> ⚠️ **Note**: The source repositories are currently private. This documentation repository provides public access to the project documentation.

- **AncientStoa** (Chainweb Node): StoaChain node implementation
- **AncientPact** (Pact 5.4.1): Pact smart contract language fork with StoaChain extensions

---

## License

StoaChain is released under the **MIT License**.

---

*StoaChain - Building the future of decentralized computing*

*Last Updated: December 2025*

