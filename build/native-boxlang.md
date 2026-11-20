# Native BoxLang conversion (experimental)

The CFML sources remain authoritative. This build creates a separate native distribution under `.tmp/native-boxlang`; do not edit generated files there. Java dependencies are deliberately unchanged. MatchBox is not part of this milestone.

## Build and run

Use BoxLang JVM 1.19.0 and CommandBox (1.16.0 has the file-read defect described below):

```bash
box install --production
boxlang build/native-boxlang.bxs
cd .tmp/native-boxlang
boxlang system/runners/BoxLangRunner.bx --directory=tests.specs --reporter=agent
```

The build deletes and recreates only `.tmp/native-boxlang`. It copies the framework, installed dependencies, tests, fixtures, and report build metadata, converts CFML files, validates the generated syntax, and removes the copied CFML files. Existing native BoxLang files and static assets are copied unchanged. Cached mock sources are not copied into the distribution.

The runner currently exits with status 0 even when tests fail. Inspect the Agent reporter's `ok`, `totals.fail`, and `totals.error`; process exit status alone is not acceptance evidence.

## Conversion adjustments

`build/native-boxlang.bxs` invokes the BoxLang transpiler API. It preserves behavioral documentation annotations (including lifecycle hooks and expected exceptions), but removes the unparseable generated `@return` annotations. It also:

- Adds native component prefixes while preserving include shorthand.
- Converts references to converted filenames, discovery globs, and interpolated template extensions, without changing arbitrary example extensions such as `A.cfc`.
- Converts MXUnit's colon-delimited exception annotation to a native dotted annotation and updates its lookup key.
- Normalizes CR-only fixture files before parsing.
- Works around the pretty printer's `not contains` and comment-between-else-and-if output.
- Guards cbMockData 4's HTTP-service detection in CLI runs, where no CGI scope exists.

These are temporary conversion accommodations, not a replacement transpiler. Remove them as the relevant upstream fixes become available. Filename rewriting uses known converted basenames; review ambiguous filenames when adding fixtures. The generated files are syntax-checked, but runtime tests remain essential.

MockBox itself now generates native `.bx` classes and `.bxm` methods on BoxLang, and retains CFML generation on Adobe/Lucee. `NativeMockGenerationSpec.bx` checks both execution and generated source types in an isolated directory.

## Validation checkpoints

### BoxLang 1.19.0+67

Rebuilding the native distribution succeeds. All 48 bundles excluding HTMLReportersTest pass: **894 passed, 0 failed, 0 errors, 34 skipped** (928 specs).

The file-read defect is fixed: terminal LF/CRLF, embedded CRLF, and spaces are preserved, and `fileReadBinary()` returns binary data. The standalone file-read check and the previously failing asset checksum spec both pass.

The 82-spec HTML reporter bundle was not rerun on 1.19.0. It passed on 1.16.0 as recorded below.

For this check, 1.19.0 was invoked explicitly from BVM's version directory; the session's default `boxlang` command still resolved to 1.16.0. No BVM defaults were changed.

### BoxLang 1.16.0+57 (earlier checkpoint)

The converted suite was run in two groups:

| Group | Passed | Failed | Errors | Skipped |
| --- | ---: | ---: | ---: | ---: |
| 48 bundles excluding HTMLReportersTest | 893 | 1 | 0 | 34 |
| HTMLReportersTest | 82 | 0 | 0 | 0 |

The HTML bundle took approximately 14 minutes. Shorter time limits do not establish that it hangs.

There are no `.cfc`, `.cfm`, or `.cfs` files left in the generated tree. Combined across the two runs: 975 passed, 1 failed, 0 errors, 34 skipped.

The remaining failure is the vendored-asset checksum check. On this runtime, both `fileRead()` and `fileReadBinary()` read text files line-by-line: terminal newlines are lost and CRLF is normalized to LF. Trailing spaces are preserved. `fileReadBinary()` unexpectedly returns a string for text files. Bootstrap's 232,066-byte CSS file becomes 232,065 UTF-8 bytes after losing its final LF. The earlier 232,062 figure counted Unicode characters, not bytes. Reading the bytes with Java NIO preserves the file and matches the manifest. The checksum assertion remains enabled; do not change the manifest to match the truncated read. Reproduce independently with `boxlang build/check-native-file-read.bxs` from the repository root.

A focused regression run of the original CFML sources on BoxLang also passed: 103 passed, 3 skipped, no failures/errors (MockBox, native mock generation, Set/Range mocks, BDD lifecycle/inheritance, Agent reporter, and streaming).

Adobe ColdFusion, Lucee, Windows, browser interaction, and real HTTP streaming were not validated. No upstream BoxLang or dependency repositories were modified.
