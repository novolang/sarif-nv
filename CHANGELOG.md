# Changelog

All notable changes to sarif-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
