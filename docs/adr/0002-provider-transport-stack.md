# Provider transport stack

The Rust port keeps `agente-types`, `agente-core`, and `agente-tui` synchronous and tokio-free while `agente-provider` owns one long-lived multi-thread tokio runtime and a reqwest 0.13 client on rustls with the platform verifier: `Client::stream` returns an opaque pull-based `EventStream` implemented over a bounded `mpsc(1)`, and cancellation crosses the boundary through a runtime-neutral `CancelToken`. Go's transport behavior is replicated explicitly rather than inherited from reqwest defaults — the `httpproxy` matcher is ported and wired through `Proxy::custom` (no `ALL_PROXY`, no OS proxy settings), only gzip is negotiated, and retry and SSE stay hand-rolled — so the parity contract's E-locked surfaces survive the port.

Status: accepted

## Considered options

- **Thread-per-stream with a current-thread runtime and `std::sync::mpsc::sync_channel(0)`** (exact Go backpressure): rejected; one thread and runtime per stream, no shared connection pool. The chosen `mpsc(1)` delivery adds at most one event of lookahead, which is not observable in goldens (they assert order, not scheduling).
- **ureq (blocking, no runtime)**: rejected; an in-flight body read cannot be cancelled, proxies and TLS diverge by default (webpki-roots, `ALL_PROXY`), and HTTP/2 is unavailable.
- **`tokio_util::CancellationToken` as the port's cancellation type**: rejected; it leaks tokio into `agente-types`. A neutral `CancelToken` is adapted inside the provider instead.
- **reqwest's default proxy matcher**: rejected; it honors `ALL_PROXY` and OS proxy settings and uses a different `NO_PROXY` grammar. Parity contract D6 requires gaps be closed by explicit configuration.
- **Off-the-shelf SSE parsers and reqwest's built-in retry**: rejected; no crate reproduces the Go parser's byte-level semantics (UTF-8 leniency, 10 MiB cap, empty-data events), and reqwest's retry classifier and budget differ from the Go loop.
- **hyper server for the OAuth and API-key login callbacks**: rejected in favor of a minimal blocking `std::net::TcpListener` handler; no server runtime in `agente-provider-auth`.

## Consequences

- reqwest default features drop `system-proxy` (dead once `Proxy::custom` is attached); `default-tls`, `charset`, `http2`, and `gzip` are enabled.
- The ported proxy matcher adds an `idna` dependency and Go's `httpproxy` test table as fixtures.
- `agente-types` gains `CancelToken` and the `EventStream` port; tokio remains confined to `agente-provider` (and auth HTTP), per ADR 0001.
- Exact `Client`/`EventStream` signatures, retry and cancellation propagation, and the determinism `Transport` seam stay with tickets #9, #10, and #13.
- Verified reqwest/tokio facts: `.scratch/rust-port/research/reqwest-tokio-facts.md`; decision recorded on GitHub issue #6.
