# State-file fixture corpus — session/state compatibility (ticket 08)

Binding corpus manifest for ticket 13 (golden-fixture verification harness). Decision context: parity contract §3 (state rows) and ADR `docs/adr/0004-session-state-compat.md`. Byte generation happens at ticket 13; the manifest pins scope, anchors, normalization, and verification mode so 13 can start without re-deriving it.

## Verification modes (from ADR 0004)

- **Parse golden (P)**: a Go-authored file replayed in Rust; Rust loads it and asserts semantic equality (messages, meta, usage, pins). Emission differences (serde escaping, float text, map key order) are permitted.
- **Micro byte-quirk (Q)**: a byte-exact assertion on a tiny row/file with no serde-divergent content (fixed string keys, no floats, no non-ASCII), freezing the on-disk shape.
- **Behavior (B)**: headless behavior assertions (repair output, prune selection, mtime ordering, append/delete semantics, mutation rules), no file bytes compared.
- **Rust→Go readability**: never in CI; dev-time differential probing against the local Go oracle only.

## Determinism seams (session module)

- `Clock`: `meta.started`, filename `<UTCstamp 20060102-150405>`, message `time`.
- `IdGen`: `meta.id`, filename `uuid8` (two distinct ids per file — quirk 5).
- File bytes that contain `cwd` use a fixture temp dir; `sha256(cwd)[:8]` → normalized in goldens (path machines differ).

## Corpus

### 1. Sessions JSONL (`agente-core`)

| Artifact | Mode | Anchor (Go) | Normalization | Content |
|---|---|---|---|---|
| session-meta-only | P + B | `session.go:133` | started, id, cwd | meta row only; must be prunable + deleted on fresh close |
| session-simple-messages | P | `core_test.go`, `session_portable_test.go` | times, ids | message rows with text + tool blocks, `added_tool_names`, message `meta` |
| session-hydration-precedence | P | `session.go:812-901` (hydrateMessageObject) | times, ids | one file per precedence branch: reasoning-only, tool-call `name && id`, tool-result `call_id`, image `mime_type`, text; plus mixed; plus degradation inputs (empty text block, text-only tool-result content, image-only nested) |
| session-tool-arguments-raw | P | `session.go:849-864` | — | `arguments` preserved byte-for-byte as raw passthrough (incl. non-JSON shapes Go tolerated) |
| session-compaction | P | `session.go:270-273,711-720` | times | two compaction rows where the later replaces transcript; old rows still on disk |
| session-usage-double-write | Q + P | `cli.go:1000-1007`, `session.go:735-740` | times, ids, cost_usd | usage row with identical `usage` and `cumulative` (quirk 4); delta derivation asserted via `SessionUsageDetail` semantics (`nonNegDelta`, cache/cost clamps) |
| session-rename-row | Q | `session.go:406-416` | — | `{"title":"…","type":"rename"}\n` — byte-locks sorted map key order |
| session-orphan-repair | P + B | `session_repair_test.go`, `session.go:313-376` | ids | orphan tool_call pairs stubbed with `tool call was aborted; no result recorded.`; stub merged into next tool row or synthetic row inserted |
| session-tolerated-rows | B | `session.go:140-159,253-274` | — | CRLF endings, unparseable line skipped, unknown `type` skipped, empty-content message dropped, last-meta-wins, blank lines |
| session-list-order | B | `session.go:660-693`, `session_prune_test.go` | — | mtime desc, path tiebreak; prune-empty sweep on open; fresh-file deletion on close |
| session-branch | P | `session_portable_test.go`, `session_portable.go:258-420` | ids, times | fork/reparent meta (`parent`, `fork_point`, `hide_from_sessions`), byte-copy pre-branch (no compaction), re-marshaled rows post-compaction, rename-title derivation from last user text |
| session-export-import | P | `session_portable_test.go`, `session_portable.go:35-242` | ids, times | meta rewritten (cwd stripped/rotated, fresh id/started/version), non-meta rows streamed verbatim; `.agentesession` naming + slug |
| session-path-scheme | B | `session.go:85-89` | — | `sessions/<16 hex>` = `hex(sha256(cwd)[:8])`; not 8 chars |
| session-append-rules | B | `session.go:280,747-763,765-777` | — | O_APPEND reopen, per-row flush, no in-place rewrite |

### 2. auth.json (`agente-provider-auth`)

| Artifact | Mode | Anchor (Go) | Normalization | Content |
|---|---|---|---|---|
| auth-full-schema | P | `auth_test.go`, `store.go:17-62` | tokens | all six provider objects + `additional_api_key_creds`; every field incl. `oauth` with zero `expiry` emitted |
| auth-six-objects-always | Q | `store.go:18-26` | tokens | empty provider objects still emitted (structs never "empty" to Go); `additional_api_key_creds` omitted when absent |
| auth-mutation-rules | B | `store_additional_test.go`, `store.go:155-299` | tokens | SetAPIKey clears command+OAuth except OpenAI; SetOAuth clears key/command except OpenAI; Clear/ClearAPIKey/ClearOAuth; additional-provider add/remove/empty-delete |
| auth-deep-d10 | P | D10 (ADR 0004) | tokens | unknown keys inside a `ProviderCreds`/`OAuthToken` survive a sibling mutation and rewrite sorted after known fields |
| auth-atomic-write | B | `store.go:301-315` | — | temp+rename, 0600, Go's fixed `.tmp` race semantics preserved (no unique temp, no fsync) |

### 3. config.json (`agente-agent`)

| Artifact | Mode | Anchor (Go) | Normalization | Content |
|---|---|---|---|---|
| config-schema | P | `settings_store_test.go`, `config.go:26-155` | — | 31 top-level keys; tri-state `*bool`/`*int`/`*float32` presence; always-emitted `provider/model/reasoning/theme`; `keymap`, `quick_model_shortcuts` maps/arrays |
| config-save-quirks | Q + B | `config.go:263-277` | — | `MarshalIndent` 2-space, no trailing newline; re-read only `keymap` on save; direct non-atomic write |
| config-deep-d10 | P | D10 (ADR 0004) | — | unknown top-level keys and nested map keys survive a settings change, sorted after known fields |
| config-precedence | B | `settings_store_test.go` | — | flag > config > default; `FlatToolRender`/`CompactUserInput` env overrides |

### 4. models.json / models-cache.json (`agente-provider`)

| Artifact | Mode | Anchor (Go) | Normalization | Content |
|---|---|---|---|---|
| usermodels-load | P | `usermodels_test.go`, `usermodels.go:33-80,124-324` | — | camelCase schema; whole-file reject on parse error; precedence user > live > catalog; numeric zero fields don't override |
| models-cache-pascal | P | `cache.go:11-56` ("Go has none; Rust must add") | fetched_at | PascalCase keys (no JSON tags); `{fetched_at, models[]}`; 6 h TTL; tmp+rename |

### 5. skill-pins.json (`agente-agent`)

| Artifact | Mode | Anchor (Go) | Normalization | Content |
|---|---|---|---|---|
| pins-schema | Q + P | `skill_pins_test.go`, `skill_pins.go:54-156` | — | `{global[], projects{cwd[]}}` sorted arrays; trailing newline; tmp `CreateTemp` 0600 + rename |
| pins-corrupt | B | `skill_pins.go:83-85` | — | corrupt file → `invalid skill-pins.json`, never overwritten |
| pins-deep-d10 | P | D10 (ADR 0004) | — | unknown keys in `projects` entries survive a pin toggle, sorted after known fields |

## Generation notes (for ticket 13)

1. Generate Go-authored files by running the named Go tests/anchors against the local oracle with seam injection where available; where Go lacks a fixture (models-cache), author the golden from the schema in the contract row.
2. All goldens committed under a `fixtures/` tree; manifest records the oracle-tree fingerprint (contract §11).
3. Normalization applies at replay time in Rust tests (contract §11 list); session `cwd`-derived paths and times/ids normalized, never edited in the fixture files.
4. CI is Go-free: only P/Q/B modes run. Dev-time differential probing is optional and never wired into CI.