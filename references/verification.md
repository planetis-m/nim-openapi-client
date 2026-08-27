# Verification and Review

Use this reference before handoff or when auditing whether a minimal client is sufficient.

## Protocol Tests

Keep tests deterministic, offline, and focused on observable wire behavior.

For every used request operation, test:

- method, URL/path/query encoding, authentication and content headers;
- exact required JSON and fixed literals, including byte order/newlines when observable;
- omission of each default/sentinel;
- every constructed union branch and explicit non-default override;
- multipart/form/binary framing, filename/header injection, and boundary safety where applicable.

For every used response operation, test:

- a realistic success fixture with nested unknown fields;
- null and omitted fields that affect application control flow;
- each used union arm and a future unknown output literal/branch where compatibility is promised;
- error envelopes and non-success status handling;
- malformed JSON, known-field type mismatch, and required semantic accessor failures;
- pagination cursors and raw downloads where applicable.

Use normal tolerant decoding for production fixtures. Decode selected schema fixtures directly with
`unknownFields = ufReject` when exact-shape validation is meaningful. Strict validation should not
replace forward-compatible public parsing.

## Coverage Audit

Compare the completed code against both directions:

1. Application -> client: every imported symbol, endpoint call, constructed field, read field,
   accessor, and failure branch remains supported.
2. Client -> OpenAPI: every modeled method, path, parameter, literal, property, status, and union arm
   has a source in the selected specification.

Then search for stale SDK imports, package names, compiler paths, feature defines, compatibility
aliases, locally redefined Brian types, and unused client exports.

## Build and Reproducibility

- Compile the main application entry points, not only isolated schema tests.
- Run the repository's full deterministic suite in its normal configurations.
- Run focused Brian tests if the work extends Brian itself.
- Run `git diff --check` in every changed repository.
- Ensure new client modules and tests are tracked and generated binaries are not.
- Verify the dependency lock pins the exact Brian commit containing required behavior.
- Do not claim a clean fresh-checkout build while relying on uncommitted or unpinned dependency code.

## Performance Comparison

Benchmark only when performance matters or the user asks. To measure the value of minimal models:

- Use the same Brian version, compiler mode, payload, allocator, and consumption pattern for both
  models; do not compare a minimal Brian model against a full model from another JSON library.
- Model all complete-result fields present in the fixture on the full side and the actual local type
  on the minimal side.
- Use realistic small, medium, and large payloads. Alternate measurement order, warm up first, and
  report medians across multiple process runs.
- Distinguish throughput gain (`full/minimal`) from decode-time reduction
  (`(full-minimal)/full`).
- State that absent schema branches have no materialization cost and that I/O or envelope parsing
  dilutes response-body-only improvements.

## Handoff

Report the supported endpoint subset, intentionally opaque boundaries, compatibility behavior,
dependency changes, tests/configurations run, and any OpenAPI ambiguity that required an explicit
assumption. Do not describe omitted unrelated API surface as unfinished work.
