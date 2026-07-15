# StoaChainDocs — Corrections Required

> **Read me first.** This file is a handoff note, not regular documentation. It describes how the current contents of this repository diverge from the actual live StoaChain project, and what should change. Written 2026-04-19 by a Claude session working in the StoaChain code repository (`D:\_Claude\StoaChain`). A future Claude session in a chat tied to this docs repository is expected to read this file and update the docs accordingly.

## TL;DR

StoaChainDocs was originally written around a planned rewrite ("AncientStoa") that never landed as described. The **live StoaChain** (running on `node1.stoachain.com` and `node2.stoachain.com`, genesis 2026-02-23) is a **minimal software fork of Kadena's chainweb-node**: essentially the upstream code, plus a new `stoa` ChainwebVersion, modified Pact genesis files, and raised block gas limits. Several specific claims throughout these docs are wrong and need to be corrected. **This is a docs update; do not propose code changes — the repo being documented is the code as it actually is.**

The live code repository is at `D:\_Claude\StoaChain` on this machine; treat it as the source of truth when a claim in the docs conflicts with it.

### A note on the word "fork"

Use "software fork" (or "software fork of chainweb-node") when describing StoaChain's relationship to Kadena. **Do not** say simply "a fork of Kadena" without qualification, because that phrasing commonly implies a *chain fork* — a split of an existing ledger at a given block. **StoaChain is not a chain fork of Kadena**: it has its own genesis, its own network identity (`stoa`), and shares no ledger history with Kadena's mainnet. It reuses the chainweb-node *software* with modifications, and runs as an entirely independent blockchain.

Acceptable phrasings across the docs:
- "StoaChain is a **software fork** of Kadena's chainweb-node, running as its own independent blockchain with its own genesis."
- "StoaChain is a **minimal fork** of chainweb-node — same Chainweb consensus protocol, new network, new genesis, raised gas limits."
- "StoaChain is **built on Kadena's Chainweb protocol**" (if you want to avoid "fork" entirely).

Remove any language implying StoaChain split off from Kadena's existing chain.

---

## Ground truth — what is actually true about StoaChain today

The facts below are verified against the StoaChain code repo and confirmed by the project owner. Use them to correct the documentation.

### 1. Networks — **one** network, not three

StoaChainDocs currently claims three networks (`stoamainnet01` 0x00000015, `stoatestnet02` 0x00000016, `stoadevnet03` 0x00000017). **This is wrong.**

Actual:
- **One network**, named `stoa`.
- Version code `0x0000_000A` (= 10).
- Defined in the code repo at `src/Chainweb/Version/Stoa.hs` — note the module is **`Chainweb.Version.Stoa`**, not `Chainweb.Version.StoaChain` as the architecture diagram implies.
- No testnet / devnet exist as distinct Chainweb versions. Development and testing happen against the same `stoa` network.
- The CLI flag is `--chainweb-version stoa`.

Every mention of `stoamainnet01`, `stoatestnet02`, `stoadevnet03`, the Triangle graph, the three-version comparison table row, or version codes 0x15/0x16/0x17 needs to be removed or replaced with single-network language.

### 2. Mainnet chain count — 10 (Petersen graph) — **unchanged, correct**

This claim in StoaChainDocs is accurate and does not need to change. Keep the Petersen-graph ASCII diagram. Remove the Triangle-graph diagram (no three-chain testnet exists).

### 3. Block gas limits — **1.6M default / 2M max**, not 400k/500k

StoaChainDocs says 400k default / 500k max. That was an earlier state. **Current, live state** (code commits `2b62535` and later):
- `_configBlockGasLimit` default: **1,600,000** (in `src/Chainweb/Chainweb/Configuration.hs`).
- Hard cap in the version definition: **2,000,000** (`_versionMaxBlockGasLimit = Bottom (minBound, Just 2_000_000)` in `src/Chainweb/Version/Stoa.hs`).
- Production nodes **run at 2M** and have been tested up to near-2M gas blocks in practice without issue.

Every occurrence of "400k", "500k", "400,000", "500,000", "400k default / 500k max", or the Kadena-vs-StoaChain comparison-table row for gas must be updated to **"1.6M default / 2M max"** (and the throughput math in any derived analysis redone accordingly).

### 4. Pact 4 was **NOT** removed from the codebase

StoaChainDocs currently contains an entire `PACT4_REMOVAL.md` file claiming ~3,300 lines of Pact 4 code were removed and states throughout the main docs that StoaChain "uses only Pact 5 from genesis" / is "Pact 5 exclusive". **This is wrong and must be rewritten.**

Actual state:
- The StoaChain code repo **still contains all Pact 4 infrastructure**: `src/Chainweb/Pact4/`, the `Chainweb.Test.Pact4.*` test modules, and the `pact` source-repository-package pin in `cabal.project` (alongside the `pact-5` pin). Removal was attempted but abandoned because Pact 4 and Pact 5 share code paths and cleanly extracting Pact 4 was out of reach without more expertise.
- The **live chain**, however, was upgraded at the Chainweb level to **Pact 5.4** (the latest Pact version released before Kadena LLC dissolved). There is no supported path to execute Pact 4 code on the live chain by default — the live chain runs Pact 5.4 in practice.
- In short: **"the code retains Pact 4 infrastructure; the live chain runs Pact 5.4"**. Both statements must appear together wherever this comes up.

Suggested treatment:
- Rename or replace `PACT4_REMOVAL.md`. It should no longer present Pact 4 removal as accomplished. Either turn it into a "Pact 4 removal — attempted, abandoned" retrospective, or delete it and fold a short paragraph into the main README. Check with the project owner before deleting.
- Remove all "Pact 5 exclusive", "Pact 5 from genesis", "~3,300 lines removed" claims from the main READMEs.
- Replace with: "StoaChain is built against Pact 5.4 (the final Pact release). The node still carries Pact 4 infrastructure internally for compatibility reasons, but all on-chain execution is Pact 5.4."

### 5. Coin contract path and module nomenclature

StoaChainDocs references `pact/coin-contract/stoa.pact` with submodules `STOA`, `URSTOA`, `URSTOA-Vault` and a `StoaFungibleV1` interface.

Actual state in the genesis code:
- The genesis coin contract lives at **`pact/stoa-coin/new-coin.pact`** (not `pact/coin-contract/stoa.pact`).
- Genesis coin is a **single-module** Pact contract implementing a `fungible-v2`-style interface (non-managed TRANSFER, per user-stated design), **not** the three-submodule STOA + URSTOA + URSTOA-Vault architecture described.
- **The live chain has diverged from the genesis code**. Post-genesis module upgrades have been applied, introducing custom nomenclature (project-specific interfaces and module names visible on the explorer: `stoa-ns.stoic-predicates`, `stoa-ns.stoic-xchain`, `stoa-ns.stoic-fungible-v1`, `stoa-ns.ur-stoic-fungible-v1`, `stoa-ns.fungible-v1`, `stoa-ns.fungible-xchain-v1`, `stoa-ns.gas-payer-v1`, plus upgraded `coin` and `ns` in the root namespace). Some live modules also contain bug-fix changes relative to their genesis version. **The live code is slightly more complex than the genesis code.**

Docs should:
- Stop pointing to `pact/coin-contract/stoa.pact`. The correct genesis path is `pact/stoa-coin/new-coin.pact`.
- Clearly separate **genesis state** from **live state** when describing contracts. Genesis = what's in the repo. Live = what's on-chain after upgrades.
- The live module layout (from the explorer) is the authoritative description of what's currently on-chain. Actual live source lives in the chain itself; when documenting live behavior, either fetch via `describe-module` from a node, or link to the explorer pages rather than copying source into the docs.
- Until the live module source has been pulled and reviewed, avoid inventing the structure/responsibilities of `stoa-ns.stoic-*` modules. Ask the project owner to paste the live sources if deeper documentation of them is needed.
- The `StoaFungibleV1` concept (merged `fungible-v2 + fungible-xchain-v1`, non-managed TRANSFER) may be partially accurate as a description of the design intent, but the **interface names on the live chain are `stoa-ns.stoic-fungible-v1` / `stoa-ns.ur-stoic-fungible-v1` / `stoa-ns.fungible-xchain-v1`**, not `StoaFungibleV1`. Use the live names.

### 6. Emission — CSV kept for compatibility; Pact computes real emission

StoaChainDocs positions StoaChain's emission as "deterministic formula replacing CSV-based vesting". This is misleading in two directions.

Actual state:
- The CSV file `rewards/miner_rewards.csv` **is still present** in the code repo and is still read by the node. It was retained for backward compatibility because extracting the CSV machinery cleanly wasn't feasible.
- However, the **Pact-level coin module computes its own emission** (the formula described in `EMISSION_SYSTEM.md` — declining emission, vault split, etc.) and **ignores** the value the node would otherwise serve from the CSV. The amount actually minted is the Pact-computed amount, not the CSV value.
- Net effect: **the CSV is vestigial; the Pact code is authoritative.** Documentation that says CSV was "removed" or "replaced" is wrong. Documentation that says emission is deterministic-formula-based is correct in effect.

Every place the docs describe this should state both facts together: "CSV retained for structural/compat reasons but not authoritative; actual emission is computed inside the Pact coin module and ignores the CSV value."

Kadena-vs-StoaChain comparison table row for "Emission Model" / "Initial Supply" needs to reflect this nuance rather than presenting CSV removal as accomplished.

### 7. Roadmap: gradual minimum-gas-price ramp (planned, not shipped)

StoaChain has a planned protocol-level change to **gradually increase the minimum allowed gas price** over time. The intended cadence is a bump every **3 hours**. This was not shipped at genesis due to time constraints and will be added in a later upgrade.

What this means for the docs:
- Current `GAS_PRICE_SYSTEM.md` appears to describe a "dynamic minimum gas price, time-based increases" as if it were already implemented. **It is not.** The live node is running with a static minimum (whatever the chainweb-node default is — currently `_configMinGasPrice = 1e-8` in `src/Chainweb/Chainweb/Configuration.hs`, inherited from upstream).
- The correct positioning is **"planned feature, not yet implemented on the live chain."** Document the design intent if the project owner wants a public roadmap item, but label it clearly as future work — do not describe it as shipped.
- If `GAS_PRICE_SYSTEM.md` currently claims this is live, add a banner at the top: *"Roadmap item — not yet shipped. The current live chain uses a static minimum gas price inherited from upstream chainweb-node. The periodic ramp described below is planned for a future protocol upgrade."*
- Update the Kadena-vs-StoaChain comparison table "Gas Price" row: either remove the "dynamic" claim or split it into *current* ("static minimum, same as Kadena") and *planned* ("periodic ramp, +N every 3 h").
- The specific numbers (starting minimum, per-ramp increment, total ramp duration, cap) need to come from the project owner before being published. Ask; do not invent them.

---

## Bootstrap nodes and live `/info` endpoints

The live StoaChain mainnet is currently served by **two operator nodes**, both advertising over standard HTTPS (reverse-proxied on port 443 in front of the chainweb service API):

- **Node 1**: `https://node1.stoachain.com/info`
- **Node 2**: `https://node2.stoachain.com/info`

As of 2026-04-19 both endpoints return:

```json
{
  "nodeApiVersion": "0.0",
  "nodeBlockDelay": 30000000,
  "nodeChains": ["0","1","2","3","4","5","6","7","8","9"],
  "nodeGenesisHeights": [["0",0], ... ["9",0]],
  "nodeGraphHistory": [[0, [[0,[2,3,5]], [1,[6,4,3]], ..., [9,[4,5,8]]]]],
  "nodeLatestBehaviorHeight": 1,
  "nodeNumberOfChains": 10,
  "nodePackageVersion": "2.32.0",
  "nodeVersion": "stoa"
}
```

This confirms the live protocol: **single network named `stoa`**, 10 chains, 30 s block delay, package version 2.32.0 — all consistent with corrections 1/2 above.

**Important nuance re: "bootstrap node".** In Chainweb terminology, a *bootstrap node* is specifically a peer whose hostname is baked into the `ChainwebVersion` definition so new nodes can discover the network at first start. The code repo's `src/Chainweb/Version/Stoa.hs` currently lists **only `node1.stoachain.com:1789`** in `_versionBootstraps`. Node 2 is running and serving the chain, but is **not currently wired in as a protocol-level bootstrap peer** — it is a second operator node reachable via HTTPS, not an auto-discoverable bootstrap. Until the code is updated to add node2 to the bootstrap list, the docs should either:

- (a) describe **node 1 as the sole canonical bootstrap, with node 2 as an additional public endpoint**, or
- (b) describe both as "mainnet nodes" without claiming both are bootstraps.

Do not claim both are bootstraps unless the project owner has confirmed a code update adding node2 to `_versionBootstraps` has shipped.

Recommended README content derived from this:
- A "Bootstrap / Public Nodes" section listing both URLs with their roles made clear.
- Replace the Kadena-specific "Current Mainnet Bootstrap Nodes" list (us-e1/us-e2/... etc.) with the actual StoaChain list.
- Include a `curl` example against `/info` as a quick liveness check, mirroring the existing "Monitoring the health" pattern in Kadena's original README.

---

## Asset: the StoaChain logo

The StoaChainDocs repo already ships the canonical logo at [`assets/StoaLogo.png`](assets/StoaLogo.png). Any README — here or in the code repo — that wants a header image should reference that file. The Kadena logo (`https://i.imgur.com/bAZFAGF.png`) must not survive into StoaChain-branded documentation; delete any remaining `<img src="https://i.imgur.com/bAZFAGF.png">` tags or equivalent and substitute the Stoa logo.

For the code repo at `D:\_Claude\StoaChain`, the logo file is not currently present; it will need to be copied in (as `assets/StoaLogo.png` or similar) so its README can reference it locally. That copy is a code-repo action, outside the scope of this docs update, but worth noting in your summary to the project owner.

---

## Per-file update pointers

The other Claude session will need to read each file and apply corrections based on the ground truth above. Pointers below are not line-level diffs; they indicate the most likely locations and the direction of the change.

### `/README.md` (root, public overview)
- **Comparison table** (Key Differences from Kadena section): update "Networks" row, "Chain Count" row, "Block Gas Limit" row, "Pact Version" row, "Emission Model" row, "Initial Supply" row per corrections 1, 3, 4, 6.
- **Networks section**: collapse to a single-network description. Remove `stoatestnet02` / `stoadevnet03` / Triangle-graph content.
- **"Pact 5 Exclusive" bullet** in the Overview: rewrite per correction 4.
- **"Streamlined Codebase: Removed legacy Pact 4 execution code (~3,300 lines)"** bullet: delete or rewrite truthfully.
- **Architecture ASCII diagram**: update file paths — `Version/StoaChain.hs` → `Version/Stoa.hs`; `pact/coin-contract/stoa.pact` → `pact/stoa-coin/new-coin.pact`; describe genesis vs live modules honestly.
- **"No Allocations" bullet**: soften to reflect correction 6 (CSV kept, ignored).
- **Pre-Launch Configuration section**: the `stoachain-config.yaml` / `apply-config.sh` mechanism does not exist in the actual code repo at `D:\_Claude\StoaChain`. Verify whether it exists in a private "AncientStoa" repo or is aspirational. If it is aspirational or never existed, remove the section. Ask the project owner.
- **Source Repositories section**: states sources are "currently private" and names two private repos ("AncientStoa", "AncientPact"). The actual code at `D:\_Claude\StoaChain` (which builds the binary that runs on `node1.stoachain.com`) is not private — it is a git repo on disk and apparently hosted at `https://github.com/StoaChain/stoa-chain.git` per `deploy.sh`. Either drop the "private" framing, or clarify what the two "Ancient*" names refer to (planned rewrite? earlier design?). Ask the project owner.
- **"Last Updated"** footer: bump.

### `/CLAUDE.md` (project-level guidance for future Claude sessions)
Many "Domain invariants worth preserving across edits" items are drawn from the same outdated picture:
- Three-version-codes invariant: replace with single-version description (code 0x0000_000A, name `stoa`, 10 chains, Petersen).
- "Block gas limits: 400k default / 500k max": update to 1.6M / 2M; update the "1.0–1.25 s execution per block at 30 s block time" back-of-envelope accordingly.
- "StoaFungibleV1 ... merges `fungible-v2` + `fungible-xchain-v1` ... non-managed TRANSFER" — retain the non-managed TRANSFER invariant (it is an intentional design choice), but use the correct live module names (`stoa-ns.stoic-fungible-v1` etc.) rather than `StoaFungibleV1`.
- "URSTOA and the URSTOA-Vault exist on Chain 0 only" — verify with the project owner whether URSTOA / URSTOA-Vault are actually on-chain. The live explorer listing given did not include them. If they don't exist on-chain, remove these invariants. If they do, keep.
- "Governance: 7 Stoa Masters keysets governed by `enforce-one`" — verify with the project owner that 7 keysets is still the live design; the current genesis `ns.yaml` uses a single `ns-admin-keyset` / `ns-operate-keyset` pair, not seven masters.
- Keep the ASCII-diagram style convention, the "Aspect | Kadena | StoaChain" column order, and the "Yang Emission / Yin Earnings" terminology.
- "GENESIS-TIME synchronization" invariant: verify the file paths still match. The actual code repo does not have `pact/coin-contract/stoa.pact` or a `stoaGenesisTime` constant in `src/Chainweb/GasPrice.hs` — genesis time lives in `src/Chainweb/Version/Stoa.hs` (`_genesisTime = AllChains $ BlockCreationTime [timeMicrosQQ| 2026-02-23T18:00:00.000000 |]`). Update accordingly.

### `/docs/chainweb-node/README.md` (the "full technical README")
This file largely duplicates the root README and needs the same corrections (1, 3, 4, 5, 6). In addition:
- **Building from Source / Running a Node sections**: verify commands against `D:\_Claude\StoaChain`. The actual build uses GHC 9.10.1 / Cabal 3.14.1.1 (see `deploy.sh`); the run command is in `run-stoa.sh`. Replace any aspirational `./scripts/apply-config.sh` references with the actual deploy path (`deploy.sh`, `stoa-node.service`).
- Path references to `src/Chainweb/...`, `pact/...`, `cwtools/...`: these paths refer to the upstream StoaChain code repo and are fine to keep — just make sure the file **names** under those paths are real (`Version/Stoa.hs` not `Version/StoaChain.hs`; `stoa-coin/new-coin.pact` not `coin-contract/stoa.pact`).

### `/docs/chainweb-node/EMISSION_SYSTEM.md`
- Add a prominent note up top: "The CSV file `rewards/miner_rewards.csv` is still present in the node code for backward compatibility, but the value served from it is ignored. Actual emission is computed inside the Pact coin module per the formula below." This resolves correction 6.
- Leave the formula description itself in place if it accurately describes the Pact-side computation (verify with project owner).
- Remove claims about Pact-4-removal or CSV-removal if any are present.
- Any mention of URSTOA-Vault needs to be verified against live state (correction 5).

### `/docs/chainweb-node/GAS_PRICE_SYSTEM.md`
- Update any absolute numbers that depend on block gas limit to reflect 1.6M/2M (correction 3).
- **The "dynamic gas price" / "time-based minimum" mechanism is not shipped** (correction 7). It is a planned future feature, not live. Add a prominent "Roadmap — not yet implemented" banner at the top of this file, and rewrite the descriptive content to be clearly future-tense ("will", "is planned to") rather than present-tense ("is", "does"). The live chain runs with the upstream chainweb-node static minimum gas price (`1e-8` from `_configMinGasPrice`).
- Do not claim specific ramp increments, cadence values other than "every 3 hours", or a final floor until the project owner has approved them.

### `/docs/chainweb-node/GENESIS_SYSTEM.md`
- Update file paths (correction 5): `pact/genesis/stoa/*`, `pact/stoa-coin/new-coin.pact`, `pact/namespaces/ns-install.pact` are the real paths.
- Update keysets narrative to match the actual `pact/genesis/stoa/keysets.yaml` and `pact/genesis/stoa/ns.yaml` in the code repo (verify with the owner before rewriting — the genesis keyset story has evolved across several commits, e.g. `ec108ce` "regenerate genesis with multi-sig governance").
- Align described genesis time with the actual value in `src/Chainweb/Version/Stoa.hs` (currently `2026-02-23T18:00:00.000000`).

### `/docs/chainweb-node/NODE_LAUNCH_CHECKLIST.md`
- Replace references to `apply-config.sh` and `stoachain-config.yaml` with the real deployment entrypoints (`deploy.sh`, `run-stoa.sh`, `stoa-node.service`) — unless the project owner confirms those tools exist in a private repo and will be made public.
- Networks: single `--chainweb-version stoa` flag. No testnet/devnet flag variants.

### `/docs/chainweb-node/PACT4_REMOVAL.md`
- The whole premise of this file is wrong (correction 4). Pact 4 was **not** removed.
- Options, in preference order:
  1. **Rewrite** as "Pact 4 removal — attempted, abandoned" with an honest retrospective (why it was attempted, why it didn't land, and what that means for the codebase going forward).
  2. **Delete** and fold a one-paragraph note into the main README.
  3. **Preserve as historical record** but add a bold "SUPERSEDED — Pact 4 was not in fact removed" banner at the top.
- Ask the project owner which they prefer.

### `/docs/pact-5/README.md` and `/docs/pact-5/chain-data.md`
- Verify the `global-supply-register` and `external-fpa` `chain-data` extensions are actually implemented in the live StoaChain's Pact 5.4 build at `D:\_Claude\StoaChain`. If yes, keep. If they are aspirational, either remove or mark as planned. Grep the code repo for `global-supply-register` and `external-fpa` to decide.

---

## Items that must be confirmed with the project owner before rewriting

These are claims in the current docs that the StoaChain code repo neither confirms nor cleanly refutes; a docs update should not simply delete them without a check, nor keep them without verification.

1. **URSTOA and URSTOA-Vault.** Do they exist on-chain today on StoaChain? The explorer module list provided did not include them. If they exist, under what module names? If they don't, remove the entire URSTOA / URSTOA-Vault / RPS narrative from all docs.
2. **"7 Stoa Masters" governance.** Is the live `stoa-ns` namespace actually governed by 7 keysets with `enforce-one`? Genesis YAMLs in the code repo show a simpler single-keyset pattern. If it's simpler than 7-of-any-1, correct the governance section accordingly.
3. **`stoachain-config.yaml` / `apply-config.sh`.** Does this centralized-configuration tooling exist anywhere, or is it aspirational? It is not in the code repo at `D:\_Claude\StoaChain`.
4. **"AncientStoa" and "AncientPact" as project names.** The public-facing repo owner used `StoaChain/stoa-chain` (per `deploy.sh`) and there is no public evidence of repos literally named "AncientStoa" / "AncientPact". Confirm whether these names are still used, or should be replaced throughout the docs.
5. **Gradual minimum gas price ramp** (correction 7). Confirmed by the project owner as *planned but not yet shipped*. Before publishing specifics, confirm the intended starting floor, per-ramp increment, total ramp duration / final-floor cap, and whether the 3-hour cadence is still the target. Until then, describe it as a roadmap item only.
6. **Genesis supply numbers (12M STOA, 1M URSTOA, 480M ceiling).** Docs currently say these are placeholders. What are the real numbers as of the live 2026-02-23 genesis? Pull from `pact/genesis/stoa/*.yaml` in the code repo or ask the owner.

When in doubt: **prefer to delete an unverifiable claim** rather than leave an incorrect one in the public-facing docs.

---

## What to preserve

Not everything is wrong. The following should be kept as-is:

- The **Chainweb / Petersen graph / 10-chain** architecture description (correction 2).
- The **Yang Emission / Yin Earnings** vocabulary (project terminology).
- The **30-second block time**, **braided PoW** framing.
- The **non-managed TRANSFER capability** design rationale (correction 5 preserves this intent, just with corrected module names).
- The **"Development Method" / time-dilation** section at the bottom of both READMEs — it is intentional flavor text per the StoaChainDocs CLAUDE.md and should be preserved verbatim.
- The **ASCII box diagram** style for architecture and flow diagrams.
- The **"Aspect | Kadena | StoaChain"** comparison-table column order.
- The **MIT license** framing and the **Kadena acknowledgment**.

---

## One-line summary to hand to the next Claude

> "StoaChainDocs describes a planned 'AncientStoa' rewrite that never landed. The live StoaChain is a **minimal software fork** of Kadena chainweb-node (not a chain fork — its own genesis) with a single `stoa` network (code 0x0000_000A, 10 chains, Petersen, nodes at `node1.stoachain.com` + `node2.stoachain.com`), 1.6M/2M block gas, Pact 5.4 execution on-chain over a codebase that still carries Pact 4 infrastructure, a retained-but-ignored emission CSV, a custom `stoa-ns` module namespace whose live modules have drifted from their genesis sources, and a planned-but-not-shipped periodic gas-price-floor ramp (every 3 h). Use `assets/StoaLogo.png` for branding. Fix the docs to say those things. Confirm the URSTOA-Vault / 7-Masters / apply-config.sh / dynamic-gas-price items with the project owner before either keeping or removing them."
