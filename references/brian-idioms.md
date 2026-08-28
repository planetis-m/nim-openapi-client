# Brian JSON Idioms

Read the pinned Brian README before implementation. Use its public API directly; do not add a JSON
compatibility layer, local raw type, DOM, probe model, or reparse step.

## Prefer Generic Projection

Model only fields the application reads. Brian decodes ordinary containing objects and skips other
fields without custom code, even when the full OpenAPI schema describes a union:

```nim
type
  OutputPart = object
    `type`: string
    text: string

  Output = object
    `type`: string
    content: seq[OutputPart]

  Result = object
    id: string
    output: seq[Output]
```

Add a variant only when callers use branch-specific fields from multiple shapes.

## Required Wrapper Fields

Brian's generic object reader leaves an absent field at its default. When callers must distinguish
an absent required envelope key from that default, give only the envelope a custom reader: start
with `var foundError = false`, set it when the `"error"` field occurs, decode that value with the
ordinary `readJson`, then finish with
`if not foundError: p.raiseParseError("missing error field")`. Forward `UnknownFieldPolicy`
normally; do not custom-decode the inner object unless it has its own special shape. A Boolean is
clearest for one required key; when several keys are required, track their presence with a
`set[enum]` instead of parallel Booleans.

## Tolerant Response Enums

For an evolving server-owned response enum, map unfamiliar strings with a focused reader:

```nim
import brian

type
  ResponseStatus {.pure.} = enum
    unknown = "" ## Unrecognized server value; spelling is discarded; do not assume completion.
    completed
    failed

proc readJson(dst: var ResponseStatus; p: var JsonParser;
    unknownFields: UnknownFieldPolicy) =
  readJson(dst, p, unknownFields, unknown)
```

Every `unknown` member must document that it is a read-side fallback, whether the original spelling
is retained, and what callers may safely infer. Use `string` instead if callers need the spelling.
Request enums are strict. A request-side `unspecified = ""` is a different concept: its writer omits
the field.

## Discriminated Object Unions

When multiple used branches have different fields, parse the nested union once with `p.jsonFields`.
The containing result remains generic:

```nim
import brian

type
  EventKind = enum
    opened
    closed

  Event = object
    case kind: EventKind
    of opened:
      id: string
    of closed:
      reason: string

proc readJson(dst: var Event; p: var JsonParser;
    unknownFields: UnknownFieldPolicy) =
  var kind: EventKind
  var id, reason: string
  for field in p.jsonFields:
    if field == "type":
      readJson(kind, p, unknownFields)
    elif field == "id":
      readJson(id, p, unknownFields)
    elif field == "reason":
      readJson(reason, p, unknownFields)
    elif unknownFields == ufReject:
      p.raiseParseError("expected known field, got \"" & $field & "\"")
    else:
      p.skipJson()

  case kind
  of opened: dst = Event(kind: opened, id: id)
  of closed: dst = Event(kind: closed, reason: reason)
```

This is a closed union with no raw or empty fallback. If unfamiliar branches must decode but their
payload is unused, reconsider generic projection. Add an opaque `RawJson` arm only when callers must
retain the unfamiliar payload. If clean retention needs a missing Brian primitive and Brian is in
scope, add the primitive there rather than collecting fields or parsing twice in the client.

## Token-Kind Unions

When branches differ by JSON token kind, branch on `p.kind` and use the same variant for writing:

```nim
import brian

type
  ContentKind = enum
    text
    parts

  Content = object
    case kind: ContentKind
    of text:
      body: string
    of parts:
      items: seq[string]

proc readJson(dst: var Content; p: var JsonParser;
    unknownFields: UnknownFieldPolicy) =
  case p.kind
  of jkString:
    dst = Content(kind: text)
    readJson(dst.body, p, unknownFields)
  of jkArray:
    dst = Content(kind: parts)
    readJson(dst.items, p, unknownFields)
  else:
    p.raiseParseError("expected string or array")

proc writeJson(w: var JsonWriter; value: Content) =
  case value.kind
  of text: writeJson(w, value.body)
  of parts: writeJson(w, value.items)
```

## Request Writers

Use a custom writer when generic object serialization would emit request defaults or inactive
fields that the protocol expects omitted:

```nim
import brian

type
  Params = object
    model: string
    instructions: string
    store: bool = true

proc writeJson(w: var JsonWriter; x: Params) =
  w.write "{\"model\":"
  writeJson(w, x.model)
  if x.instructions.len > 0:
    w.write ",\"instructions\":"
    writeJson(w, x.instructions)
  if not x.store:
    w.write ",\"store\":false"
  w.write "}"
```

Always emit required fields. Emit optional fields only when they differ from the chosen omission
sentinel or documented server default. Use Brian's generic writer when it already produces the
required request shape.

## Raw JSON

Use Brian's `RawJson` for a complete open value the application retains or passes through:

```nim
import brian

type
  Tool = object
    name: string
    schema: RawJson
```

- `$value` returns stored text for `RawJson` and `CanonRawJson`.
- `writeJson` and `toJson` emit the raw value as JSON rather than quoting it.
- `fromJson(input, RawJson)` validates and captures one complete value.
- `CanonRawJson` is for normalized re-emission, not stronger typing.

Manually constructing `RawJson` trusts its bytes. `RawJson("")` is only an omission sentinel checked
by a custom writer; it is not valid JSON. Never define client-local aliases, converters, wrappers,
or `$` for Brian's raw types. Convert typed data once with `RawJson(toJson(value))`.

## Custom Code Checklist

- Every reader accepts `UnknownFieldPolicy` and forwards it to nested reads.
- Use `p.kind` for token shape and `p.skipJson()` only for a selected value a reader ignores.
- Use `w.write` for fixed syntax, `w.escapeJson` for dynamic strings or keys, and `writeJson` for
  values.
- A request union writer emits only the active arm.
