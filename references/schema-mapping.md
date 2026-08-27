# Selecting and Mapping the Schema

Choose the smallest faithful transitive closure of application use.

## Inventory the Surface

For every used operation, record:

| Area | Record |
| --- | --- |
| Operation | HTTP method, resolved path, relevant status codes and media types |
| Request | Used path/query/header fields and constructed body fields |
| Response | Read fields, accessors, used union branches, null handling |
| Control flow | Pagination, polling, JSONL, multipart, uploads or downloads |
| Failure | Error envelope and malformed-response behavior |
| Byte identity | Encodings used in hashes, IDs, caches, snapshots or persistence |

Search source, tests, examples, fixtures, and manifests. Direct field access matters as much as
function calls. Start from used operations and follow their references; do not begin by translating
`components.schemas` wholesale. Stop recursive cycles once the used fields are represented.

The method, path, parameters, request body, responses, and referenced schemas define the operation.
Do not treat `operationId` naming or examples as stronger authority than the schema. If the document
is contradictory in a way that affects behavior, surface the ambiguity.

## Direct Mappings

- Object -> plain Nim `object` with wire-facing `snake_case` fields.
- Array -> `seq[T]`; use `array[N, T]` only for a guaranteed fixed length.
- Closed wire literals -> enum with explicit string values when spelling differs.
- Open-ended string namespace -> `string`.
- Null or absence whose distinction matters -> `Option[T]`.
- Open JSON value that callers retain or pass through -> Brian's `RawJson`.
- Unused response property -> omit it and let Brian skip it.

Honor `readOnly`, `writeOnly`, OpenAPI 3.0 `nullable`, and OpenAPI 3.1 unions containing `null`. A
property missing from `required` is not automatically nullable.

`additionalProperties` controls undeclared object keys; it does not create a field or require custom
JSON code. Ignore it when dynamic keys are unused. If callers read them, model the consumed value
shape with Brian's ordinary mappings. If callers pass the open object through, use `RawJson`.
When `additionalProperties` appears inside a JSON Schema document the application is constructing,
it is simply data in that document, commonly a Boolean field.

## Composition

Flatten consumed `allOf` properties into one public object when the composition represents one
semantic value. Preserve relevant required fields and constraints; do not create inheritance-like
wrappers.

An OpenAPI `oneOf` or `anyOf` does not automatically require a local variant. Project the fields the
application reads when that is sufficient. Use a discriminated object only when callers consume
multiple branch-specific shapes. Put any custom reader on that nested union type, not on its
containing result. See [brian-idioms.md](brian-idioms.md).

Do not add empty unknown arms or raw fallbacks mechanically. Preserve an unfamiliar branch as
`RawJson` only when retaining that payload is a requirement.

## Optionality and Enums

For responses, use `Option[T]` when omitted or `null` is meaningful. For requests, prefer a clear
omission sentinel or documented server default when omission and a concrete value are the only
states. Use `Option[T]` when omitted, `null`, and concrete values mean different things.

Request enums remain strict. A server-owned response enum may use a documented `unknown = ""`
fallback only when discarding the unfamiliar spelling is safe. Use `string` when callers need that
spelling. Keep `unknown` distinct from request-side `unspecified = ""`, which writers omit.

Brian skips unknown object fields in normal `ufSkip` decoding while still validating known fields.
Use `ufReject` for exact fixture validation. Do not write an object-field dispatcher merely to skip
fields Brian already handles.

## Directional Implementation

- Request-only type: construction and `writeJson` only as needed.
- Response-only type: generic decoding or a focused `readJson`.
- Bidirectional type: both paths only when the application sends and receives it.

Request writers emit required fields, omit selected sentinels/defaults, encode fixed literals, and
serialize only the active union arm. Convert a typed value to embedded raw JSON exactly once with
`RawJson(toJson(value))`.
