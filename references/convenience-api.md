# Focused Convenience API Patterns

The useful novelty in `deps/openai` is a thin ergonomic layer over public wire types. Constructors
prevent invalid or noisy assembly; accessors express application semantics. Neither creates a
second model.

## Typed Custom Schemas with a Raw Canonical Overload

JSON Schema is an open language, so the canonical helper accepts Brian's `RawJson`. When the
application repeatedly constructs one known schema shape, it may define a small serializable Nim
type and use a generic overload that calls `toJson` exactly once:

```nim
import brian

type
  SchemaProperty = object
    `type`: string
    description: string

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

let format = formatSchema("search", SearchSchema(
  `type`: "object",
  properties: (query: SchemaProperty(
    `type`: "string",
    description: "Search query")),
  required: @["query"],
  additionalProperties: false
))
```

Here the named tuple models a fixed `properties` object and `additionalProperties` is the Boolean
JSON Schema keyword. No `Table` is needed. Do not model the whole JSON Schema language; add only the
small schema structs the application actually constructs.

The raw overload remains the canonical implementation. The generic overload delegates to it with
`RawJson(toJson(schema))`. Never double-serialize and never add a second schema representation.

For a required fallback schema, keep one explicit constant such as an empty-object schema and have
the constructor or writer supply it. Do not make every caller assemble the same literal.

## Constructors Should Remove Protocol Boilerplate

Follow the `deps/openai` pattern for stable request shapes:

- Text-or-parts input gets typed variant constructors such as `contentText` and `contentParts`.
- Stable message, content-part, function-output, and tool-choice shapes stay typed rather than raw.
- A JSON-producing helper such as `functionOutputJson[T]` serializes once and delegates to the text
  constructor when the wire protocol carries JSON inside a string.
- A constructor uses `sink` for values stored in its result and builds the public schema object
  directly.
- Fixed single-valued protocol fields are filled by the constructor or writer rather than exposed as
  caller choices.

Do not add a constructor that merely repeats an ordinary exported field assignment.

## Accessors Should Add Semantics

For optional response data, pair a predicate with a strict borrowed accessor:

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

proc totalTokens(x: Result): int =
  result = x.usageOf().total_tokens

proc resultParse(body: string; dst: out Result): bool =
  try:
    dst = fromJson(body, Result)
    result = true
  except JsonParsingError:
    dst = default(Result)
    result = false

```

Apply the same pattern to optional errors, files, metadata, counts, and response/error arms. Do not
fabricate empty objects or strings when absence matters.

For heterogeneous output sequences, semantic helpers scan in response order. `firstText` searches
for the first non-empty text part; function-call helpers search for function calls. Positional access
is a separate, index-validating accessor. Do not assume `output[0]` has the desired subtype.

A typed branch accessor checks the discriminator and raises `ValueError` for the wrong branch before
returning `lent` payload storage. Parsing helpers for JSON contained in returned text or function
arguments decode directly with Brian, reset the destination on `JsonParsingError`, and return `bool`
when success/failure is the complete contract.

Every `bool` parse helper assigns `result = true` on success and `result = false` in the handled
failure branch. Do not rely on the implicit default value of `result` for either outcome.

## Keep the Layer Thin

- Direct wire fields remain directly readable.
- Return `lent T` for borrowed strings, sequences, objects, and `RawJson`; return scalars by value.
- Add mutable accessors only for direct owned storage where callers need to move or mutate it.
- Keep validation, serialization mechanics, and repeated failure helpers private.
- Add only conveniences used by the application or needed to prevent invalid construction.
