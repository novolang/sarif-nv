# sarif-nv

The Static Analysis Results Interchange Format (SARIF) is, in the words of its
[OASIS specification](https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html),
"a standard, vendor-neutral format for the output of static analysis tools". A
tool writes one JSON document; a code-scanning service, a pull request or an
editor reads it. This package brings SARIF 2.1.0 to novo-lang as a typed value,
written to and read from the JSON value the standard library's `std.json`
produces, with a builder shaped for a compiler.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

A SARIF **log** is one JSON document. It holds one or more **runs**, and a run
is one execution of one analysis tool. A run names the **tool** that produced
it, lists the **artifacts** it looked at, lists the **rules** it applied, and
carries the **results** it found.

A **result** is one finding. It has a **rule**, a **kind**, a **level**, a
**message** and one or more **locations**. The kind says what the result is:
`fail` for a rule that was violated, `pass` for one that was checked and not
violated, and four others. The level says how serious a failure is: `error`,
`warning`, `note`, or `none`.

A result does not carry the path of the file it is about. It carries an
**index** into the run's artifact table, and the artifact at that index carries
the URI. A result's rule works the same way: it carries an index into the tool's
rule table, where the identifier, the help text and the default level live. This
indirection is what makes SARIF a format for a whole project rather than one
diagnostic at a time. Ten thousand findings over two hundred files carry two
hundred URIs.

A **region** is a range inside an artifact. SARIF gives a region three
independent coordinate systems, and a document may carry more than one of them
for the same range.

| Coordinate system | Fields | Base |
| --- | --- | --- |
| Text | `startLine`, `startColumn`, `endLine`, `endColumn` | 1-based |
| Character | `charOffset`, `charLength` | 0-based |
| Binary | `byteOffset`, `byteLength` | 0-based |

A **JSON Pointer**, defined by [RFC 6901](https://www.rfc-editor.org/rfc/rfc6901),
is a string such as `/runs/0/results/318/locations/0` that names one place
inside a JSON document. Every refusal this package produces carries one.

## Install

```
novo pkg add sarif-nv
```

## Example

```novo
use sarifref
use sarifresult
use sarifbuild
use sarifjson

fn main() [io]
    // A builder for the tool whose findings these are.
    var b = sarifbuild.builder("novo", "0.8.9")

    // One diagnostic: a rule identifier, a level, a path, a span and a
    // sentence. add_result interns the path into the run's artifact table
    // and puts the index in the result.
    b = sarifbuild.add_result(b, "E2004", SarifLevelError, "src/parser.nv",
                              sarifresult.text_region(13, 1, 13, 9),
                              "no field `nmae` on `Token`")

    // A second finding in the same file. The table still holds one artifact.
    b = sarifbuild.add_result(b, "E2004", SarifLevelWarning, "src/parser.nv",
                              sarifresult.text_region(41, 5, 41, 9),
                              "unused binding `tok`")

    // Two results, one artifact.
    println("${sarifbuild.artifact_count(b)}")

    // The whole log as JSON text, ready to upload.
    println(sarifjson.write_text(sarifbuild.finish_log(b)))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails on
purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `sarifref` | The two index types a result points through, the five enumerations the format fixes, the property bag, and the rule that ties a level to a kind. |
| `sarifresult` | One finding: regions in all three coordinate systems, physical and logical locations, code flows, fixes, suppressions, fingerprints and the baseline field. |
| `sarifrun` | The log, the run, the tool and its rules, the artifact table, the logical locations, the invocations and the URI bases, and the questions a gate asks a run. |
| `sarifbuild` | The builder. It owns the tables, interns a path or a rule identifier into an index, and is the only place an index is produced. |
| `sarifjson` | Writing a log to a JSON value or a string, reading one back, writing one into a sink, and the checks that report what is wrong with a log that parsed. |
| `sariferr` | The two error types. One is a refusal to read. The other is a finding about a log that read successfully. |

## How to choose an entry point

**A tool emitting its own diagnostics uses `sarifbuild` and nothing else.**
`builder` starts a run, `add_result` takes the five things a diagnostic already
has, and `finish_log` hands back the log. `sarifjson.write_text` turns it into
the bytes to upload.

**A tool reading a log somebody else wrote uses `sarifjson.read` or
`.read_text`.** Both answer a `Result`, and the refusal carries a JSON Pointer.
`sarifjson.problems` then answers everything that is wrong with the log without
refusing it.

**A gate that fails a build asks `sarifrun`.** `run_is_clean` is the whole
question: a run with no results and a failed invocation is not clean.
`sarifrun.failing` answers the results at or above a level, and
`sarifrun.new_results` answers the ones the baseline has not seen.

**`sarifjson.write_to` writes into any sink implementing `Write`.** A log of a
large analysis is tens of megabytes, and this avoids building the whole string
first. A file costs the file's effects and an in-memory buffer costs none.

**`sarifbuild.add_result_kind` is `add_result` for a result that is not a
failure.** It routes the level through `sarifref.level_for_kind`, so a `pass`
cannot leave this package carrying `error`.

## The rules a user needs

1. **A result names its artifact and its rule by index, never by path or
   identifier alone.** The index is a type, `SarifArtifactRef` or
   `SarifRuleRef`, with no public constructor. SARIF 2.1.0 sections 3.28.3 and
   3.27.6.
2. **The only way to obtain an index is to intern.** `sarifbuild.artifact` and
   `sarifbuild.add_result` fill the table and answer the index together, so a
   result naming an artifact the run does not have cannot be written.
3. **`sarifref.no_artifact()` is the absent reference.** Its index is -1, and
   every table admits it. A result about the project as a whole carries it.
4. **A rule index is relative to its tool component.** `SarifRuleRef.component`
   is 0 for the driver and 1-based into `tool.extensions`. A plugin's rule 3 and
   the driver's rule 3 are two rules. SARIF 2.1.0 section 3.27.7.
5. **`level` means something only when `kind` is `fail`.** Every other kind must
   carry `SarifLevelNone`. `sarifref.level_ok` is that question and
   `sarifref.level_for_kind` is the correction. SARIF 2.1.0 section 3.27.10.
6. **The three coordinate systems of a region never become each other.** Columns
   are 1-based and offsets are 0-based, which is the format's own asymmetry.
   `region_has_text`, `region_has_chars` and `region_has_bytes` say which are
   real. `region_from_span` converts a half-open byte span. SARIF 2.1.0 section
   3.30.
7. **A region with nothing set means the whole artifact.**
   `sarifresult.whole_artifact` builds one and `region_is_whole` recognises one.
8. **A fix whose replacement names no region replaces the whole artifact.**
   Applying such a fix deletes the file. `sarifresult.fixes_are_bounded` is the
   check, and `sarifjson.problems` reports the fault. SARIF 2.1.0 section 3.57.
9. **A run owns its tables, so an index from one run means nothing in another.**
   `sarifbuild.merge` re-interns rather than concatenating.
10. **Reading refuses an index that points outside its table and only reports a
    `ruleId` that disagrees with the rule at its index.** The first is
    well-formed JSON that means nothing. The second is a document real tools
    produce every day.
11. **Every call answers a new builder.** A builder is threaded, not mutated, so
    `b = sarifbuild.add_result(b, …)` is the shape of every line. A builder can
    therefore be forked, which is what two analyses over one artifact table need.
12. **Timestamps are strings the caller supplies.** Nothing here consults a
    clock. `sarifrun.timestamp_ok` checks a string against SARIF's own RFC 3339
    profile. SARIF 2.1.0 section 3.9.
13. **A URI is kept exactly as the caller wrote it.** SARIF resolves a relative
    URI against `originalUriBaseIds` at read time, by whoever consumes the log,
    so nothing here normalises one. `sarifrun.uri_base_missing` says when the
    base a URI needs is absent.
14. **Two artifacts that are two spellings of one path are reported, not
    merged.** Interning is by byte equality, because deciding two paths are one
    means knowing what they are relative to.
    `sarifjson.duplicate_artifacts` answers them.
15. **A writer omits every absent value.** An empty string, an empty list, an
    index of -1 and a region with nothing set are all left out. `version` and
    `$schema` are always written.
16. **Reading is depth-bounded.** A log arrives from a build service, and code
    flows nest. `sarifjson.default_depth` is the bound and
    `sarifjson.read_with_depth` replaces it.

## What is not included

Nine SARIF objects are outside this package. A read does not drop them
silently: `sarifjson.unknown_members` answers the JSON Pointer of every member
a read had no field for.

| Object | Reason |
| --- | --- |
| `graphs`, `graphTraversals` | A general graph model with its own node and edge identity. A taint analysis is served by `codeFlows`, which every consumer renders. |
| `webRequest`, `webResponse` | For a dynamic scanner replaying HTTP. Modelling them would mean a second HTTP message type. |
| `conversion` | Records that a log was converted from another tool's format. This package converts nothing. |
| `externalPropertyFileReferences` | Splits one log across several files. Reading them means opening files, and nothing here opens a file. |
| `taxonomies`, `translations`, `policies` | Rule metadata for a deployment with several tools and several languages. Each is a table with its own reference discipline. |
| `addresses` | Memory addresses, for a binary analyser. |
| `stacks` | A call stack per result. Without an address model it is half the object, and `codeFlows` covers what a source-level tool has. |
| `attachments`, `newlineSequences`, `specialLocations` | Small optional members no consumer of a source-level tool's log reads. |
| `automationDetails`, `runAggregates` | How a run relates to other runs of the same pipeline. The identifiers belong to the continuous-integration system, not the tool. |

Also absent:

- **A clock.** `invocation.startTimeUtc` is a timestamp, and nothing here can
  know what time it is. A log whose timestamps the caller supplies is a log that
  can be reproduced.
- **URI normalisation.** See rule 13.
- **Running on a microcontroller.** A SARIF log is a document of lists of lists
  built around two growing tables, and no device consumes a static-analysis
  interchange format.
- **Dependencies.** The JSON value is the standard library's, so a caller holds
  one JSON value rather than two.

## Related packages

- [jsonquery-nv](https://novo-lang.org/packages/jsonquery-nv) runs a jq program
  over a JSON value. It is how a reader queries a log this package wrote without
  reading it back into typed values.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) parses an RFC 3339
  timestamp into a typed instant. A consumer that wants SARIF's timestamps typed
  uses it directly; `sarifrun.timestamp_ok` only says whether a string conforms.
- `std.json` in the standard library parses and renders JSON. `sarifjson.write`
  answers its value and `sarifjson.read` takes one, so a caller serialises with
  `json.stringify` or `json.pretty`.

## Tests

```bash
novo test tests                            # every suite
novo test tests/sarifbuild_tests.nv        # the builder, and the interning
novo test tests/sarifresult_tests.nv       # the three coordinate systems
novo test tests/sarifrun_tests.nv          # the questions a gate asks a run
novo test tests/sarifjson_tests.nv         # both directions, and the two error types
```

`novo test` fails on purpose today. Every assertion reaches a `not implemented:
sarif-nv.<module>.<fn>` panic, because every body is a `todo()`. The tests are
the specification the implementation will have to satisfy.

The vectors are the SARIF Technical Committee's own example documents from the
specification's appendices and the sample logs of `sarif-tutorials`. The cases
that exercise a rule an emitter breaks come from the validator test corpus of
`microsoft/sarif-sdk`. The typed model is ported from `serde-sarif`, the Rust
implementation.

The suite asserts that six results over two files produce two artifacts, that a
region built from a byte span does not answer true to `region_has_chars`, that a
run with no results and a failed invocation is not clean, and that reading
refuses an out-of-range index while only reporting a `ruleId` that disagrees
with its index.

## Implementation status

| Item | Implemented |
| --- | --- |
| `sarifref.SarifArtifactRef`, `.SarifRuleRef`, `.SarifLogicalRef`, `.SarifProperty` | declared |
| `sarifref.SarifLevel`, `.SarifKind`, `.SarifBaseline`, `.SarifSuppressionKind`, `.SarifSuppressionStatus` | declared |
| `sarifref`'s eight name and name-from-string functions | no |
| `sarifref.level_ok`, `.level_for_kind` | no |
| `sarifref.ref_absent`, `.no_artifact`, `.no_rule`, `.rule_ref_absent`, `.rule_ref_eq` | no |
| `sarifref.property`, `.with_property`, `.property_tags` | no |
| `sarifresult`'s thirteen types, from `SarifRegion` to `SarifReplacement` | declared |
| `sarifresult`'s five region builders and five region predicates | no |
| `sarifresult.location`, `.location_with_message`, `.logical_location` | no |
| `sarifresult.no_baseline`, `.baseline`, `.primary_location`, `.is_reported`, `.fails_at` | no |
| `sarifresult.fingerprint`, `.touched_artifacts`, `.fixes_are_bounded` | no |
| `sarifrun`'s nine types, from `SarifLog` to `SarifUriBase` | declared |
| `sarifrun.sarif_version`, `.schema_uri`, `.empty_log` | no |
| `sarifrun.artifact_of`, `.rule_of`, `.logical_of`, `.rule_by_id` | no |
| `sarifrun.artifact_uri`, `.resolve_uri`, `.uri_base_missing` | no |
| `sarifrun.result_count`, `.level_counts`, `.failing`, `.new_results` | no |
| `sarifrun.timestamp_ok`, `.run_is_clean`, `.invocations_succeeded` | no |
| `sarifbuild.SarifBuilder` | declared |
| `sarifbuild.builder`, `.builder_with` | no |
| `sarifbuild.artifact`, `.artifact_under`, `.rule`, `.set_rule`, `.extension`, `.rule_of_component`, `.logical`, `.uri_base` | no |
| `sarifbuild.add_result`, `.add_result_kind`, `.add`, `.add_global` | no |
| `sarifbuild.with_fix`, `.with_fingerprint`, `.with_baseline`, `.with_suppression`, `.invocation`, `.set_property` | no |
| `sarifbuild.finish`, `.finish_log`, `.log_of`, `.merge`, `.artifact_count`, `.rule_count` | no |
| `sarifjson.default_depth`, `.write`, `.write_run`, `.write_text`, `.write_pretty`, `.write_to` | no |
| `sarifjson.read`, `.read_with_depth`, `.read_text`, `.read_run`, `.round_trip`, `.unknown_members` | no |
| `sarifjson.problems`, `.run_problems`, `.result_problems`, `.duplicate_artifacts`, `.is_sound` | no |
| `sariferr.SarifError`, `.SarifProblem` and `impl Error for SarifError` | declared |
| `sariferr.problem_message`, `.problem_is_serious`, `.error_pointer`, `.render` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
