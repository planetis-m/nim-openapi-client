---
name: nim-openapi-client
description: Implement or reduce Nim REST API clients from OpenAPI definitions, modeling only the operations and wire fields an application uses with Brian. Use when replacing a broad SDK, adding a local API binding, or auditing an OpenAPI-derived minimal client. Do not use for complete SDK generation or non-Nim clients.
---

# Nim Minimal OpenAPI Client

Build a local client that is complete for the application and deliberately incomplete for the API.
The OpenAPI document defines the wire protocol; application code and tests define the required
surface and compatibility contract. An existing SDK may illustrate ergonomics, but implementation
must not depend on one being available.

## Rules

- Inventory application usage before defining types. Include operations, constructed request fields,
  read response fields, accessors, errors, pagination, files, fixtures, and serialized bytes used in
  hashes, IDs, caches, or persistence.
- Follow only the OpenAPI references needed by that inventory. Preserve methods, paths,
  authentication, parameter encoding, media types, literals, required fields, omission, nullability,
  defaults, errors, and used response shapes.
- Use Brian directly: `fromJson`, `fromFile`, `toJson`, `RawJson`, and focused `readJson`/`writeJson`
  overloads. Do not add JSON shims, aliases, wrapper types, probe models, or DOM-based parsing.
- Let Brian handle ordinary objects and containers generically. Add custom JSON code only for a
  shape generic mapping cannot express or for deliberate request omission.
- Keep stable used wire data typed. `RawJson` is for an open value the application must retain or
  pass through; it is unrelated to whether the OpenAPI schema uses `oneOf`.
- Use the project's existing HTTP transport directly. Do not add another transport abstraction.
- When replacing a dependency, preserve existing names, call shapes, fields, accessors, and failure
  behavior. Application files should ideally change only imports.
- If Brian is editable and lacks a generally useful primitive, add it to Brian rather than spreading
  conversions or compatibility helpers through the application.
- If the specification is silent or contradictory for an in-scope behavior, preserve an established
  local contract when clear and report the ambiguity; do not invent protocol behavior from an SDK.
- Do not copy SDK code unless its license and the task permit it. Derive protocol behavior from the
  OpenAPI document and observable application contracts.

## Workflow

1. Locate the supplied OpenAPI document and identify its version, servers, security schemes, and
   in-scope operations. Do not browse for a different specification unless requested.
2. Search source and tests for every use of the old client or remote API. Record an
   operation-to-usage matrix before coding.
3. Walk the request and response schemas only as far as runtime use requires, including `$ref`,
   composition, discriminator, nullability, enums, and open-object constraints.
4. Design public request/result objects and only the constructors and accessors callers use. During
   replacement, do not introduce unrelated type tightening or cleanup.
5. Implement deliberate request writers, response decoding, request construction, parse boundaries,
   and error envelopes. Keep containing result objects generic when only a nested value is custom.
6. Replace imports and dependency metadata. Treat every non-import application diff as a possible
   compatibility leak and justify or remove it.
7. Verify exact requests, realistic responses, unknown-field compatibility, malformed input,
   application builds, dependency locks, untracked files, and diff cleanliness.

## References

- Read [references/schema-mapping.md](references/schema-mapping.md) to select the minimal OpenAPI
  schema surface.
- Read [references/brian-idioms.md](references/brian-idioms.md) before implementing enums, unions,
  `RawJson`, or custom Brian readers and writers.
- Read [references/client-structure.md](references/client-structure.md) for module ownership, custom
  schema helpers, constructors, accessors, parse helpers, HTTP integration, and replacement details.
- Read [references/verification.md](references/verification.md) before handoff or audit.

## Completion

The client is complete when every application-used behavior is locally represented and verified,
not when the OpenAPI document is exhausted.
