# Pact 4 in the StoaChain node — retrospective

> **📖 Main Documentation**: See the [main README](../../README.md) for an overview of StoaChain.

> **⚠️ SUPERSEDES earlier drafts.** Earlier versions of this document presented Pact 4 removal as accomplished — "~3,300 lines removed", "Pact 5 exclusive", "shared types only". **That is not the current state of the node code.** Pact 4 extraction was *attempted* and *abandoned*. This page is preserved as a retrospective.

## Summary

The StoaChain node still carries Pact 4 infrastructure internally. The live chain runs Pact 5.4 regardless, because the Chainweb-level configuration routes all on-chain execution through the Pact 5 execution path. Both statements must appear together whenever this topic comes up in the docs:

- **Code state**: the node repo still contains `src/Chainweb/Pact4/`, the `Chainweb.Test.Pact4.*` test modules, and the `pact` source-repository-package pin in `cabal.project` (alongside the `pact-5` pin).
- **Live chain state**: all on-chain execution is Pact 5.4 (the final Pact release before Kadena LLC dissolved).

Describing the codebase as "Pact 5 exclusive" or "~3,300 lines of Pact 4 removed" is incorrect. Describing the live chain as "running Pact 5.4" is correct.

## Why the extraction was attempted

The original motivation was simplification: StoaChain runs exclusively on Pact 5, so in principle Pact 4 execution code is dead weight. A clean removal would have reduced the Haskell surface area by a few thousand lines and eliminated a whole tree of Pact 4 tests.

## Why the extraction was abandoned

Pact 4 and Pact 5 in `chainweb-node` share non-trivial code paths. In particular:

- Shared type machinery (`PactVersionT`, `SomeBlockM`, `RunnableBlock`, checkpointer plumbing) that the Pact 5 path depends on cannot be cleanly separated from the Pact 4 path without refactors that outstripped the available expertise.
- Shared transaction plumbing (parsing, validation, TTL/gas helpers) is referenced from both paths.
- Test infrastructure that now backs Pact 5 tests originally targeted Pact 4 and still carries those dependencies.

Extracting Pact 4 cleanly would have required non-trivial restructuring of `chainweb-node` itself, which was not feasible under the project's constraints.

## What this means for the node repo today

- `src/Chainweb/Pact4/` exists and compiles.
- `Chainweb.Test.Pact4.*` test modules exist.
- `cabal.project` pins both `pact` (Pact 4) and `pact-5` as source-repository-packages.
- The live chain routes all on-chain execution through the Pact 5 path. There is no supported path that executes Pact 4 code on the live chain by default.

## What this means for these docs

- Do **not** describe StoaChain as "Pact 5 exclusive from genesis" or "Pact-4-free".
- Do **not** claim a specific line count was removed.
- When describing on-chain behavior, say **Pact 5.4**.
- When describing codebase state, say the node **retains Pact 4 infrastructure internally**.
- The two statements belong together.

## Related Documentation

- **Emission System**: [`EMISSION_SYSTEM.md`](EMISSION_SYSTEM.md)
- **Genesis System**: [`GENESIS_SYSTEM.md`](GENESIS_SYSTEM.md)
- StoaChain GitBook: https://demiourgos-holdings-tm.gitbook.io/kadena-evolution

---

*Last updated: 2026-04-19*
