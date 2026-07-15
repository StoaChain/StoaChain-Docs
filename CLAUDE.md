# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This is a **documentation-only repo** for the StoaChain project. It contains no source code, build system, or tests — only Markdown and one logo asset. The actual implementation lives in a separate node repo (referenced from `deploy.sh` as `github.com/StoaChain/stoa-chain`, on disk at `D:\_Claude\StoaChain`) — a **software fork** of Kadena's `chainweb-node`. The Pact 5.4 fork the node depends on is likewise separate.

There is no package manager, linter, or test runner configured for this repo. Treat changes here as prose edits. The build/run/test commands the docs describe (`cabal build chainweb-node`, `cabal run ea`, `cabal test`, `deploy.sh`) belong to the node repo and cannot be executed inside this repo.

## Reconciliation note — docs vs. reality (2026-04-19)

The docs in this repo were originally written around a planned rewrite that never landed as described. A first pass of corrections (handed off in [DOCS-CORRECTIONS.md](DOCS-CORRECTIONS.md)) flagged many items as "under review". A second reconciliation pass with the project owner has since confirmed most of those items — see the "Ground truth" section below. When a claim in the DOCS-CORRECTIONS.md handoff contradicts the ground-truth section below, **the ground-truth section below wins** (it is newer). DOCS-CORRECTIONS.md is preserved as a historical record of the first pass.

## Document layout and authority

- [README.md](README.md) at the root is the canonical public overview.
- [docs/chainweb-node/](docs/chainweb-node/) holds deep-dive node docs: `EMISSION_SYSTEM.md`, `GAS_PRICE_SYSTEM.md`, `GENESIS_SYSTEM.md`, `NODE_LAUNCH_CHECKLIST.md`, `PACT4_REMOVAL.md`.
- [docs/pact-5/](docs/pact-5/) holds the Pact-5.4-fork overview and the `chain-data` extension reference.
- [DOCS-CORRECTIONS.md](DOCS-CORRECTIONS.md) is a reconciliation handoff (not user-facing documentation).

When facts are duplicated between the root README and a deep-dive doc, update both.

Some internal links inside the deep-dive files point at paths like `src/Chainweb/...`, `pact/...`, `cwtools/...` — those refer to files in the **node repo**, not this repo. Don't "fix" those paths to resolve inside this repo.

## Ground truth about the live StoaChain chain (use these facts)

These are the facts confirmed against the node repo, live `/info` endpoint, and direct owner confirmation. Keep documents consistent with them.

### Network & build
- **Single network** named `stoa`. Version code `0x0000000A` (= 10). Defined in `src/Chainweb/Version/Stoa.hs` (module `Chainweb.Version.Stoa`). CLI flag is `--chainweb-version stoa`. No separate testnet/devnet ChainwebVersion exists.
- **10 chains, Petersen graph, 30-second block delay.** Confirmed by `/info` on `node1.stoachain.com` and `node2.stoachain.com` (`"nodeVersion":"stoa"`, `"nodeNumberOfChains":10`, `"nodeBlockDelay":30000000`). Package version on those nodes: `2.32.0`.
- **Block gas limits: 1.6M default / 2M max.** `_configBlockGasLimit = 1_600_000` in `src/Chainweb/Chainweb/Configuration.hs`; `_versionMaxBlockGasLimit = Bottom (minBound, Just 2_000_000)` in `src/Chainweb/Version/Stoa.hs`. Production nodes run at 2M.
- **Bootstraps.** Only `node1.stoachain.com:1789` is listed in `_versionBootstraps`. `node2.stoachain.com` is a public HTTPS endpoint but not a protocol-level bootstrap.
- **Genesis time**: `2026-02-23T18:00:00.000000` UTC, defined as `_genesisTime = AllChains $ BlockCreationTime [timeMicrosQQ| 2026-02-23T18:00:00.000000 |]` in `src/Chainweb/Version/Stoa.hs`. **There is no `stoaGenesisTime` in `src/Chainweb/GasPrice.hs`**; older docs that claim a Pact-vs-Haskell sync requirement are stale.

### Pact
- **Pact 5 is stock.** The node pulls in unmodified upstream Pact 5.4 (the final Pact release before Kadena LLC dissolved). **No `chain-data` extensions** (no `global-supply-register`, no `external-fpa`), no custom `PublicData` fields, no `src/Chainweb/Pact/GlobalSupply.hs`. Any doc that describes "AncientPact" or chain-data extensions is wrong and should be corrected.
- **Pact 4 infrastructure retained in the node code** (`src/Chainweb/Pact4/`, `Chainweb.Test.Pact4.*`, `pact` pin in `cabal.project`) — extraction was attempted and abandoned. State both facts together wherever Pact version comes up: code carries Pact 4 infra, live chain runs 5.4.

### Coin module & emission
- **Coin module** is at `pact/stoa-coin/new-coin.pact` (single file; defines both STOA and URSTOA, plus the UrStoaVault state). **Governance: 7 Stoa Masters keysets** control the coin module. The older `ns-admin-keyset` / `ns-operate-keyset` mentioned in earlier docs apply to namespace operations, not to coin-module governance.
- **Emission is computed entirely in Pact**, by `coin.URC_Emissions`, which returns `[<block-emission> <urv-emission>]` — i.e. a per-block reward for the miner plus a per-block credit to the UrStoaVault. The formula derives yearly allocation from Gregorian-leap-aware day counts, then divides by expected blocks-per-year and chains to produce the per-chain per-block emission. It was rewritten from a recursive formulation into a linear equivalent so Pact could evaluate it without iteration. Each year the amount steps down.
- **Supply is tracked in a Pact table inside the coin module** (not via Haskell-side querying). Mint and burn events update this table, and the explorer reads it for live per-chain supply. There is **no `Chainweb.Pact.GlobalSupply` module** and no Haskell-side supply aggregation.
- **Emission split**: 90% of every block's `URC_Emissions` goes to the miner (Yang block reward). 10% goes to the UrStoaVault, which lives on Chain 0 and distributes to URSTOA stakers via RPS. The cross-chain settlement mechanism (how chains 1–9 feed their 10% share into the Chain-0 vault) is implemented inside the coin module — document at the interface level (`URC_Emissions` produces `urv-emission` credited to the vault), not with Haskell-level delta-tracking claims.
- **CSV**: `rewards/miner_rewards.csv` is still present in the node repo and is still read by the node, but the authoritative emission is the Pact-computed amount from `URC_Emissions`. The CSV value is ignored. Describing the CSV as "removed" is wrong.

### Tokens & supply
- **Genesis STOA supply: 16,000,000 STOA**, split at genesis as:
  - 10M — ICO allocation (held in the foundation account until the ICO finalizes, then distributed)
  - 2M — Foundation
  - 4M — Migration from Ouronet (the prior project running on Kadena)
- **URSTOA supply: 1,000,000 URSTOA** (fixed, minted at genesis, Chain 0 only). Distribution:
  - 250k — split between blockchain founders
  - 250k — ICO sale (**1 URSTOA per $5** contributed)
  - 500k — foundation
- **URSTOA is live** — defined inside the coin module (not a separate module), can be staked in UrStoaVault to earn the 10% Yang share. The explorer exposes "UrStoa Rich List" and "Vault Participation" tabs.

### Gas price
- **Minimum gas price ramp is not shipped at the protocol level.** The live chain uses the upstream static minimum (`_configMinGasPrice = 1e-8`). The coin module does contain a Pact function that computes the intended ramp value, but it is not wired into the protocol-level minimum check. The 10,000 → 400,000 ANU / +1 ANU per 3 h / 133 y cap design is still the target; no upgrade date has been published.

### Live module set (under `stoa-ns`, plus upgraded `coin` and `ns` in root)
- `stoic-predicates` — custom keyset predicates anyone can use (e.g., `5-of-9` patterns, custom authorization logic).
- `stoic-xchain` — constructs the autonomous accounts that pay for on-chain cross-chain transfers.
- `stoic-fungible-v1` — StoaChain's equivalent of `fungible-v2`, under local nomenclature.
- `ur-stoic-fungible-v1` — interface for the URSTOA token (token itself is defined in the coin module).
- `fungible-v1` — identical to upstream `fungible-v2` (StoaChain just started at v1).
- `fungible-xchain-v1` — identical to Kadena's `fungible-xchain-v1`.
- `gas-payer-v1` — gas-station interface, identical to Kadena's.
- **Post-genesis upgrades**: the coin module has been upgraded since genesis, mostly to fix bugs in the UrStoaVault logic (slightly incorrect at genesis, now functioning correctly). When documenting live behavior, fetch via `describe-module` or the explorer — don't copy the genesis source into the docs.

## Style conventions observed in existing docs

- ASCII box diagrams (├──, │, ─) for architecture and flow diagrams — match this style rather than introducing Mermaid or images.
- Comparison tables consistently use the column order **Aspect | Kadena | StoaChain**.
- The "Yang Emission" / "Yin Earnings" terminology is project vocabulary (Yang = block rewards, Yin = gas fees) — use it as-is.
- The "Development Method" section at the bottom of both top-level READMEs is intentional flavor text describing the project's origin story; preserve it verbatim if editing those READMEs.
- **TRANSFER capability nomenclature**: the coin module retains the original Kadena `transfer` (with `@managed` TRANSFER capability) for compatibility, and **additionally** exposes StoaChain-named entry points: `C_Transfer` (wraps the managed `transfer`) and `C_Transmit` (same transfer logic but with a non-managed capability). URSTOA mirrors the pattern. Don't describe TRANSFER as "non-managed" flat — describe it as dual (managed + non-managed available side by side). Validations (sender guard, amount, precision) fire on both paths; the difference is only how the capability is acquired at sign time. The interface name used in the *live* modules is `stoa-ns.stoic-fungible-v1`, not `StoaFungibleV1`.
- Use "software fork" (or "software fork of `chainweb-node`") when describing StoaChain's relationship to Kadena. Avoid bare "a fork of Kadena", which connotes a *chain fork* (ledger split). StoaChain has its own genesis and no Kadena ledger history.
