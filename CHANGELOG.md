# Changelog

All notable changes to sarif-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-11

The **implementation**: every body the 0.0.1 interface stubbed. The
signatures, the effect rows and the four published suites are unchanged;
`stability` stays `experimental`, and the release is `implemented`.

### Added

- Every body under `src/`: the builder's interning and `merge`, the
  region and location constructors, the eight checks `sarifjson.problems`
  runs over a parsed log, and the `serde`-shaped writer and reader over
  `std.json`'s `JsonValueH`.
- `tests/sarifapi_tests.nv` — the calls a suite driven by a worked
  example does not reach: the other spellings of every enumeration, the
  region constructors that are not the one the example uses, and the
  reads a gate makes over a run.
- `tests/sarifedge_tests.nv` — one run with every optional member set,
  pinned as a write-read-write round trip; the documents the reader
  refuses, each with the JSON Pointer it refuses them at; and the faults
  `problems` reports about a log that parsed anyway.

### Changed

- `sarifjson`'s reader threads the run's artifact table as a `var`
  parameter rather than answering it beside the value, and
  `sarifresult.location` builds its structs unqualified. Both are
  workarounds for compiler defects and both are sighted where they
  appear; see below.
- The manifest's argument against a `url-nv` dependency credited a
  `sarifuri` module that has never existed; it now names the three calls
  that answer a caller's questions about a URI — `artifact_uri`,
  `resolve_uri` and `uri_base_missing`.
- Five unreachable guards were removed rather than excused: the
  `let Some` arm in `too_deep_object` that `json.get` never takes, and
  the `is_object` checks in `read_tool`, `read_tool_members`,
  `inner_rule_index` and `read_component`, whose callers have already
  checked.

### Known

- **Two regions carry `cov: skip`**, both with the reason at the site:
  `with_logical_parent`'s out-of-range guard, which only a caller that
  is not `intern_logicals` could reach, and `shape_member`'s final
  `None`, which no shape in any of its tables falls through to. The
  publish gate reports both as excused.
- **`problem_is_serious` answers `true` for `SarifBadTimestamp`**, the
  one variant the function's own sentence does not name. A timestamp
  outside SARIF's RFC 3339 profile is worse than untidy — a build
  service refuses the log — and it is not a wrong report either; the
  reason is at the match arm. The three it answers `false` for are the
  duplicate artifact, the duplicate rule and the orphan artifact.

### Workarounds recorded here

Three compiler defects, each sighted on its filing and each carrying
`fixed_in: 0.20.3`. The installed toolchain is 0.20.2 and the floor is
`>= 0.20.2`, so every shape below stays — a closed filing is not
evidence that the toolchain in hand carries the fix, and the
qualified-struct-literal refusal was re-checked against 0.20.2 and
still refuses the literal. **Revert a workaround when the floor moves
past 0.20.2 and the defect no longer reproduces.**

- **A tuple bound out of a `Result` leaks**
  (`language/a-tuple-bound-out-of-a-result-is-leaked`). The reader's
  first shape answered `Result<(Value, [SarifArtifact]), SarifError>` to
  thread the artifact table through the location readers; a suite that
  printed every verdict green still exited 23, with the objects the
  tuples held leaked. Every reader now writes the table through a `var`
  parameter and answers one value.
- **A list read from `??` and indexed in the same expression leaks the
  element**
  (`language/a-list-read-from-qq-and-indexed-in-one-expression-is-leaked`).
  `sarifedge_tests` indexed a `json.get_list(...) ?? []` temporary in one
  expression and turned a green suite into exit 23; the test now names
  the list before indexing it.
- **A module-qualified struct literal is a parse error**
  (`language/a-qualified-struct-literal-is-a-parse-error`).
  `sarifresult.location` builds `SarifPhysical` and `SarifLocation` by
  their unqualified names, which `use` has in scope.
- Two W0002 warnings remain in `src/sarifjson.nv`: the parameter `log`
  in `write` and `problems` shadows the standard library's `log` module.
  The name is the published interface's and is kept as published; the
  two bindings that shadowed `ptr` the same way were locals and are
  named `pointer` instead.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `sarifref` — `SarifArtifactRef` and `SarifRuleRef` as types rather
  than integers, the five enums with both halves of their name
  mapping, and `level_ok` / `level_for_kind`.
- `sarifresult` — `SarifRegion` in all three of the format's coordinate
  systems, locations physical and logical, code flows, fixes with
  replacements, suppressions with a status, fingerprints and the
  baseline field.
- `sarifrun` — the log, the run, the tool and its components, the rule
  table, the artifact table, invocations and URI bases, and the reads a
  gate makes.
- `sarifbuild` — `add_result(rule_id, level, file, region, message)`,
  the interning that fills the tables, and `merge`, which re-interns.
- `sarifjson` — `write` / `read` over the standard library's
  `JsonValueH`, `write_to<W: Write[e]>`, and the eight checks.
- `sariferr` — `SarifError` for a refusal and `SarifProblem` for a
  finding about a log that parsed.

### Known

- **The index is the load-bearing interface.** A result names an
  artifact and a rule by index, `sarifbuild` is the only place either
  ref is minted, and interning is a side effect of answering one — so
  a result naming an artifact the run does not have is not a mistake
  this package can make.
- **Two error types**, because a validator that raised on a real log
  would be unusable and a reader that ignored its faults would pass
  them on.
- **`level_ok` is § 3.27.10 as a function**: a `pass` result carrying
  `level: "error"` is the fault that makes a consumer report a passing
  check as a failure.
- **A region has three coordinate systems**, all carried, each with its
  own absent value and its own predicate.
- **Nine SARIF objects are outside**, each named in the README with its
  reason, and `unknown_members` reports what a read dropped rather than
  leaving a round trip quietly lossy.
- **No dependencies.** The JSON value is the standard library's; url-nv
  is absent because SARIF resolves a URI at read time; calendar-nv is
  absent because a `core` package has no clock, and the timestamps are
  the caller's strings with `timestamp_ok` beside them.
- **`@tier(embedded)` is not claimed**, and the README says why.
- The scaffold's `src/sarif.nv` was dropped for six prefixed modules.

### Design notes

- **The consumers this interface was shaped for.** `novo build --review`
  and `novo ci` print `path:line: severity: [category] message` to
  stderr, which nothing can consume: a pull request cannot annotate a
  line from it and a pipeline cannot compare two runs of it. That is a
  run with one artifact, one rule per category and one result per
  finding. `novols`'s `diagnostic` record is already a text region, a
  level and a rule identifier, so the same record writes as a SARIF
  result through `sarifresult.text_region` and `sarifbuild.add_result`;
  the one line of code needed is the mapping from LSP's severities 1..4
  to SARIF's four names. `novo doc --errors --json` writes every
  `error_kind` with its code, which is `tool.driver.rules` through
  `sarifbuild.set_rule`. A shard audit run emits rows with a name, a
  verdict and a site list, which is a rule per row and a result per
  site.
- **`novo bugs list --json` is deliberately not a consumer.** A bug
  tracker's entries are not results of an analysis run, and pressing
  them into `result` would produce a log whose `ruleId`s are issue
  numbers.
- **`novo lint` does not exist.** The toolchain's commands are
  `novo build --review`, `novo ci` and `novo review show`, and the AI
  review pass is what produces findings.
- **The reading half is the smaller half.** What the consumers above
  need is the writing half plus the artifact table. The reader exists
  for a baseline comparison, which is the one thing a tool does with a
  log it did not write.
- **`serde-sarif` differs in two ways.** It derives its types from the
  published JSON schema and therefore declares every object, including
  the nine this package leaves out. And it leaves the artifact table to
  the caller, where here the interning is the builder's whole job and
  the reference type is what makes it unavoidable.
