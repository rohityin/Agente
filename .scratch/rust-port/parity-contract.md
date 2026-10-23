# Executable parity contract — Rust port of Agente

Status: binding. Recorded from ticket [Define the executable parity contract](../issues/03-parity-contract.md) (2026-09-24).
Consumers: crate graph (04), JSON strategy (07), session/state (08), provider architecture (09), error/retry (10), extension host (11), RPC (12), verification harness (13), cutover (14), done-order (15), dialogs (16), input fallback (17). Session/state strategy (08) recorded as ADR `docs/adr/0004-session-state-compat.md`; fixture corpus manifest at `.scratch/rust-port/research/session-state-fixture-corpus.md`.

## How to read this contract

- **Oracle**: the frozen Go implementation on the developer machine is the oracle; it is local-only (never committed or pushed) and is deleted at cutover. Where code and docs disagree, **code wins**; shipped docs are rewritten to match code during the port.
- **Amendment rule**: every externally observable surface must have a row here. A surface discovered later is added by an amendment row (new table row or quirk entry) before it is ported; nothing ships outside the contract.
- **Parity bars**:
  - **E — exact**: bytes/streams must match. Normalization allowed only for declared volatile values (timestamps, random ids, pids, absolute paths, temp names, float formatting) and only in fixtures, never in production shape.
  - **S — schema-exact**: parse-equivalent. Same keys, types, values, fixed field order where Go's struct order is observable; key order elsewhere free. Both implementations can read the other's output.
  - **B — behavioral**: same observable behavior and UX; layout/wording/timing may differ within reason. Human text is *copied faithfully* (no cleanups) but is not byte-tested.
  - **D — divergence**: intentionally different; recorded in the divergence register.
- **Verification ladder** (per bar): E → captured-stream/byte goldens replayed by Rust tests with a normalizer; S → parse-and-compare goldens; B → behavior assertions (headless ANSI emulator, local HTTP fixture servers, exit-code checks); D → a test asserting the divergent behavior and a register entry.
- **Determinism seams (required from day one in Rust)**: injectable clock, id/RNG source, terminal size, environment, and HTTP transport so E/S goldens replay without time or randomness. Ticket 13 owns the concrete seam list.
- **Fixture policy**: goldens are generated once against the local Go oracle, committed to the repo, and replayed by Rust tests with CI staying Go-free. The manifest records a deterministic fingerprint of the oracle tree (sorted path + SHA-256 per file) instead of a commit; any change to the tree that alters the fingerprint regenerates the affected fixtures. Local differential probing against the Go binary is allowed while developing but never required in CI.

---

## 1. CLI and process surface

Anchors: `cmd/agente/main.go`, `packages/agent/{cli,args,config,build,lifecycle}.go`, `packages/agent/modes/{print,stream,json}.go`.

| Surface | Bar | Verification | Anchor / notes |
|---|---|---|---|
| Exit contract: 0 on success, 1 on error, `agente: <err>` on stderr; no code 2; consent denial exits 0 | E | exit-code fixtures | `main.go:37-40`; `agentefile.go:885-897` |
| `-h/--help`: stdout + exit 0; parse errors: full help to **stderr** then `agente: <err>`, exit 1 | E (routing) / B (text) | golden streams with text normalization | `args.go:328-331`, `cli.go:259-263` |
| `-v/--version`: `agente <v>` plus ` (sha7, date)` when commit set; `v` from Rust build metadata (D4) | E shape | string-shape fixture | `cli.go:268-271`, `main.go:24-36` |
| Flag set: names, aliases, defaults, arity — `-p/--print`, `--stream`, `--json`, `--rpc`, `--provider`, `--model`, `--api-key`, `--base-url`, `--insecure`, `--reasoning`, `--temperature`, `--system-prompt`, `--append-system-prompt`, `-c/--continue`, `-r/--resume`, `--session`, `--no-session`, `--tools`, `--no-tools`, `--cwd`, `--no-yolo`, `-y/--yes`, `-e/--ext`, `--no-ext`, `--no-skill`, `--no-context-files`, `--max-steps`, `--stats`, `--list-models`, internal `--swarm-agent`, deprecated no-op flags | E | argv→behavior fixtures | `args.go:138-326`; parse-error wording B |
| Mode dispatch: last mode flag wins; `rpc` normalized to `--rpc`; piped stdin prepends `--print`; subcommand router order bot→ext→update→sessions→agentefile | E | argv fixtures | `cli.go:228-300` |
| Prompt assembly: stdin trimmed, joined with positionals by `\n`; missing prompt error per mode | E (behavior) / B (text) | stdin fixtures | `cli.go:1739-1755` |
| Print mode: stdout = final assistant text + trailing `\n` only; all diagnostics to stderr | E framing | byte golden | `modes/print.go:61-66` |
| Stream mode: deltas to stdout, diagnostics to stderr, trailing newline guaranteed | E framing | byte golden | `modes/stream.go:61-113` |
| `--json` mode: one JSON object per line on stdout; fatal error appends `{"type":"error",...}`; pre-startup errors same shape | E (framing) / S (objects, amended ticket 07) | NDJSON shape goldens | `modes/json.go:13-34`, `cli.go:540-576` |
| `--stats` file: JSON field names/types | S (timings normalized) | schema fixture | `cli.go:389-416` |
| `--list-models`: dynamic-width table, `✓`, discovery-dependent | B | none | `cli.go:1769-1824` |
| `sessions prune`: flags `--dry-run/--all/-y/--older-than/--cwd`, constraints, summary line shape | E behavior, B text | `sessionscmd_test.go` patterns | `sessionscmd.go:66-112` |
| `ext` CLI: `list/doctor/logs/enable/disable/remove/install`; help to stderr; stdout/stderr split per subcommand | E routing, B text | fixtures | `extcmd.go:56-71,119-129` |
| `update`: `--check`, refuses `0.0.0`/`dev`, asset naming, `GITHUB_TOKEN` | B (update flow is fog) | none | `updatecmd.go:89-114` |
| `telegram-bot`/`tg`: `setup/status/reset/run/start/stop/logs`; pid/log paths; per-OS detach | E behavior, B text | botspec/status tests | `botcmd.go:35-204`, `botspec.go:11-49` |
| `agente run` (agentefile): peels `-y`, consent semantics, session root `sessions/agents/<name>` | E behavior | `agentefile_test.go` | `cli.go:1571-1576` |
| Env vars — home chain: `AGENTE_HOME` > `XDG_STATE_HOME/agente` > OS app-data > `~/.local/state/agente` (ZOT_HOME dropped, D2) | E | `config_home_test.go` patterns | `config.go:161-185` |
| Env vars — child markers: `AI_AGENT=agente`, `AGENTE_AGENT=1`, clears/publishes `AGENTE_SESSION_ID/SESSION_FILE/PROVIDER/MODEL/REASONING_LEVEL` | E | env fixtures | `cli.go:217-222,1656-1683` |
| Env vars — UI/proxy/llama overrides: `AGENTE_FLAT_TOOLS`, `AGENTE_COMPACT_INPUT`, `HTTP(S)_PROXY` apply rules, `LLAMA_BASE_URL/API_KEY` | E | `config_proxy_test.go` | `config.go:200-336` |
| Env vars — provider credentials: full matrix incl. generic `{PROVIDER_UPPER_SNAKE}_API_KEY` | E names + precedence | matrix fixture | `config.go:382-566` |
| Signals: SIGTERM/SIGHUP flush session then exit 0; SIGINT left to TUI | E | signal fixture | `cli.go:1533-1555` |
| Startup repairs/warnings on stderr (`config.json: …`, skill diagnostics, extension load lines) | B (text), E (stream) | behavior tests | `modelsync.go:88-137`, `cli.go:342-345` |

## 2. Wire protocols

Anchors: `packages/agent/rpc.go`, `modes/json.go`, `packages/core/events.go`, `packages/agent/extproto/extproto.go`, `extensions/*`, `docs/rpc.md`, `docs/extensions.md`.

| Surface | Bar | Verification | Anchor / notes |
|---|---|---|---|
| RPC framing: NDJSON both ways, blank lines skipped, 16 MiB max line, malformed JSON → `{type:"response",command:"",success:false,error:"malformed json: …"}` and continue | E (behavior) / S (frames, amended ticket 07; error text D11) | behavior fixtures + shape goldens | `rpc.go:178-192,431-474` |
| Response envelope `{type,command,success,id?,data?,error?}`; events are raw payloads, never `id` | S (amended ticket 07) | schema goldens | `rpc.go:433-473` |
| Auth: `AGENTE_RPC_TOKEN` hello handshake; **quirk** pre-auth non-hello continues, wrong token exits; hello data `{protocol_version:1,version,provider,model}` | E (behavior) / S (hello data, amended ticket 07) | quirk fixture | `rpc.go:171-244` |
| Methods `prompt/abort/compact/get_state/get_messages/clear/set_model/set_reasoning/get_models/ping`; request/response schemas; busy responses for mutations; abort accepted while busy | S (amended ticket 07) | schema goldens + `rpc_*_test.go` | `rpc.go:245-429` |
| `compact`: `{started:true}`, success `compact_done`, failure `error` only, **no `done` ever** (quirk) | E (behavior) / S (data, amended ticket 07) | quirk fixture | `rpc.go:381-401` |
| Lifecycle: one prompt at a time; stdin close waits in-flight turns; no cancellation on close | E | behavior fixture | `rpc.go:158-169` |
| `--json`/RPC event schema: every `EventToJSON` type incl. `text_delta`, `tool_use_start/args/end`, `tool_call`, `tool_progress`, `tool_result`, `turn_end{stop,error?}`, `usage{…,cumulative{…}}`, content blocks | S (amended ticket 07) | NDJSON parse goldens | `modes/json.go:39-129` |
| Quirk: `user_prompt_submit`, `pre_compact`, `post_compact` serialize with **no payload** | S (amended ticket 07) | quirk fixture | `modes/json.go` gaps |
| Extension frames: `hello/hello_ack/register_command/register_tool/ready/subscribe/event_intercept_response/tool_result/command_response/open_panel/panel_render/panel_close/notify/clear_notes/submit/submit_slash/shutdown_ack`, host→ext `command_invoked/tool_call/tool_cancel/event/event_intercept/panel_key/panel_close/shutdown` | E schema (field declaration order) | frame goldens | `extproto.go:29-270` |
| Handshake: first frame must be `hello`; manifest name wins; capability `tool_cancel` exact string; `hello_ack` synchronous; **quirk: no hello deadline** | E | behavior fixtures | `manager.go:763-802` |
| Readiness: `ready` frame; 250 ms idle auto-ready shim; 3 s `WaitForReady` deadline records diagnostic and proceeds | E behavior | timing fixtures | `manager.go:580-678,845-879` |
| Intercept chain: 5 s timeout, fail-open (allow) on timeout/cancel/malformed + `ext_notify` warning; `before_agent_start` prompt rules incl. 1 MiB cap; blocked-write closes pipe | E behavior, B timings | `before_start_test.go`, `intercept_test.go` | `events.go:92,216-258` |
| Intercept ordering | **D1** deterministic load order | register + ordering test | Go: map order `events.go:180-193` |
| Outbound queue: 256 frames / 16 MiB → disconnect, never silent drop; 5 s bounded writes; overflow test | E behavior | `lifecycle_test.go` | `outbound.go:12-101` |
| Tool invocation: 60 s default deadline, `interactive` unbounded, headless reject, timeout `timed_out`, cancel `cancelled` + `tool_cancel` capability-gated, late results dropped | E behavior | `interactive_tool_test.go` | `manager.go:1247-1311` |
| Registration precedence: built-ins win; first-come; case-insensitive commands, case-sensitive tools; conflicts = diagnostics | E behavior | `manager_test.go` | `build.go:85-100`, `manager.go:922-963` |
| Cleanup escalation: `session_end`, `shutdown` frame bounded by grace (min 100 ms), stdin close, SIGTERM, 1 s, Kill; Unix `Setpgid`; Windows kill-only | E behavior | `lifecycle_test.go` | `manager.go:1406-1461` |
| Extension stdout discipline: stdout protocol-only; stderr → `$AGENTE_HOME/logs/ext-<name>.log`; malformed frames → diagnostics | E | behavior fixture | `manager.go:687-697,912-916` |
| Extension lifecycle events payload: `result` ≤256 KiB / ≤128 blocks, UTF-8 truncation, `truncated:true`; sequence counter per manager lifetime | E semantics | `lifecycle_test.go` | `packages/agent/lifecycle.go:31-69` |
| Dead frame `panel_resize` declared but never sent (quirk) | E | quirk fixture | `extproto.go:256-261` |

## 3. Persisted state

Anchors: `packages/core/session*.go`, `provider/provider.go`, `provider/auth/store.go`, `agent/config.go`, `agent/settings_store.go`, `agent/skill_pins.go`, `provider/{models,usermodels,cache}.go`.

All state rows: **S + Go serialization conventions** — Rust output must look like Go's (indent, trailing newline presence, compact JSONL rows, field order from Go structs) and both implementations must load each other's files. Volatile values (ids, timestamps, paths, pids, float formatting) normalized in goldens. Amended ticket 07: emission is parse-equivalent; serde_json's string escaping, float text (e.g. integral floats as `1.0`), and map key order may differ; unknown keys on write are governed by D10. Amended ticket 08: state goldens verify by **parse goldens** (Go-authored files replayed with semantic equality) plus **byte-locked micro-goldens** for serde-free quirk rows (rename `title,type`, usage double-write, six auth objects/zero `expiry`, skill-pins trailing newline) plus behavior tests (repair/prune/list/append); no full-file byte comparison. Rust→Go readability is by the S bar plus dev-only differential probing, never CI. Session determinism seams: `Clock` (`meta.started`, filename `<UTCstamp>`), `IdGen` (session id, filename uuid8). Corpus manifest: `.scratch/rust-port/research/session-state-fixture-corpus.md`; bytes generated at ticket 13.

| Surface | Bar | Verification | Anchor / notes |
|---|---|---|---|
| `$AGENTE_HOME` layout: `config.json`, `auth.json`, `sessions/`, `logs/`, `models-cache.json`, `models.json`, `skill-pins.json`, `update-check.json`, `docs/`, `extensions/`, `themes/`, `swarm/`, `agents/`, `bot.json`, `bot.pid` | E paths | path fixtures | `config.go:193-244`, `modelsync.go:16-21` |
| Sessions: JSONL rows `meta/message/compaction/usage/rename`; meta fields `id,cwd,model,provider,started,version,title?,parent?,fork_point?,hide_from_sessions?`; message `{role,content[],time,meta?,added_tool_names?}`; usage `{input_tokens,…,cost_usd}` + `cumulative`; compaction `{messages[]}` | S (row shapes) | parse goldens; `core_test.go`, `session_portable_test.go` | `session.go:42-78`, `provider.go:124-174` |
| Tolerant read: malformed/unknown rows skipped, last meta wins, compaction replaces transcript, empty-content messages dropped, orphan tool pairs repaired with stub errors, CRLF tolerated | E behavior | repair goldens; `session_repair_test.go` | `session.go:244-376` |
| Hydration by field presence: reasoning → tool_call → tool_result → image → else text; unknown fields dropped on re-serialize | E behavior | `core_test.go` | `session.go:812-900` |
| Quirks: filename `<UTCstamp>-<uuid8>.jsonl` unrelated to `meta.id`; usage rows write cumulative into both `usage` and `cumulative`; rename row key order `title,type` | E | quirk fixtures | `session.go:98-137,406-416`, `cli.go:1000-1006` |
| Append/rewrite rules: append-only sessions (`O_APPEND`), fresh file deleted when no messages, prune-empty on open, list sort by mtime desc, export/import stream non-meta rows verbatim | E behavior | `session_prune_test.go` | `session.go:280,611-650,722-777`, `session_portable.go` |
| `auth.json`: top-level `anthropic,openai,kimi,google,deepseek,github_copilot,additional_api_key_creds?`; per-provider `api_key?/api_key_command?/base_url?/oauth?`; **quirk** all six objects always emitted, zero `expiry` emitted | S | parse goldens; `auth_test.go` | `auth/store.go:17-62` |
| `auth.json` write: temp+rename, 0600; mutation rules (SetAPIKey clears command/OAuth except OpenAI; etc.); unknown fields dropped on save (Rust preserves them, D10) | E behavior / D10 | `store_additional_test.go` | `auth/store.go:155-315` |
| `config.json`: 31 top-level keys (not ~45); tri-state `*bool`/`*int` semantics; always-emitted `provider,model,reasoning,theme`; precedence flag > config > default | S | parse goldens; `settings_store_test.go` | `config.go:26-155` |
| `config.json` write: `MarshalIndent` 2-space, no trailing newline, **quirk** re-reads only `keymap` on save, drops unknown keys (Rust preserves them, D10), direct (non-atomic) write | E behavior / D10 | quirk fixture | `config.go:263-277` |
| User `models.json`: camelCase schema, read-only, tolerant load (whole-file reject on parse error — doc claim per-block is wrong), precedence user > live > catalog, numeric zero fields don't override | S | `usermodels_test.go` | `usermodels.go:33-80,124-324` |
| `models-cache.json`: `{fetched_at, models[]}` with **PascalCase model keys (no JSON tags)**; 6 h TTL; tmp+rename | S | parse golden (Go has none; Rust must add) | `cache.go:11-56` |
| `skill-pins.json`: `{global[],projects{cwd[]}}` sorted, tmp `CreateTemp` 0600 + rename, trailing newline; corrupt file refused with `invalid skill-pins.json`, never overwritten; unknown keys preserved on save (D10) | S + E behavior / D10 | `skill_pins_test.go` | `skill_pins.go:54-156` |
| Other writers: `update-check.json`, `bot.json` (0600), `bot.pid`, `logs/bot.log`, swarm `meta.json`/`events.jsonl`/`session.json`, consent receipts `{digest,name,version}` 0600, `kimi-cli-fallback-disabled` content `disabled\n`, gemini images into cwd | S + E behavior | per-package fixtures | see ticket 05/7/… refs; swarm section |
| Atomicity policy: tmp+rename where Go has it, fixed `.tmp` races preserved; **no fsync, no locks** — never stronger than Go | E behavior | race fixtures | `store.go:301-315`, `skill_pins.go` |
| File permissions: sessions 0644/0755, auth 0600, skill-pins 0600, config 0644; advisory on Windows | E (POSIX) | mode assertions skipped on Windows in Go too | `session.go:95-121` |

## 4. Provider wire

Anchors: `packages/provider/*.go`, `packages/provider/auth/*.go`.

| Surface | Bar | Verification | Anchor / notes |
|---|---|---|---|
| SSE reader: 10 MiB lines, blank-line dispatch, `:` comments skipped, multi-line `data:` joined `\n`, one leading space stripped, `[DONE]` | E behavior | fixture server + parser goldens | `sse.go:17-58` |
| Retry: 3 total attempts, 250/750 ms backoff; transient statuses 429/500/502/503/504/524/529; connect-error substring set; terminal quota needles; `retry-after-ms` > `Retry-After`, 60 s cap error; body cap 4 KiB | E values, classification by **outcome corpus** (D7) | retry fixture corpus; `retry_test.go` | `retry.go:21-251` |
| Exhausted retries re-synthesized as `http <code>: <body>` shape | E | golden errors | `retry.go:98-152` |
| Transport defaults: timeout 0 on inference clients, discovery/TLS timings; `insecure` clone semantics; proxy via env only (D6) | B + documented diffs | differential probe | `httpclient.go:12-55` |
| OpenAI Chat Completions: URL join rules, headers, body keys incl. exactly-one `max_tokens`/`max_completion_tokens`, clamp formula, message part shapes, DeepSeek/Kimi quirks, OpenRouter server tools | E (headers/URLs/string values), S (schema) | local fixture server; `openai_*_test.go` | `openai.go:30-35,171-218,288-409,585-590` |
| Codex/Responses: subscription endpoint, identity headers `originator/user-agent` byte-exact, session headers (values random), body/input item shapes, 400 `prompt_cache_retention` recovery, reasoning-effort map | E identity + behavior; S body | `openai_codex_test.go` | `openai_codex.go:38,71,154-166,388-461` |
| Anthropic: `/v1/messages`, API-key headers, OAuth identity header set byte-exact, exact Claude Code system string, tool-name remap, cache_control breakpoints, thinking/adaptive rules, block shapes, stream events | E identity; S body; E behavior | `provider_test.go`, `anthropic_image_test.go` | `anthropic.go:23-26,179-189,253-333,555-565` |
| Gemini/Vertex: URL + `x-goog-api-key` (never `?key=`), thought/thinking maps, stream chunks, generated image files into cwd; Vertex URL rewrite + JWT/refresh exchange + ADC paths | E URLs/headers; S body; E behavior | `gemini_test.go`, `google_vertex_test.go` | `gemini.go:484-498`, `google_vertex.go:199-333` |
| Bedrock: converse-stream URL, auth precedence, body incl. cachePoint and `_placeholder` stub, model-ID geo rewrite, AWS event-stream binary framing, SigV4; **quirk: no retry loop** | E | `amazon_bedrock_test.go` | `amazon_bedrock.go:88-136,481-572,805-1066` |
| Azure: Responses flavor URL/headers; legacy Chat Completions deployment rewrite; **quirk legacy client name `azure-openai-responses`** | E | `azure_openai_test.go` | `azure_openai.go:34-224` |
| Copilot: token exchange + `proxy-ep` parse, identity headers byte-exact, dynamic `X-Initiator`/`Openai-Intent`/`Copilot-Vision-Request`, per-family routing, discovery filter, device login | E | `github_copilot_wire_test.go` | `github_copilot*.go:40-249` |
| Auth flows: Anthropic/OpenAI/Kimi/xAI OAuth endpoints and params byte-exact, callback ports/paths, login web server routes, probes per provider, command-backed keys (128 s, 64 KiB, single-flight) | E | `auth_test.go`, `probe.go` fixtures | `oauth.go`, `callback.go`, `server.go`, `probe.go`, `api_key_command.go` |
| Discovery: endpoints, filters, page params, OpenRouter pricing math, custom/LM Studio/llama.cpp endpoints, HF search | E URLs/params; S parse | `discover*_test.go`, `lmstudio_test.go` | `discover.go`, `discover_custom.go`, `llamacpp.go` |
| Identity-impersonation strings are version constants that change with upstream; fixtures pin current values | E | frozen fixtures | `anthropic.go:24`, `openai_codex.go:71,403` |

## 5. TUI and terminal

Anchors: `packages/tui/*`, `packages/agent/modes/interactive.go`.

| Surface | Bar | Verification | Anchor / notes |
|---|---|---|---|
| Protocol escapes: synchronized-output fencing per paint, VS Code 3J suppression rules, row protocol (`MoveTo`+`0m`+`2K`), cursor hide/show, only DECSTBM reset `\x1b[r`, alt-screen enter/exit counts, keyboard enhancement enable/disable, kitty image delete, OSC 7 (RFC3986 encoding, Windows `/C:/`), OSC 12/cursor shape, exit resets | E | headless ANSI goldens | `render.go:350-563`, `terminal.go:87-192` |
| Idle paint emits zero bytes | E | render test | `render.go:521-523` |
| Input: raw mode, 50 ms Esc peek (Unix) / blocking Windows fallback (quirk), bracketed paste incl. end marker, kitty CSI-u, xterm modifyOtherKeys (ticket 17 decides), raw control bytes, SS3/Alt, UTF-8→U+FFFD, Ctrl+V | E byte-level | input corpora | `input.go:76-453`, `resize_windows.go:20-26` |
| Resize: SIGWINCH fan-out, size fallback 80×24, full purge on resize; **quirk Windows stale renderer dimensions** | E behavior | spike harness | `resize_unix.go:14-24`, `interactive.go:807-817` |
| Pacing: 16 ms redraw throttle, 120 ms animation tick, 6 runes/16 ms pacer, 256-key drain | B (behavioral, constants frozen) | behavior tests | `interactive.go:856-1025,7581-7639` |
| Markdown subset + GFM tables + width math | B | markdown goldens (ANSI-aware) | `markdown.go:23-435` |
| Syntax highlighting: per-token terminal256 SGR output for a golden corpus; monokai base + 18 overrides; background stripping | B (corpus-pinned) | snippet goldens | `highlight.go:23-338` |
| Partial JSON extractors | E behavior | `partialjson_test.go` | `partialjson.go` |
| Images: protocol detection env rules, kitty header `a=T,f=100,C=1` + 4096 chunks + `m=` flags, iTerm2 OSC 1337 format (default off), footprint math (cell aspect clamp 0.5–5), tab cleanup | E (headers/format) / D8 (PNG bytes implementation-local; decode+dimensions bar) | image goldens | `image.go:39-226,242-302` |
| OSC 11 detection: query bytes, parse rules, `AGENTE_THEME` override, luma ≥ 0.5 → Light, fallback Dark | E | Rust must add tests (Go has none) | `detect_bg.go:26-154` |
| Clipboard: per-OS command lists/argument order, 5 s timeout, safety caps 32 MiB/25 MP/64 MiB, macOS temp-file scheme 0600/0700, Windows STA + exit 3 | E behavior | platform tests (Linux/Win pinned; darwin manual) | `clipboard_*.go` |
| View/tool rendering: tool boxes, diff gutters, bash result styling, collapse `(N more lines…)`, ShortArgs preference, paste placeholders `[pasted text #N …]`, file/dir paste forms | B (shape copied; not byte-tested) | view tests | `view.go:28-2299`, `editor.go:1193-1338` |
| Themes: strict JSON schema, color forms, layering/fallback rules, preferred-name resolution, reset-to-auto on missing | S/E behavior | `theme_loader_test.go` | `theme_loader.go:27-413` |

## 6. Platform, tools, confinement

Anchors: `packages/agent/tools/*`, `packages/ignore/gitignore.go`.

| Surface | Bar | Verification | Anchor / notes |
|---|---|---|---|
| Bash: shell resolution per OS, merged stdout/stderr, 4 KiB streaming, 50 KiB/2000-line caps, `WaitDelay=2s`, process-group SIGTERM→Kill (Unix) / Kill (Win), env `AGENTE_CHILD_SESSION=1`, result text shape + `$ / [exit N] / Took` | E behavior, B text | tool goldens | `bash.go:22-299` |
| PowerShell (Windows): binary resolution, exact args `-NoLogo -NoProfile -NonInteractive -ExecutionPolicy Bypass -OutputFormat Text -EncodedCommand <b64 UTF-16LE>`, `PS>` prefix, gating (jail denies; non-`ask` denies) | E | `powershell_test.go` | `powershell.go:23-76` |
| Read/write/edit/glob: caps and truncation markers, image MIME sniffing, NUL refusal, parent dir creation 0755/0644, edit exact-match validation + preserved BOM/CRLF + diff body without `---/+++`, glob matcher (`**`, `{}`, `.git` skip, nested .gitignore), 500-match truncation | E behavior (markers E) | `tools_test.go`, `glob_test.go` | `read.go`, `write.go`, `edit.go`, `glob.go` |
| Permissions: fail-closed empty scopes, symlink-resolved containment, bash modes `none/ask/allowlist`, allowlist rejections (backticks/newlines/`<>`/`$(`), typed `ToolPolicyError` | E behavior | `permissions_test.go` | `permissions.go:61-183` |
| Jail/sandbox: outside-root denial with `jailed:` text, `DisplayPath` rewrite, banned substrings, cd-target checks, shell-word scan; docs must keep saying "speed bump, not a security boundary" | E behavior, B text | `sandbox_test.go` | `sandbox.go:41-378` |
| OpenRouter server tools: 8 names, limit default 4, local execute stub text | E | `openrouter_server_test.go` | `openrouter_server.go:12-112` |
| Ignore matching: `!` negation last-wins, trailing `/` dir-only, leading `/` anchored, no `**`, per-directory stack override | E semantics | `gitignore_test.go` | `gitignore.go:13-197` |
| Process control per OS: daemon detach (setsid / DETACHED_PROCESS+NEW_GROUP), stop escalation, liveness probes | E behavior | platform tests/manual | `bot/process_*.go` |

## 7. Peripheral subsystems

Anchors: `packages/agent/skills/*`, `docs.go`, `agentefile.go`, `swarm/*`, `modes/{bot,telegram}/*`.

| Surface | Bar | Verification | Anchor / notes |
|---|---|---|---|
| Skills discovery order: ext bundles → project `.agente` → home → `AGENTE_AGENT_SKILLS` → project → global → Claude/agents dirs → built-ins; recursive walk, exact-case `SKILL.md` | E order | `skills_test.go` | `skills.go:152-422` |
| SKILL.md frontmatter subset: `name/description/disable-model-invocation/allowed-tools/permissions`; quoted-name strip; ignored unknown keys; no-frontmatter = whole body | E behavior | parser goldens | `skills.go:442-520` |
| Shadowing: first wins, `os.SameFile` skip, diagnostic contains `" shadowed: "` (filter depends on exact spacing) | E (diagnostic format quirk) | `skills_test.go` | `skills.go:123-178` |
| Built-ins: `write-agente-extension`, `write-agente-themes`; hidden from pickers | E set | `builtin_test` | `builtin.go:16-51` |
| Skill manifest in system prompt + `skill` tool schema and result framing | E strings (model-visible) | `skills.go:321-361`, `tool.go:50-98` | |
| Pins: resolve union global+project, `pinned skill %q is unavailable` warning, preload only on fresh sessions | E behavior | `pins_test.go` | `pins.go:16-53` |
| Docs install: 7 flat filenames, mkdir 0755, overwrite only when bytes differ, 0644 | E layout | `docs_test.go` | `docs.go:18-57` |
| Docs content | **D3** rewritten for Rust and corrected against code | divergence entry | doc drift list in ticket 03 |
| Agentefile: manifest schema/validation, safe names, pack canonical tar (zeroed times/uid/gid, sorted walk) + zstd, digest def, extraction caps (100 MiB archive, 64 MiB entry, 256 MiB total, path-traversal rejection), inspect/verify output lines, consent receipts, requirement checks, GitHub refs loader | S/E | `agentefile_test.go`; archive determinism Rust must add | `agentefile.go:31-129,332-837` |
| Agentefile digest bytes | **D5** implementation-local (zstd encoder version); stability required within Rust; semantic equality across Go/Rust | register | `agentefile.go:363-376` |
| Swarm: Unix-socket transport + path probing, line protocol `user/cancel/shutdown`, preemption, 200 ms dial retry → `ErrNotReady`, credential-over-stdin (`AGENTE_SWARM_CREDENTIAL_STDIN`, ≤1 MiB, never argv/env), child argv/env contract, event log format + 250 ms dedup, meta.json, resume/reload, orphan mirror | E behavior + S formats | swarm tests | `swarm/*.go`, `swarm_agent.go` |
| Swarm Windows degradation (no socket listener) | E quirk until a transport lands | skip tests on Windows | `socketpath.go:35-40` |
| Telegram: getUpdates params + webhook deletion, per-update `LastUpdateID` save, pairing, private-chat filter, prompt assembly incl. photo download, chunking 4000 runes, typing every 4 s, command set, bridge mirror prefixes (code hardcodes `""`), daemon pid/log, per-OS stop | E behavior + S state | `telegram/*_test.go` | `adapter.go`, `bridge.go`, `api.go`, `bot/runner.go` |
| Telegram status/help text | B | none | `bot/status.go` |

## 8. Human wording policy

Copy Go's exact sentences — help, errors, warnings, dialogs, notify, `ext doctor` — with **no cleanups or "fixes"** (user decision). Verification remains **B**: behavior and stream routing are tested, strings are not byte-locked. Machine-parseable lines listed as E above are the exception and must be exact.

## 9. Quirk register (reproduce; each needs a regression test)

1. `--json` emits payload-less `user_prompt_submit` / `pre_compact` / `post_compact` events.
2. RPC `compact` never emits `done`; failure emits only `error`.
3. RPC pre-auth non-hello commands continue instead of exiting (docs say otherwise); wrong token exits.
4. Usage rows write cumulative values into both `usage` and `cumulative`; loader derives deltas.
5. Session filename UUID is unrelated to `meta.id`.
6. `auth.json` always emits all six provider objects; zero `oauth.expiry` is emitted.
7. `SaveConfig` re-reads only `keymap` and writes non-atomically; Go drops unknown config keys (Rust preserves them, D10).
8. `models-cache.json` uses PascalCase keys (no JSON tags).
9. Azure legacy Chat Completions client reports name `azure-openai-responses`.
10. `panel_resize` frame declared but never sent; `submit_slash` undocumented.
11. Extension intercept order is Go-map random → superseded by D1 (deterministic load order).
12. `--no-yolo` prints a warning and is ignored in print/stream/json/rpc.
13. Unknown `--tools` names silently dropped; `--max-steps` accepts trailing junk (`3x`).
14. Help omissions (`minimal`/`maximum` reasoning values, deprecated flags, internal flags) and stream-routing inconsistencies between subcommands (`ext` help → stderr, `sessions` help → stdout).
15. `ext enable/disable` rewrites `extension.json` preserving unknown keys; file 0644. (Go rewrites numbers through float64; Rust keeps unknown values as raw fragments — D10.)
16. Swarm event log skips malformed lines and dedups identical adjacent events within 250 ms.
17. Session list sorts by filesystem mtime desc, not meta timestamp.
18. Windows: resize handler is a no-op and renderer dimensions go stale; bare-Esc resolves only after the next keystroke.
19. Extension hello read has no deadline (hung extension blocks discovery).
20. Doc-vs-code drift found during inventory (~15 spots, e.g. `docs/rpc.md` auth/compact claims, `docs/providers.md` Bedrock credential chain, `docs/skills.md` manifest/visibility) — code is oracle; docs get corrected as D3.

## 10. Divergence register

- **D1** — Extension intercept chain uses deterministic load order (Go: nondeterministic map order).
- **D2** — Legacy `ZOT_*` env aliases and `zotfile` names are neither read nor emitted (charting decision). Revisit only if extension compatibility testing finds dependents.
- **D3** — `$AGENTE_HOME/docs` content is rewritten for the Rust implementation and corrected against code; layout is exact.
- **D4** — Version/build metadata is Rust's own; the `agente <v>` output shape stays exact.
- **D5** — Agentefile archive digest bytes are implementation-local; semantic equality across implementations, stability within Rust.
- **D6** — Provider transport differences (proxy env honoring, connect timeouts, connection reuse) are behavioral; any parity gap must be closed by explicit configuration, recorded per ticket 06/10.
- **D7** — Retry classification is structural (Rust error types), verified against an outcome corpus; Go's error-string matching is not reproduced.
- **D8** — PNG encode bytes are implementation-local; decodability, dimensions, and format are the bar (kitty/iTerm2/clipboard paths).
- **D9** — No new features, no UX redesigns beyond the frozen Go behavior (charting decision).
- **D10** — Unknown keys are preserved on whole-file state rewrites (`config.json`, `auth.json`, `skill-pins.json`, `extension.json`) as raw JSON fragments **deep**: every typed state struct that round-trips from disk (config root, auth root, each `ProviderCreds`, each `OAuthToken`, pins entries) carries a `#[serde(flatten)] BTreeMap<String, RawValue>`; known fields emit in Go declaration order, unknown fragments append after in sorted key order (ADR 0004). Go drops unknown keys in config/auth/skill-pins and rewrites extension.json numbers through float64. Sessions keep Go behavior exactly: raw-copy paths preserve unknown fields, hydrated rewrites drop them. Rust-written files remain readable by Go.
- **D11** — JSON parse/type error text in RPC error frames comes from serde_json; the frame shape, keys, and the `malformed json: ` prefix are preserved, but Go's `encoding/json` error strings are not reproduced.
- **D12** — serde_json parse strictness where Go is lenient: invalid UTF-8 and lone surrogate escapes error (Go replaces with U+FFFD), duplicate keys error in derived-struct decoders (Go last-wins; hand-written map-based codecs still last-win), and nesting beyond serde_json's recursion limit errors (Go tolerates far deeper). Tolerant surfaces (provider SSE chunks, session lines, extension frames) keep Go's skip/continue behavior; their skip can differ only on these pathological inputs.

## 11. Fixture and freeze protocol

- The Go oracle is local-only: never committed, never pushed. The public repo carries the Rust port and its artifacts; after cutover the Go code is deleted.
- Before generation: fix `packages/agent/session_environment_test.go:52` (calls undefined `setAgenteSessionEnvironment`, so `go test ./agent` does not compile), then confirm `go test ./...` is green.
- Record a deterministic fingerprint of the local Go tree (sorted relative path + SHA-256 per file) in the fixture manifest. Fixture generation runs against that exact tree; it never depends on the oracle being present in the repo or in CI.
- Any change to the oracle tree that alters the fingerprint regenerates the affected fixtures.
- Rust tests replay committed goldens with the declared normalizer; CI is Go-free. Differential probing against a local Go binary is a dev-time option.
- Normalization list (fixtures only): RFC3339(Nano) timestamps and zero times, random ids (`rand.Text`, UUIDs, correlation counters, agent ids), pids, absolute/machine paths, temp/log filenames, float formatting (`cost_usd`, prices), durations, base64 payload sizes.
- Machine-JSON surfaces amended to S (ticket 07) are verified by parsing, not byte comparison; serde_json escaping, float text, and map key order are permitted to differ.
