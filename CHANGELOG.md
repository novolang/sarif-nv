# Changelog

All notable changes to sarif-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
