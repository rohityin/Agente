# reqwest 0.13.5 / tokio 1.53 fact sheet (ticket #6)

Date: 2026-09-25. Verified against pinned sources: reqwest `v0.13.5`
(`github.com/seanmonstar/reqwest`), hyper-util `v0.1.21`, tower-http `tower-http-0.6.11`,
tokio `tokio-1.53.0`, tokio-util `0.7.19`, Go `go1.25.0` vendored
`golang.org/x/net v0.41.0` (`src/vendor/golang.org/x/net/http/httpproxy/proxy.go`).
This sheet backs the stack decisions on GitHub issue #6.

## 1. reqwest proxy control (0.13.5)

- `ClientBuilder::no_proxy()` **clears all proxies and sets `auto_sys_proxy=false`**
  (client.rs:1436-1438) — it does not install NO_PROXY filtering.
- `ClientBuilder::proxy(Proxy)` **also disables automatic system proxy**
  (client.rs:1423-1425). `auto_sys_proxy` is private, defaults `true` (client.rs:309-310),
  and is only ever turned off.
- `Proxy::http<U: IntoProxy>(U) -> crate::Result<Proxy>` / `https` / `all`
  (proxy.rs:191/209/230). Per-scheme filtering works: only `Proxy::https` means
  http requests go direct.
- `Proxy::custom` exists (proxy.rs:255-263):
  `pub fn custom<F, U: IntoProxy>(fun: F) -> Proxy where F: Fn(&Url) -> Option<U> + Send + Sync + 'static`.
  - Invoked **per request** with `scheme://host[:port]` (no path/query/userinfo; proxy.rs:776-786).
  - Return `None` ⇒ no proxy. Returned value must itself parse as a proxy URL
    (it is re-checked through a hyper-util matcher; proxy.rs:788-794).
  - **Bypasses the env/system matcher entirely** and sets `no_proxy: None`.
    An attached `.no_proxy(...)` is ignored for custom matchers (`Custom::call` never
    reads it; proxy.rs:775-796) — the closure is fully in control of bypass.
  - The closure is `Send + Sync + 'static`; adding it disables system proxy discovery.
- `NO_PROXY` applies only to the env/system matcher: `NoProxy::from_env()` reads
  `NO_PROXY`/`no_proxy` (proxy.rs:476) and must be attached explicitly to a manual proxy
  (`Proxy::no_proxy`, proxy.rs:361).
- Auto system matcher (`auto_sys_proxy=true`) = env first (`ALL_PROXY` > `HTTPS_PROXY`/`HTTP_PROXY`,
  CGI guard drops all proxies) then OS config on Windows/macOS.
- hyper-util `NoProxy` grammar (0.1.21): comma-separated; `*` only as an exact entry;
  domain entries match domain + subdomains (leading dot optional); IPs and CIDR supported;
  **`host:port` is not supported**; case-insensitive in 0.1.21 (was case-sensitive in 0.1.12).

## 2. Go 1.25 `httpproxy` matcher (the parity target)

- Single file, 373 lines; `ProxyFromEnvironment` caches the decision in a `sync.Once`
  (`net/http/transport.go:961-966`) — env read once per process.
- Env vars only `HTTP_PROXY`/`http_proxy`, `HTTPS_PROXY`/`https_proxy`, `NO_PROXY`/`no_proxy`;
  upper-case checked first; no `ALL_PROXY`; no OS settings.
- CGI guard: `REQUEST_METHOD` set + an http request + `HTTP_PROXY` present ⇒ explicit error
  (`refusing to use HTTP_PROXY value in CGI environment`); `HTTPS_PROXY` is not blocked.
- Scheme dispatch: https→`HTTPS_PROXY` only; http→`HTTP_PROXY` only; no cross-scheme fallback.
- Always bypassed: `localhost` and loopback IPs, regardless of `NO_PROXY`.
- `NO_PROXY` parsing: `*` matches all and stops; CIDR; `host:port` / `[v6]:port`;
  bare IP; domain (leading dot or `*.` stripped) matches domain + subdomains; IDNA ToASCII
  applied to entries and hosts. IP rules only match IP-literal hosts, domain rules only names;
  entries without a port match any port.
- Faithful Rust port ≈250-400 LOC plus an IDNA dependency.

## 3. reqwest compression (0.13.5)

- Features `gzip`, `brotli`, `zstd`, `deflate` (Cargo.toml:59-62), each defaulting on when
  compiled in (client.rs:124-136). All negotiation/decompression goes through
  `tower_http::decompression::Decompression`.
- `Accept-Encoding` is added only when the request has none (tower-http service.rs:115-120);
  value is composed from enabled codecs.
- Transparent decode removes `Content-Encoding` and `Content-Length` (future.rs:70-71),
  matching Go (`transport.go:2378-2384`). `Response::chunk()` and `bytes_stream()` see
  decompressed bytes.
- Caveat: reqwest docs promise no automatic `Accept-Encoding` on `Range` requests, but
  tower-http 0.6.8 checks only `Accept-Encoding`; verify at implementation.

## 4. tokio 1.53 bridging

- `tokio::sync::mpsc::channel` requires capacity ≥ 1 (bounded.rs:158-160); no rendezvous
  variant. `Receiver::blocking_recv()` is the documented async→sync consumer path
  (bounded.rs:379-381) and **panics inside an async context** (`block_on` guard).
  `blocking_send` has the same constraint.
- `CancellationToken`: `cancel()` is synchronous, callable from any thread, cancels parent +
  children; child cancellation does not touch the parent; cloneable/Send/Sync. Cancelling a
  child ≠ cancelling the parent; `drop_guard()`/`run_until_cancelled()` exist.
- A neutral `CancelToken` (agente-types) can be adapted to `select!` in the provider; no
  tokio type needs to cross the crate boundary.

## 5. Client/runtime binding

- reqwest captures no tokio `Handle` at build: it builds a hyper-util legacy client with
  `TokioExecutor`, which calls `tokio::spawn` at request time (client.rs:941-942).
- Requirements: requests must be polled inside a tokio runtime with timers; default DNS uses
  `spawn_blocking`; pool/connection tasks are spawned into the runtime running the request.
- Consequence: one long-lived multi-thread runtime owning the shared `Client` keeps pooled
  connections alive; a short-lived or paused runtime can stall on its own pooled sockets
  (reqwest issue #2501 — maintainer discussion, not formal docs).

## Unverified

- Whether the `Range` doc claim (no auto `Accept-Encoding`) holds in 0.13.5.
- hyper-util NO_PROXY case-sensitivity drift between 0.1.12 and the resolved patch version.
- `idna` profile equivalence with Go's `idna.Lookup`.
- Byte-level brotli/zstd behavior (not adopted; gzip only).
