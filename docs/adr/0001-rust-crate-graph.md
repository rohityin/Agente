# Rust crate graph

The Agente Go implementation is the frozen oracle for a full-parity Rust port, and crate boundaries are hard to change later. We mirror the Go package tree one-to-one into a root Cargo workspace, one crate per substantive package (`crates/agente-*`), with one added crate, `agente-types`, owning the neutral domain model (messages, content blocks, usage, stream events, the `Client` port) plus the determinism seams. This breaks Go's `core -> provider` and `tui -> provider` edges so that "provider quirks must not leak into core" is structural rather than a convention. `agente-core` stays synchronous; tokio lives inside `agente-provider`, mirroring Go's blocking channel-based agent loop.

Status: accepted

## Considered options

- **Coarse grouping (~7 crates)**: fewer boundaries but much larger crates, weaker compile firewalls, and a less legible mapping to the Go packages being ported one at a time.
- **Mirroring Go's entanglement**: keep `Message`/`Client` in `provider` and let `core`/`tui` depend on it; rejected because the ownership map requires provider isolation and terminal code should not know provider types.
- **Inverting ownership into `core`**: `provider` would depend on `core`; rejected because the provider wire layer and catalog are independent of the agent loop, and `tui` would still need a shared model crate.
- **Async core**: rejected because Go's `Tool` and `Confirmer` contracts are synchronous; async would spread through every crate for no parity gain.

## Consequences

- `agente-types` is the only crate with no Go counterpart. The extension-author SDK (`agent/ext`) is deferred to ticket #11 under the reserved name `agente-ext`.
- Two seam-driven edges do not exist in Go: `agente-swarm -> agente-types` and `agente-provider-auth -> agente-types`.
- Goldens can inject `Clock`, `IdGen`, `Env`, `TerminalSize`, and `Transport` from `agente-types::seams` without touching production paths.
- The full member list, edge list, feature flags, and workspace conventions are recorded on GitHub issue #4.
