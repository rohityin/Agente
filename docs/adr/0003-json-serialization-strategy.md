# JSON serialization strategy

The Rust port uses serde and serde_json as its only JSON codec. Fixed-shape wire and state structs derive `Serialize`/`Deserialize` with fields in Go declaration order and shared `skip_serializing_if` helpers mirroring Go's `omitempty`; surfaces serde cannot express get hand-written codecs (the untagged `provider.Content` field-presence union with Go's precedence and silent degradation, session message rows, swarm event flattening, provider oneofs); tool arguments, tool schemas, the six extension-protocol raw fields, and session argument bytes cross the system as `Box<RawValue>`, mirroring Go's `json.RawMessage`. `serde_json::Value` appears only where Go itself decoded through `any` (RPC/`--json` args, Gemini schema sanitizing, Bedrock tool args, swarm events, timeline/preview), with an explicit coerce-to-float64 pass reproducing Go's number degradation at those sites. Machine JSON is schema-exact, not byte-exact: ticket 07 amended the contract's RPC/`--json` E rows to S, accepting serde_json's escaping, float text, and key order, while frame builders still mirror Go's construction (sorted maps where Go built maps, declaration order where Go used structs). Unknown keys are preserved on whole-file state rewrites as raw fragments — a deliberate divergence (D10) from Go, which drops them.

Status: accepted

## Considered options

- **Go-compatible emitter** (a `serde_json::ser::Formatter` implementing Go escaping, float formatting, and key ordering): rejected; byte goldens on RPC/`--json` are not worth a custom formatter when those consumers parse the output, and the contract rows were amended to S instead.
- **Hand-rolled Go-compatible parser**: rejected; serde_json parses the grammar, and Go's remaining leniencies (invalid UTF-8 replacement, duplicate-key last-wins, deep nesting, error wording) are pathological inputs, recorded as D11/D12 rather than reproduced.
- **`serde_json::Value` everywhere**: rejected; destroys the raw passthrough Go guarantees for tool arguments and schemas and gives up typed shape safety.
- **`#[serde(untagged)]` for `provider.Content`**: rejected; serde cannot express Go's conjunction predicate (`name && id`) or its silent degradation to an empty text block.
- **Dropping unknown state-file keys like Go**: rejected (user decision); preserving keys a future version or the user wrote is worth the recorded divergence.

## Consequences

- Dependencies: `serde` + `serde_json` (`raw_value` enabled; `preserve_order` and `arbitrary_precision` disabled so maps stay BTreeMap-sorted like Go's map marshaling) + RustCrypto `base64` for `[]byte` fields. No JSON-manipulation crates; never `deny_unknown_fields`; no `serde(flatten)` for protocol frames.
- Placement: the shared JSON layer (omitempty helpers, number coercion, content codec) is an `agente-types` module; the session-row codec lives with sessions in `agente-core`; RPC/`--json` frame builders live in the agent crate. No new crate.
- Parity contract amendments (recorded on ticket 07): §1 `--json` row and §2 RPC rows E→S; §3 write rows and quirks 7/15 annotated with D10; divergence register gains D10, D11, D12.
- Session JSONL keeps Go semantics exactly: raw-copy paths preserve unknown fields, hydrated rewrites drop them; the row codec reproduces `hydrateMessageObject` precedence (reasoning > tool call when `name && id` > tool result via `call_id` > image via `mime_type` > text) including degradation and nested text/image-only tool-result content.
- Emission conventions stay Go-shaped: compact JSONL plus newline, `MarshalIndent` 2-space with no trailing newline, field order from Go struct order, sorted maps, `models-cache.json` PascalCase renames.
- The datetime representation (RFC3339Nano) and provider oneof codecs are inherited by tickets 08 and 09.
