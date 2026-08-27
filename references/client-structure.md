# Client Structure and Ergonomics

Keep the client thin: public wire objects, focused constructors and accessors, and the project's
existing transport.

## Modules

Use capability modules plus only shared modules that are actually needed:

- `config.nim`: base URL and used authentication/configuration.
- `http.nim`: shared headers and request construction over the existing transport.
- `error.nim`: common remote error envelope.
- `<capability>.nim`: schema types, constructors, request builders, parsers, and accessors.
- `schema/`: optional; split it out only when schema size or reuse warrants it.

Public request and result objects are the wire types directly. Do not introduce private `Wire`
objects, aliases, converters, or a second representation.

## Constructors

Constructors should remove protocol boilerplate or prevent an invalid shape. The `deps/openai`
patterns generalize well:

- Typed variants and focused constructors for stable message, content, and tool shapes.
- `sink T` for inputs stored in the result.
- Fixed single-valued protocol fields set internally rather than exposed as caller choices.
- A generic JSON-producing helper serializes once and delegates to its canonical constructor.

Do not add a constructor that merely repeats exported field assignment.

### Open JSON Schema with Typed Local Helpers

Keep arbitrary JSON Schema as `RawJson`. For a schema shape the application constructs repeatedly,
define only that shape and provide a generic overload that delegates to the raw overload:

```nim
import brian

type
  SchemaProperty = object
    `type`: string

  SearchSchema = object
    `type`: string
    properties: tuple[query: SchemaProperty]
    required: seq[string]
    additionalProperties: bool

  Format = object
    name: string
    schema: RawJson

proc formatSchema(name: sink string; schema: sink RawJson): Format =
  Format(name: name, schema: schema)

proc formatSchema[T](name: sink string; schema: T): Format =
  formatSchema(name, RawJson(toJson(schema)))

proc searchFormat(): Format =
  result = formatSchema("search", SearchSchema(
    `type`: "object",
    properties: (query: SchemaProperty(`type`: "string")),
    required: @["query"],
    additionalProperties: false
  ))
```

Here `properties` has fixed names, so a named tuple is sufficient;
`additionalProperties: false` is an ordinary Boolean in the JSON Schema document. Do not model the
whole JSON Schema language. Keep any required empty-object fallback as one explicit constant.

## Accessors and Parse Helpers

Direct wire fields remain directly readable. Add accessors for semantic selection, validated
indexing, optional required-by-caller data, or hidden variant storage.

```nim
import std/options
import brian

type
  Usage = object
    total_tokens: int

  Result = object
    usage: Option[Usage]

proc raiseResultError(message: string) {.noinline, noreturn.} =
  raise newException(ValueError, message)

proc hasUsage(x: Result): bool =
  x.usage.isSome

proc usageOf(x: Result): lent Usage =
  if x.usage.isNone:
    raiseResultError("result has no usage data")
  x.usage.get

proc resultParse(body: string; dst: out Result): bool =
  try:
    dst = fromJson(body, Result)
    result = true
  except JsonParsingError:
    dst = default(Result)
    result = false
```

- Pair optional data with `hasX` and a strict `xOf` accessor; do not fabricate empty values.
- Return `lent T` for borrowed strings, sequences, objects, and `RawJson`; return scalars by value.
- Scan heterogeneous collections in response order for semantic helpers; do not assume item zero has
  a particular subtype.
- Validate a variant discriminator before returning its typed payload.
- Every `bool` parse helper sets `result` explicitly in success and handled-failure branches and
  resets `dst` on `JsonParsingError`.

## HTTP Integration

Use the project's transport without wrapping it in another client abstraction. For each selected
operation, implement only the protocol features it uses:

- exact server, path, method, parameter location and OpenAPI serialization style;
- security requirements and secret-safe headers;
- declared request and response media types, including status-dependent parsing;
- binary, multipart, JSONL, pagination, or download behavior when present.

Retries, pooling, scheduling, and generic HTTP status policy stay with the transport or application
unless the API protocol defines them. Keep tests offline unless live requests are explicitly
authorized.

## Dependency Replacement

- Preserve existing public names and behavior; application files should ideally change only imports.
- Put a missing general JSON operation in Brian when Brian is editable, not in a compatibility shim.
- Remove the old dependency, compiler paths, defines, and lock entries; pin the Brian revision that
  contains required changes.
- Review every non-import application diff and avoid unrelated cleanup or type tightening.
