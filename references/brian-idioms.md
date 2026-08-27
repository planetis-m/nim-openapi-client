# Brian Idioms for Minimal API Models

Use these patterns directly. They are the reusable parts of the `deps/openai` design, expressed with
Brian's public API. Do not add a JSON compatibility module, a local `RawJson` type, a wire-model
layer, a discriminator probe followed by a second parse, or a JSON DOM.

## Ordinary Objects Stay Ordinary

Brian generically decodes ordinary objects and skips unknown fields by default. If a result contains
a heterogeneous output union, only that nested union needs a custom reader:

```nim
type
  Result* = object
    id*: string
    status*: Status
    output*: seq[Output]
    usage*: Option[Usage]

let value = fromJson(body, Result)
```

Do not write `readJson(Result, ...)` merely because `Output` is a `oneOf`. Brian finds the
`readJson(Output, ...)` overload while decoding `seq[Output]`. Likewise, let Brian generically decode
ordinary nested objects, sequences, tuples, options, and tables.

## `RawJson` Is Brian's Open Boundary

Use Brian's type directly for JSON Schema documents, arbitrary metadata, and deliberately unmodelled
extension payloads:

```nim
import brian

type Tool = object
  name: string
  schema: RawJson

const EmptyObjectSchema =
  RawJson("""{"type":"object","properties":{}}""")
```

The standard operations already have the desired ergonomics:

- `$value` returns the stored JSON text for both `RawJson` and `CanonRawJson`.
- `writeJson(w, value)` and `toJson(value)` emit the raw value as JSON, not as a quoted JSON string.
- `fromJson(input, RawJson)` validates and captures one complete value without materializing a DOM.
- `CanonRawJson` is only for deliberately normalized, whitespace-free re-emission. It is not a more
  typed form of `RawJson`.

Manually constructing `RawJson` trusts the supplied bytes. Use it only with one complete valid JSON
value. `RawJson("")` is useful as an omission sentinel, but it is not itself valid JSON and must not
reach `writeJson`:

```nim
proc writeJson(w: var JsonWriter; value: Params) =
  w.write "{"
  if ($value.metadata).len > 0:
    w.write "\"metadata\":"
    writeJson(w, value.metadata)
  w.write "}"
```

For an ergonomic helper that accepts either already-raw JSON or a typed Nim value, keep the
`RawJson` overload canonical and serialize the typed value exactly once:

```nim
proc formatSchema(name: sink string; schema: sink RawJson): Format =
  Format(name: name, schema: schema)

proc formatSchema[T](name: sink string; schema: T): Format =
  formatSchema(name, RawJson(toJson(schema)))
```

Never use `RawJson(toJson(toJson(value)))`: the second `toJson` turns JSON text into a JSON string.
Do not define `$`, `string` converters, aliases, or wrapper objects around Brian's raw types.

## Discriminator Unions: One Pass at the Union Boundary

For a server-owned evolving `oneOf`, use the same shape as the Responses output model:

1. Keep the wire discriminator as a typed field.
2. Use a separate private variant discriminator for Nim storage.
3. Give each application-used branch a typed payload.
4. Keep one opaque arm for unknown server-owned branches when callers need forward compatibility.
5. Parse the object once with `p.jsonFields`; do not parse a probe and then parse the same JSON again.

This reduced example models message and function-call branches:

```nim
import std/tables
import brian

type
  OutputKind* {.pure.} = enum
    unknown = ""
    message
    function_call

  OutputPart* = object
    `type`*: string
    text*: string

  OutputMessage* = object
    role*: string
    content*: seq[OutputPart]

  OutputFunctionCall* = object
    call_id*: string
    name*: string
    arguments*: string

  OutputShape = enum
    outputMessage
    outputFunctionCall
    outputOpaque

  Output* = object
    id*: string
    status*: string
    `type`*: OutputKind
    case shape: OutputShape
    of outputMessage:
      message*: OutputMessage
    of outputFunctionCall:
      functionCall*: OutputFunctionCall
    of outputOpaque:
      extraFields*: RawJson

proc readJson(dst: var OutputKind; p: var JsonParser;
    unknownFields: UnknownFieldPolicy) =
  var value: string
  readJson(value, p, unknownFields)
  case value
  of "message": dst = OutputKind.message
  of "function_call": dst = OutputKind.function_call
  else: dst = OutputKind.unknown

proc readJson(dst: var Output; p: var JsonParser;
    unknownFields: UnknownFieldPolicy) =
  var id, status, role, callId, name, arguments: string
  var kind = OutputKind.unknown
  var content: seq[OutputPart]
  var extra = initOrderedTable[string, RawJson]()

  for fieldName in p.jsonFields:
    case fieldName
    of "id": readJson(id, p, unknownFields)
    of "status": readJson(status, p, unknownFields)
    of "type": readJson(kind, p, unknownFields)
    of "role": readJson(role, p, unknownFields)
    of "content": readJson(content, p, unknownFields)
    of "call_id": readJson(callId, p, unknownFields)
    of "name": readJson(name, p, unknownFields)
    of "arguments": readJson(arguments, p, unknownFields)
    else:
      var value: RawJson
      readJson(value, p, unknownFields)
      extra[fieldName] = move(value)

  case kind
  of OutputKind.message:
    if unknownFields == ufReject and extra.len > 0:
      p.raiseParseError("unexpected field for message output")
    dst = Output(
      id: id,
      status: status,
      `type`: kind,
      shape: outputMessage,
      message: OutputMessage(role: role, content: move(content))
    )
  of OutputKind.function_call:
    if unknownFields == ufReject and extra.len > 0:
      p.raiseParseError("unexpected field for function-call output")
    dst = Output(
      id: id,
      status: status,
      `type`: kind,
      shape: outputFunctionCall,
      functionCall: OutputFunctionCall(
        call_id: callId,
        name: name,
        arguments: arguments
      )
    )
  of OutputKind.unknown:
    dst = Output(
      id: id,
      status: status,
      `type`: kind,
      shape: outputOpaque,
      extraFields:
        if extra.len == 0: RawJson("")
        else: RawJson(toJson(extra))
    )
```

This preserves the important compatibility behavior of the full pattern: a known typed branch
rejects truly extra fields under `ufReject`, while an unknown branch treats its unmodelled fields as
its opaque payload. Under normal `ufSkip`, known branches remain forward-compatible.

Only add the opaque arm if the selected API and application need to retain unknown branch data. If
they only need to ignore unknown branch fields, use a smaller representation that lets Brian skip
them. Do not turn stable used branches into `RawJson`.

## Token-Kind Unions

When `oneOf` arms differ by JSON token kind rather than an object discriminator, use Brian's direct
README pattern:

```nim
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
```

The matching writer switches on the same variant discriminator and delegates each active payload to
Brian's `writeJson`. Do not serialize inactive fields.

## Custom Reader and Writer Rules

- Every custom reader accepts `unknownFields: UnknownFieldPolicy` and forwards it to nested reads.
- Use `p.jsonFields` for objects, `p.kind` for token-shape unions, and `p.skipJson()` only for values
  intentionally ignored.
- Use `w.write` for fixed JSON punctuation/literals, `w.escapeJson` for dynamic keys or strings, and
  `writeJson` for values.
- Custom-read only shapes Brian cannot map generically: genuine unions, tolerant evolving enums,
  positional wire shapes, or unusual envelopes.
- Request unions get custom writers that emit only the active arm. Ordinary response/result objects
  remain generic even when a nested member has a custom reader.
