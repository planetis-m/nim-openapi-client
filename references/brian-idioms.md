# Brian Idioms for Minimal API Models

Use Brian directly. Do not add a JSON compatibility module, a local `RawJson` type, a private wire
model, a JSON DOM, or a parse-probe/reparse layer.

## Project Before Customizing

An OpenAPI `oneOf` does not force the local model to be a variant. If the application only reads
fields common to the useful branch and checks the wire discriminator, an ordinary projection is
smaller and Brian decodes it generically:

```nim
import std/options
import brian

type
  ResponseStatus* {.pure.} = enum
    unknown = "" ## Unrecognized server value; spelling is discarded; do not assume completion.
    completed
    failed

  OutputPart = object
    `type`: string
    text: string

  Output = object
    `type`: string
    content: seq[OutputPart]

  Usage = object
    total_tokens: int

  Result = object
    id: string
    status: ResponseStatus
    output: seq[Output]
    usage: Option[Usage]

proc readJson(dst: var ResponseStatus; p: var JsonParser;
    unknownFields: UnknownFieldPolicy) =
  var value: string
  readJson(value, p, unknownFields)
  case value
  of "completed": dst = ResponseStatus.completed
  of "failed": dst = ResponseStatus.failed
  else: dst = ResponseStatus.unknown

```

The containing `Result`, `Output`, sequences, options, and ordinary nested objects need no custom
reader. This is the preferred minimal response model when branch-specific payloads are not used.

## Document Every `unknown` Enum Member

Use `unknown = ""` only for a server-owned response enum where unfamiliar future values must not
break decoding. Its doc comment must state:

- that it represents an unrecognized wire value;
- whether the original spelling is discarded;
- what application behavior is safe when it occurs.

The focused reader above maps only documented literals and sends everything else to `unknown`.
Known literals still get type-checked. If the application must log, store, compare, or round-trip the
unrecognized spelling, use `string` instead; an `unknown` enum loses that information.

Do not put `unknown` on request enums. Requests should reject undocumented values. Do not confuse it
with `unspecified = ""`, which is a request-side omission sentinel that a custom writer must omit.

## Use a Variant Only for Multiple Used Shapes

When callers consume branch-specific fields from two or more shapes, use a discriminated object and
put the custom reader on that object—not on its containing result:

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
  var wireType, id, reason: string
  for name in p.jsonFields:
    case name
    of "type": readJson(wireType, p, unknownFields)
    of "id": readJson(id, p, unknownFields)
    of "reason": readJson(reason, p, unknownFields)
    else:
      if unknownFields == ufReject:
        p.raiseParseError("unexpected event field: " & name)
      p.skipJson()

  case wireType
  of "opened": dst = Event(kind: opened, id: id)
  of "closed": dst = Event(kind: closed, reason: reason)
  else: p.raiseParseError("unexpected event type: " & wireType)

type Envelope = object
  events: seq[Event]

```

This example is deliberately closed and contains no raw fallback. If unfamiliar server branches
must remain decodable but their fields are unused, first reconsider the ordinary projection pattern
above. Only when the application must preserve an unfamiliar branch's payload should the variant
gain an opaque arm containing Brian's `RawJson`. That is an explicit pass-through feature, not a
property of `oneOf`.

Do not collect discarded fields, add an empty unknown arm, or parse twice merely to avoid deciding
whether unknown payload preservation is required. If preservation is required and the pinned Brian
version lacks a direct primitive needed to implement it cleanly, add that generally useful primitive
to Brian when Brian is in scope.

## Token-Kind Unions

When arms differ by JSON token kind, use Brian's direct parser-kind pattern:

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

## `RawJson` Is an Independent Open Boundary

Use Brian's `RawJson` only when the application needs to retain or pass through a complete JSON
value whose internal schema is intentionally not modeled, such as a JSON Schema document or
arbitrary metadata:

```nim
import brian

type Tool = object
  name: string
  schema: RawJson

let tool = fromJson(
  """{"name":"search","schema": { "type": "object" }}""",
  Tool)
```

Brian already supplies the required ergonomics:

- `$value` returns stored text for `RawJson` and `CanonRawJson`.
- `writeJson` and `toJson` emit a raw value as JSON rather than quoting it.
- `fromJson(input, RawJson)` validates and captures one complete value.
- `CanonRawJson` is for deliberately normalized re-emission, not stronger typing.

Manually constructing `RawJson` trusts the bytes. `RawJson("")` may be used as an omission sentinel
only when a custom writer checks it before serialization. Never add aliases, converters, wrappers,
or a client-local `$` for Brian's raw types.

## Custom Reader and Writer Rules

- Every custom reader accepts `UnknownFieldPolicy` and forwards it to nested reads.
- Use `p.jsonFields` for objects, `p.kind` for token-shape unions, and `p.skipJson()` for deliberately
  ignored values.
- Use `w.write` for fixed syntax, `w.escapeJson` for dynamic strings or keys, and `writeJson` for
  values.
- Custom-read only genuine unions, tolerant response enums, positional shapes, or unusual envelopes.
- A request union writer emits only its active arm. An ordinary containing result stays generic.
