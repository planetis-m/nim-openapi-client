# Minimal Client Structure and Ergonomics

Use this reference when shaping the local package and HTTP-facing operations.

## Module Ownership

- `config.nim`: base URL and the authentication/configuration values actually used.
- `http.nim`: authentication headers, query construction, and a small helper that produces the
  project's native request specification.
- `error.nim`: the service's common error envelope and accessors.
- `<capability>.nim`: one cohesive endpoint family, including its public schema, constructors,
  request builders, parsers, and semantic accessors.
- `schema/`: optional; use only when schema volume or cross-capability reuse makes separation useful.

Do not hide an existing transport behind a second client/transport interface. Status classification,
retry policy, connection pooling, and scheduling stay with the transport or application unless the
OpenAPI protocol itself defines them.

If the project has no transport, add the smallest direct HTTP integration needed by the selected
operations rather than designing a reusable transport framework. Do not make live requests merely to
validate generated bindings unless the user authorized them.

## Public API

Use public schema objects directly. Avoid private `Wire` objects, type aliases, converters, and a
second model copied into an ergonomic representation.

For a new client, choose one consistent operation vocabulary, for example:

- `<operation>Create` or a focused constructor for request parameters.
- `<operation>Request` for the native HTTP request specification.
- `<operation>Parse` for parse-into-caller-storage returning `bool`.

Keep constructors focused on preventing invalid or noisy wire assembly. Do not add a helper that
merely repeats an exported field assignment.

Schema fields describe the wire. Accessors describe application semantics:

- Use direct field access for ordinary required wire data.
- Return `lent T` for borrowed strings, sequences, objects, and `RawJson`.
- For optional required-by-caller data, expose `hasX` and a strict `x`/`xOf` accessor that raises a
  precise `ValueError` rather than manufacturing a default.
- Scan heterogeneous output collections semantically; do not assume the first item has one subtype.
- Route repeated accessor failures through one private `{.noinline, noreturn.}` helper.

## HTTP Fidelity

Derive every request detail from the OpenAPI operation:

- Resolve the server/base URL and path exactly.
- Expand server variables as specified. Encode path and query values with the transport's URL
  facilities; do not concatenate unescaped application values.
- Include only parameters for their documented location.
- Implement OpenAPI parameter `style`, `explode`, `allowReserved`, and repeated-array behavior for
  every used path, query, header, and cookie parameter; ordinary key/value encoding is not equivalent
  to every OpenAPI serialization style.
- Apply security schemes exactly, including the OR between security requirement objects and the AND
  between schemes named in one object. Avoid logging credentials.
- Set the declared request media type. A shared JSON header helper must allow multipart, form, binary,
  or streaming operations to override `Content-Type`.
- Send the documented `Accept` value when a used operation has multiple response media types, and
  dispatch parsing by status and media type rather than assuming every success body is JSON.
- Preserve raw/binary response bodies for download endpoints.
- Reject or safely encode CR/LF, quotes, and other header-significant characters in multipart field
  names and filenames. Use deterministic boundaries in tests; production boundaries must be legal,
  unpredictable enough for the context, and checked against body collision when needed.
- Encode fixed protocol literals inside writers when callers have no choice to make.

Pagination belongs to the capability when the protocol defines cursors and page shapes. Polling and
batch orchestration can stay in the application while the client provides typed create/list/retrieve
requests and results.

## Parse and Error Boundaries

A public parse helper may catch Brian's `JsonParsingError`, reset the destination, and return `false`
when success/failure is the entire contract. Do not catch unrelated operational errors or defects.

Keep the common remote error envelope typed, tolerant of additive fields, and separate from local
HTTP/network failures. If an endpoint returns a response-or-error sum inside another format such as
JSONL, expose predicates and strict accessors for both arms.

## Replacement Discipline

When replacing an SDK or broad dependency:

1. Preserve imports, type names, constructor calls, field access, and accessor behavior where the
   local subset can do so honestly.
2. If Brian is an editable dependency within scope, put generally applicable missing JSON operations
   there, not in application shims. Examples are direct file decoding or `$` on Brian-owned raw JSON
   representations. If it is outside scope, report the missing capability instead of silently
   expanding authority.
3. Avoid changing application code for style while changing its dependency. Review every non-import
   diff as a possible leaked compatibility workaround.
4. Remove old dependency declarations, compiler paths, defines, and lock entries. Pin the Brian
   revision that actually contains required APIs.

During a replacement, compatibility outranks opportunistic type tightening. Keep an established
string-taking call shape when changing it would create unrelated caller churn; validate or encode
the closed literal inside the local boundary. For a new API, prefer the stronger enum or constructor
from the start.

An existing SDK can be consulted for ergonomic ideas or regression fixtures, but the client must be
implementable from the OpenAPI definition alone. Do not inherit undocumented SDK behavior without an
application contract or protocol basis.
