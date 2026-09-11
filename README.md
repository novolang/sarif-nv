# sarif-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

SARIF 2.1.0 — the static-analysis interchange format continuous
integration reads — as a typed value, written to and read from the
standard library's JSON value, with a builder shaped for a compiler.

- `sarifref` — the indices a result points through, and the level rule
  the format states and emitters break;
- `sarifresult` — one finding: regions, locations, code flows, fixes,
  suppressions;
- `sarifrun` — the log, the run, the tool and its rules, the artifact
  table;
- `sarifbuild` — the builder, and the only place an index is minted;
- `sarifjson` — both directions, and the checks that belong beside them;
- `sariferr` — what a read refuses, and what it only reports.

```
novo pkg add sarif-nv
novo pkg build
novo test
```

## The one example that will work

A compiler has a diagnostic with a code, a severity, a path, a span and
a sentence. That is the whole of what it has, and that is the whole of
what this asks for.

```novo ignore
use sarifbuild
use sarifresult
use sarifjson

fn log_of(diags: [Diag]) -> Str
    var b = sarifbuild.builder("novo", "0.8.9")
    for d in diags
        b = sarifbuild.add_result(b, d.code, d.level, d.file,
                                  sarifresult.text_region(d.line, d.col,
                                                          d.line, d.end_col),
                                  d.message)
    sarifjson.write_text(sarifbuild.finish_log(b))
```

Ten thousand diagnostics over two hundred files produce two hundred
URIs, because `add_result` interns the path.

## The load-bearing interface: the index, as a type

```novo ignore
pub struct SarifArtifactRef
    index: Int
```

A SARIF result does **not** carry the path of the file it is about. It
carries an index into `run.artifacts`, and the artifact at that index
carries the URI. The rule works the same way: a result carries a
`ruleIndex` into `tool.driver.rules`, where the identifier, the help
text and the default level live.

That indirection is the whole reason SARIF is a format a compiler can
emit for a project rather than one diagnostic at a time. An emitter
that repeats the URI per result produces a document that validates,
means the same thing, and is many times the size — which is how a CI
upload starts being refused for size, and is the single most common way
a hand-rolled SARIF writer goes wrong.

So the index is a **type and not an `Int`**, and `sarifbuild` — the
module that owns the tables — is the only place either ref is produced.
The only ways to obtain one are `artifact` and `add_result`, both of
which fill the table as a side effect of answering:

```novo ignore
pub fn artifact(b: SarifBuilder, uri: Str) -> (SarifBuilder, SarifArtifactRef)
```

Answering the builder *and* the ref together is what makes it
impossible to get a ref without filling the table. A signature that
answered only the ref would have needed a mutable table — `[mutate]` is
a host effect and this is a `core` package — and one that answered only
the builder would have made the caller search for the index it had just
created.

A result naming artifact seven of a run with three artifacts is
therefore not a mistake this package can make.
`sarifref.no_artifact()` is the single exception, and it is the absent
ref, which every table admits.

## The second decision worth arguing: two error types

`sariferr` declares `SarifError` and `SarifProblem`, and the split is
the interesting one.

**`SarifError` is a refusal.** The bytes are not a SARIF 2.1.0 log and
there is nothing to hand back. It carries an RFC 6901 JSON Pointer,
because a reader staring at a 40 MB log needs to be told
`/runs/0/results/318/locations/0` rather than "invalid document". The
variant worth naming is `SarifIndexOutOfRange`: `ruleIndex: 7` in a run
with three rules is well-formed JSON that matches the schema and means
nothing, so it is the one class of fault a schema validator passes and
a reader cannot.

**`SarifProblem` is a finding about a log that parsed.** A result whose
`ruleId` and `ruleIndex` disagree, a `pass` carrying `level: "error"`, a
fix whose replacement names no region and therefore deletes the file,
two artifacts that are two spellings of one path. None of those stop a
document being read, and every one is a bug in whatever wrote it.

They are two types because a validator that *raised* on the second
class would be unusable against real logs — the tools that produce
SARIF break these rules constantly — and a reader that *ignored* the
second class would silently pass their damage on. So `sarifjson.read`
refuses the first and does not run the second, and
`sarifjson.problems` answers a list, always, and the caller decides.

## The rule the specification states and emitters break

```novo ignore
pub fn level_ok(k: SarifKind, l: SarifLevel) -> Bool
```

§ 3.27.10: `level` is meaningful only when `kind` is `fail`, and every
other kind must carry `none`. The damage is not the validation failure
— it is the consumer that reads one field and not the other and reports
a passing check as a failure. `sarifbuild.add_result_kind` routes every
level through `level_for_kind`, so a `pass` cannot leave this package
carrying `error`.

## The third: a region has three coordinate systems

§ 3.30 lets a region be a text region (1-based `startLine` /
`startColumn`), a character region (0-based `charOffset` /
`charLength`), or a binary region (`byteOffset` / `byteLength`), and a
document may carry more than one for the same range. The trap is that a
consumer picks whichever it understands: an emitter that wrote a byte
offset into `charOffset` produces a document that validates and
highlights the wrong text in every viewer that prefers character
regions.

`SarifRegion` carries all three with their own absent values, and
`region_has_text` / `region_has_chars` / `region_has_bytes` are how a
caller asks which are real. Columns are 1-based and offsets are
0-based, which is the specification's own asymmetry rather than this
package's, and `region_from_span` is published precisely because
converting a half-open byte span by hand is where the off-by-one lives.

## What is declared, and what is outside

**Declared.** `runs`, `tool` with `driver` and `extensions`, `rules`
with their descriptions and help URIs, `artifacts` with roles and URI
bases, `logicalLocations`, `results` with `kind`, `level`, `message`
and its Markdown twin, `locations` and `relatedLocations`, `regions` in
all three coordinate systems with snippets, `codeFlows` /
`threadFlows` / `threadFlowLocations`, `fixes` / `artifactChanges` /
`replacements`, `suppressions` with kind and status,
`baselineState`, `partialFingerprints`, `invocations`,
`originalUriBaseIds`, and `properties` bags at every level that has
one.

**Outside, and each for a reason.**

| object | why |
| --- | --- |
| `graphs`, `graphTraversals` | a general graph model with its own node and edge identity, used by almost nothing; a taint analysis that wants one is better served by `codeFlows`, which every consumer renders |
| `webRequest`, `webResponse` | for a dynamic scanner replaying HTTP; nothing on this grid produces one, and modelling it would mean a second HTTP message type beside http-codec-nv's |
| `conversion` | records that the log was converted from another tool's format. This package converts nothing; a converter built on it writes the member itself |
| `externalPropertyFileReferences` | splits one log across several files, which is a strategy for logs larger than a consumer will load — and a reader for it would have to open files, which a `core` package cannot |
| `taxonomies`, `translations`, `policies` | rule metadata for a multi-tool, multi-language deployment; a tool with one rule set and one language has no use for any of them, and each is a table with its own reference discipline |
| `addresses` | memory addresses, for a binary analyser |
| `stacks` | a call stack per result. `codeFlows` covers what a source-level tool has, and a stack without an address model is half the object |
| `attachments`, `newlineSequences`, `specialLocations` | small optional members that no consumer this package targets reads |
| `automationDetails`, `runAggregates` | how a run relates to other runs of the same pipeline — a fleet concern, and one whose identifiers are the CI system's rather than the tool's |

**A read does not silently drop them.** `sarifjson.unknown_members`
answers the JSON Pointers of every member a read had no field for, so a
round trip through this package can be *shown* to be lossy rather than
discovered to be. Adding an object later is then a compatible change:
the pointers it used to report stop appearing.

## The layer, and why

`core`. A SARIF log is a value the caller assembles out of diagnostics
it already holds, and writing one is a conversion to the standard
library's JSON value: nothing is read, no file is opened, no clock is
consulted.

The clock is the interesting one. `invocation.startTimeUtc` is a
timestamp and a `core` package has none, so the timestamps are strings
the caller supplies, `sarifrun.timestamp_ok` checks them against
SARIF's own RFC 3339 profile, and a consumer that wants them typed
parses them with calendar-nv itself. That is not a workaround: it is
what makes a log reproducible, the same way tar-nv and zip-nv default
their headers to a fixed epoch.

The one place a stream could have entered is writing a large log out,
and that is the effect-polymorphic shape:

```novo ignore
pub fn write_to<W: Write[e]>(w: W, log: SarifLog) -> ?IoError [e]
```

The clause is `[e]`, bound by the caller's `Write` impl, so a file
costs `[fs]` and an in-memory buffer costs nothing — and this package
has spent neither. It is worth having rather than `write_text` plus the
caller's own write, because a log of a large analysis is tens of
megabytes and building the whole string first is the allocation this
avoids.

## `@tier(embedded)` is not claimed

Deliberately. A SARIF log is a document of lists of lists, and the
tables it is built around are the two things a device has no room for.
There is no device consumer of a static-analysis interchange format,
and a claim here would be one the probe could only keep by never
allocating in a package whose whole job is building tables.

## The reference implementation

`serde-sarif` (Rust), whose typed model is the one ported: the
`SarifLog` / `Run` / `Tool` / `ToolComponent` / `ReportingDescriptor`
split, `Result` with `kind` and `level` apart, and the builder pattern
its `sarif-fmt` and its converters use.

Two things differ, on purpose. `serde-sarif` derives its types from the
published JSON schema and therefore declares every object, including
the nine above; this package declares what a source-level tool
produces and says which objects it does not. And `serde-sarif` leaves
the artifact table to the caller — its converters each write their own
interning loop — where here the interning is the builder's whole job
and the ref type is what makes it unavoidable.

The vectors are the SARIF Technical Committee's own example documents
from the specification's appendices, the `sarif-tutorials` sample logs,
and the two documents that exercise the rules in § The rule the
specification states and emitters break and § The third: `microsoft/sarif-sdk`'s
validator test corpus is where those live.

## The consumers, and what adopting this would take

**`novo build --review` / `novo ci`** (`compiler/bin/novo.ml`,
`process_review_findings`) reads an LLM's findings as a JSON array of
`{line, severity, message, category}` and prints
`path:line: severity: [category] message` to stderr. That is a SARIF
run with one artifact, one rule per category, and one result per
finding, and the reason to write it as one is that nothing consumes the
stderr form: a pull request cannot annotate a line from it, and a
pipeline cannot compare two runs of it.

**`novols`** (`compiler/lsp/novols.ml`) already carries the harder
half. Its `diagnostic` record is `{line, col, end_col, severity,
message, code, source}` — a text region, a level and a rule identifier
— and `diag_to_json` writes it as LSP. The same record writes as a
SARIF result with `sarifresult.text_region` and `add_result`, which is
what turns "the editor shows squiggles" into "the CI job annotates the
diff", from one diagnostic pipeline rather than two. LSP severities are
1..4 and SARIF's are four names, and the mapping is the one place a
line of code is needed.

**`novo doc --errors --json`** writes `ERRORS.json`: every `error_kind`
in the tree with its code, sorted. That is `tool.driver.rules` —
`sarifbuild.set_rule` per entry — and it is what makes a SARIF log
this project emits carry help text instead of bare codes.

**`novo bugs list --json`** is not a consumer and is named so the
absence is on the record: a bug tracker's entries are not results of an
analysis run, and pressing them into `result` would produce a log whose
`ruleId`s are issue numbers.

**A shard audit run** (`scripts/shard_audit.sh`) emits rows with a
name, a verdict and a site list. Every row is a rule and every site is
a result, which would make the audit's output something a pull request
could annotate rather than something a person reads in a terminal.

## What a row wanted to widen

Nothing widened. Every function in this package is `[]` except
`write_to`, whose row is its caller's.

Three findings from the lane:

1. **The plan's note calls this "the lint interchange format continuous
   integration reads", and the reading half is the smaller half.** What
   the consumers above need is the *writing* half plus the artifact
   table; the reader exists for a baseline comparison, which is the one
   thing a tool does with a log it did not write.
2. **The clock is the row's only real tension with `core`, and it
   resolves.** `invocation` is the one SARIF object that wants a
   timestamp. Taking calendar-nv would not have helped — a `core`
   package still has no way to know what time it is — so the timestamps
   are the caller's strings with a check beside them, and the package
   stays dependency-free.
3. **`novo lint` does not exist.** The brief names it as a consumer; the
   toolchain's commands are `novo build --review`, `novo ci` and
   `novo review show`, and the AI review pass is what produces
   findings. The consumers section above names what is actually there.

## The surface

| module | `pub fn` | `pub struct` | `pub enum` |
| --- | --- | --- | --- |
| `sarifref` | 18 | 4 | 5 |
| `sarifresult` | 22 | 13 | 0 |
| `sarifrun` | 17 | 9 | 0 |
| `sarifbuild` | 26 | 1 | 0 |
| `sarifjson` | 17 | 0 | 0 |
| `sariferr` | 4 | 0 | 2 |
| **total** | **104** | **27** | **7** (33 variants) |

One `impl Error` block, for `SarifError`.
