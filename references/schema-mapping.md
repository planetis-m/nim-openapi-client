# OpenAPI to Brian Schema Mapping

Use this reference while selecting schemas or implementing JSON types. The goal is the smallest
faithful transitive closure of the application's runtime use.

## Select the Surface

Build an inventory with one row per used operation:

| Item | Record |
| --- | --- |
| Operation | HTTP method, resolved path, operation ID if useful |
| Request | Used path/query/header fields, body media type, constructed body fields |
| Response | Statuses handled, fields read, derived accessors used, union branches observed |
| Control flow | Pagination, polling, retries delegated to transport, downloads, JSONL, multipart |
| Failure | Error envelope fields, malformed-response behavior, missing-data behavior |
| Byte identity | Hashes, signatures, caches, IDs, snapshots, or persistence derived from encoding |

Search source, tests, examples, manifests, and fixtures. Direct field access matters as much as
function calls. Follow `$ref` edges from this inventory rather than starting at `components.schemas`.
Resolve external references relative to the owning document, and stop recursive schema cycles once
the used fields are represented.

Do not assume `operationId` names are correct or stable. The method, path, parameters, request body,
responses, and referenced schemas define the protocol.

Treat examples as fixtures and clarification, not as stronger authority than the schema. When the
document contradicts itself in a way that changes runtime behavior, surface the ambiguity rather
than silently choosing the most convenient interpretation.

For a dependency replacement, application tests and persisted formats may establish an observable
contract beyond JSON semantic equivalence. Preserve field order, omission, whitespace, or trailing
newlines only when such bytes are actually hashed, signed, compared, cached, or stored; otherwise do
not mistake incidental encoding order for an OpenAPI requirement.

## Ordinary Mappings

- JSON object -> plain Nim `object` with wire-facing `snake_case` fields.
- Array -> `seq[T]`, or `array[N, T]` only when the protocol guarantees a fixed length.
- String literal set -> enum. Use explicit enum string values when Nim spelling differs.
- Open-ended string namespace -> `string`, not an enum that will reject future values.
- Arbitrary/open object that the application must retain or pass through -> `RawJson`; omit it from
  the local projection when it is unused.
- JSON `null` or absent response field whose absence matters -> `Option[T]`.
- Integer/number -> the narrowest sound Nim numeric type; preserve timestamp and size ranges.

`additionalProperties` is a schema constraint, not a field that automatically needs representation,
and it does not by itself require a custom Brian reader or writer. Use `Table[string, T]` only when
the application consumes dynamic keys whose declared value schema is `T`. Otherwise omit unused
dynamic entries, or keep the whole value as `RawJson` when the application must pass it through.

Honor `readOnly`/`writeOnly` direction. In OpenAPI 3.0, inspect `nullable`; in OpenAPI 3.1, inspect
whether the JSON Schema type includes `null`. A property being absent from `required` does not by
itself mean the server returns `null`.

Keep protocol constraints visible where useful, but do not invent validation the application does not
need. Parsing must still reject type mismatches, malformed JSON, and numeric overflow.

## Composition and Unions

### `allOf`

Flatten the consumed properties into one public object when the composed schema represents one
semantic value. Preserve duplicate constraints and required fields. Avoid inheritance-like wrapper
objects that make ordinary field access cumbersome.

### `oneOf` and `anyOf`

Use a discriminated object when callers need multiple shapes. Decode the discriminator when the
specification provides one; otherwise distinguish branches by token kind or required shape.

Model only branches that the application consumes. Do not add an empty unknown variant arm or a raw
fallback merely because the OpenAPI schema says `oneOf`. Preserve an unfamiliar branch as `RawJson`
only when the application actually stores or passes that payload through. Do not expose
unimplemented request branches as raw escape hatches merely to make the client look complete.

When a union is represented as a variant, put the custom reader on the union member itself. The
object containing it remains an ordinary Brian-decoded object. For example, a result with
`output: seq[Output]` needs a custom reader for `Output`, not for the result. Follow the concrete
pattern in [brian-idioms.md](brian-idioms.md#use-a-variant-only-for-multiple-used-shapes).

### Nullability

Distinguish three cases deliberately:

- A response may omit or return null: usually `Option[T]`.
- A request omits a field when it has a clear sentinel or server default: store the ordinary type
  and omit it in a custom writer.
- Omitted, null, and concrete values have different request meanings: use `Option[T]` or an explicit
  variant that represents all three states.

## Enums and Compatibility

Request enums should be strict so callers cannot construct undocumented literals. For an evolving
server-owned response enum, `unknown = ""` is appropriate only when the application can safely lose
the unfamiliar wire spelling. Give every such member a doc comment stating that it is a read-side
fallback, that the original spelling is discarded, and what callers may safely do with it. Add a
focused reader that maps unfamiliar strings to `unknown`. Use `string` instead when the application
must retain the spelling. Do not weaken every enum because one output namespace evolves.

Keep `unknown` distinct from a request-side `unspecified = ""` omission sentinel. Writers must omit
`unspecified`; they must never emit the empty string as a protocol value.

Brian's generic object reader initializes missing fields to Nim defaults; it does not prove that
OpenAPI-required properties were present. Validate required fields at the public parse boundary when
their absence would otherwise look like a valid default. Use a custom object reader with presence
flags only when several required fields or null/default distinctions make boundary validation
insufficient.

Unknown object fields should normally use Brian's `ufSkip`. `ufReject` is useful for fixture/schema
validation, not normal production parsing. Every custom reader accepts
`UnknownFieldPolicy` and forwards it through nested `readJson` calls.

Do not write manual object dispatch merely to ignore fields. Brian handles ordinary objects. A
custom reader is justified for a real union, tolerant enum, unusual envelope, or another shape
generic decoding cannot represent.

## Directional Types

Match implementation to protocol direction:

- Request-only type: construction plus `writeJson` as needed.
- Response-only type: generic or custom `readJson`.
- Bidirectional type: both only when the application genuinely sends and receives it.

Custom request writers should always emit required fields, omit chosen sentinels/defaults, encode
fixed single-valued protocol fields internally, and emit only fields valid for the active union arm.
Never serialize a value twice. To embed a typed value as raw JSON, use
`RawJson(toJson(value))`, not `RawJson(toJson(toJson(value)))`.

## Raw JSON Boundaries

Use Brian's `RawJson` directly. Good boundaries include arbitrary metadata, JSON Schema documents,
extension dictionaries, and deliberately opaque response branches. Stable messages, content parts,
errors, pagination, and fields used for control flow should stay typed.

This decision is independent of schema composition: `oneOf` and `anyOf` do not inherently require a
raw fallback, and an ordinary non-union property may legitimately be `RawJson` when it is open by
contract and used opaquely by the application.

Use `$raw` when the application needs the stored JSON text and `toJson(raw)` when serializing it as a
JSON value. Do not define an application-local alias or conversion layer.

`RawJson("")` is an omission sentinel only when a custom writer checks it before serialization. A
non-empty manually constructed `RawJson` must contain one complete valid JSON value. Use
`CanonRawJson` only when normalized re-emission is an actual requirement; both Brian raw types
already provide `$`.

When constructing a JSON Schema document, `additionalProperties: false` is simply a Boolean keyword
inside that document. The `deps/openai` structured-output pattern uses named tuples for fixed
`properties` and a Boolean for this keyword; no table or special serializer is involved.
