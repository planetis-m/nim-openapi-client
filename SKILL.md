---
name: nim-openapi-client
description: Implement or reduce Nim REST API clients from OpenAPI definitions, modeling only the operations and wire fields an application uses with Brian. Use when replacing a broad SDK, adding a local API binding, or auditing an OpenAPI-derived minimal client. Do not use for complete SDK generation or non-Nim clients.
---

# Nim Minimal OpenAPI Client

Build a small local client whose public surface is application-complete and API-incomplete by
design. The OpenAPI document is the protocol source of truth; an existing SDK may help explain
ergonomics but must not be required.

## Essential Rules

- Inventory application usage before defining types. Include endpoint calls, constructors, direct
  fields, accessors, status handling, pagination, error paths, uploads, downloads, fixtures, and any
  serialized bytes that feed hashes, signatures, caches, durable IDs, or persisted records.
- Follow only the transitive OpenAPI references needed by that inventory. Do not model adjacent
  operations or every subtype of a union merely because the specification exposes them.
- Preserve protocol fidelity inside the selected subset: methods, paths, authentication, query
  encoding, headers, media types, fixed literals, required fields, omission semantics, nullability,
  defaults, error envelopes, and used response branches must remain exact.
- Use `brian` directly for JSON. Use its `RawJson`, `fromJson`, `fromFile`, `toJson`, readers, and
  writers; do not introduce JSON compatibility aliases, wrapper types, or shim modules.
- Read the installed Brian README and inspect its pinned version before relying on convenience APIs;
  update dependency metadata through the project's existing dependency workflow when authorized.
- Keep stable modeled wire data typed. Use `RawJson` only when the application needs to retain or
  pass through an intentionally open JSON value such as JSON Schema or arbitrary metadata. A
  `oneOf` does not by itself justify storing `RawJson`.
- Keep ordinary containing objects on Brian's generic path. When a used `oneOf` genuinely needs a
  variant, attach its custom `readJson` to that nested type; do not custom-read the entire result.
- Skip unknown response fields for forward compatibility. Keep known-field type checking strict.
- Use the project's existing HTTP transport directly. A small shared request helper is useful;
  another transport abstraction is not.
- When replacing a dependency, preserve call-site ergonomics. Existing files should ideally change
  only imports. When Brian itself is in scope, put missing generally useful JSON ergonomics there
  rather than spreading conversions through the application; otherwise report the dependency gap.
- Never copy code from an SDK unless its license and the task permit it. Derive correctness from the
  OpenAPI definition and observable application contracts.

## Evidence Hierarchy

Use the supplied OpenAPI document as authority for the documented REST protocol. Use application
code and tests as authority for which subset is needed and for local observable invariants such as
stable serialization, hashing, persistence, and error translation.

If the OpenAPI document is silent about an in-scope wire format, internally contradictory, or
inconsistent with an established application fixture, do not infer missing protocol from an SDK.
Preserve an established local contract during a replacement when it is unambiguous, record the spec
gap, and ask for direction when choosing would materially affect behavior. Examples and prose can
clarify a schema but do not silently override it.

## Working Shape

For a new client, prefer capability modules plus only the shared modules they actually need:

```text
src/<service>/config.nim
src/<service>/http.nim
src/<service>/error.nim
src/<service>/<capability>.nim
```

Split JSON-mapped schema types into `schema/` only when their size or reuse makes that boundary
useful. Public request/result objects should be the schema values directly, not aliases over private
wire objects.

Use plain objects, enums for closed wire literals, discriminated objects for used unions, and
`Option[T]` where a response may be null or absent and that state matters. Put pleasant derived
behavior in constructors and accessors while leaving ordinary wire fields directly readable.

## Workflow

1. Locate the exact OpenAPI document and identify its version, servers, security schemes, and the
   operations in scope. Prefer a supplied local document; do not browse for another version unless
   the task requires verification.
2. Search the application and tests for every use of the old client or remote API. Make a small
   endpoint-to-usage matrix before coding.
3. For each used operation, walk request and response `$ref`, `allOf`, `oneOf`, `anyOf`, discriminator,
   nullable, enum, and `additionalProperties` edges only as far as runtime usage requires.
4. Design the narrow public API and module ownership. Preserve established names and call shapes
   during a replacement; do not introduce breaking enum/type tightening as incidental cleanup. Use a
   consistent operation-qualified and typed vocabulary for a new client.
5. Implement request types and deliberate writers, response types and tolerant readers, request
   construction, parsing boundaries, error envelopes, and only the accessors the application uses.
6. Replace imports and dependency metadata. Avoid opportunistic refactors; inspect every non-import
   application diff and justify why Brian or the selected schema requires it.
7. Verify exact wire output, realistic response decoding, unknown-field compatibility, malformed
   input behavior, and the application's full build and tests.
8. Audit the final dependency lock, old imports, untracked source modules, generated binaries, and
   diff cleanliness before calling the replacement reproducible.

## Detailed Guidance

- Read [references/brian-idioms.md](references/brian-idioms.md) before implementing `RawJson`, a
  `oneOf`/`anyOf`, a tolerant response enum, or a custom Brian reader/writer. Use those direct
  patterns instead of adding probe models, reparsing layers, JSON DOMs, or compatibility wrappers.
- Read [references/schema-mapping.md](references/schema-mapping.md) when selecting and translating
  OpenAPI schemas into Brian-backed Nim types.
- Read [references/client-structure.md](references/client-structure.md) when designing modules,
  request builders, errors, pagination, files, or migration ergonomics.
- Read [references/convenience-api.md](references/convenience-api.md) when adding typed constructors,
  custom-schema helpers, semantic response accessors, or parse helpers.
- Read [references/verification.md](references/verification.md) before testing or reviewing a client
  for completeness, compatibility, performance, and commit readiness.

## Completion Standard

The client is complete when every application-used API behavior is locally represented and tested,
not when the OpenAPI document is exhausted. Report intentionally unsupported operations or union
branches only when callers could reasonably encounter them in the selected endpoints.
