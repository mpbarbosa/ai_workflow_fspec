# Step 06: Test Review — Functional Specification

**Step identifier:** `step_06`
**Step name:** Test Review
**Step kind:** `CONTEXT` (see [Step Contract §2.2](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 06 performs a comprehensive review of the project's test suite. It discovers test
files, categorises them, checks for coverage reports, identifies structural issues, and
submits actual test file contents to the `test_engineer` AI persona for a quality review.

Step 06 answers three questions:

1. **What tests exist?** — by discovering test files matching language-specific glob
   patterns and categorising them by type (unit, integration, end-to-end, other).
2. **Is coverage adequate?** — by locating language-specific coverage reports and
   extracting the coverage percentage.
3. **Are the tests well-written?** — by submitting a rotating partition of test files,
   with their actual contents injected, to the AI for quality review.

The step uses a **partition-rotate strategy** for AI analysis: each run reviews one
subset (partition) of the total test file population, then advances to the next partition
on success. This keeps individual AI prompts within model limits while ensuring all test
files are eventually reviewed across multiple workflow runs.

---

## 2. Step Kind Conformance

Step 06 conforms to the **CONTEXT** step kind as defined in `step_contract.md §2.2`.

| Property | Value |
|---|---|
| Kind identifier | `"context"` |
| Execute signature | `execute(context) → Promise<StepResult>` |
| Can be skipped | **Yes** — when no test files are found (returns success with issue record) |
| AI-integrated | **Yes** — calls the AI API via `aiHelper` and `aiCache` |
| Conforms to AI Prompt Contract | **Yes** — actual test file contents are injected per partition slice (§5 Phase 7) |

**Dual calling convention.** Step 06 accepts two execute signatures for compatibility:

- **Context-style** (orchestrator): `execute({ projectRoot, categorizedFiles, … })` —
  the first argument is the workflow context object.
- **Legacy / test style**: `execute('/path', options)` — the first argument is a
  project root path string.

Both conventions produce identical behaviour. The step detects which style is in use
at runtime by checking whether the first argument is a string.

**Logging convention deviation.** Step 06 is declared as a CONTEXT step but uses the
**global logger singleton** rather than `this.logger` injected at construction time.
This deviates from the CONTEXT step logging contract in `step_contract.md §2.2`, which
requires CONTEXT steps to use `this.logger`. The global singleton is used throughout
the execute body. There is no `logger` constructor option.

**Error handling deviation.** Step 06 re-throws unhandled exceptions from `execute`
rather than returning `{ success: false, error }`. This deviates from the step contract
rule (§5 rule 4) that forbids propagating exceptions. Callers must be prepared to catch.

---

## 3. Domain Concepts

### 3.1 Test File Patterns

Test files are discovered by language-specific glob patterns. The language is detected
at execution time from the project's technology stack.

| Language | File patterns |
|---|---|
| JavaScript | `**/*.test.js`, `**/*.spec.js`, `**/*.test.mjs`, `**/*.spec.mjs` |
| TypeScript | `**/*.test.ts`, `**/*.spec.ts`, `**/*.test.tsx`, `**/*.spec.tsx` |
| Python | `**/test_*.py`, `**/*_test.py` |
| Go | `**/*_test.go` |
| Java | `**/*Test.java`, `**/*Tests.java` |
| Ruby | `**/*_spec.rb`, `**/*_test.rb` |
| Rust | `tests/**/*.rs` |
| C++ | `**/*_test.cpp`, `**/*_test.cc` |
| Bash | `**/*.bats`, `**/test_*.sh` |

When the detected language has no matching pattern, JavaScript patterns are used as a
default.

**Bash fallback.** When the primary language patterns yield no results, the step
automatically retries with Bash patterns. If Bash patterns find files, those are used
instead and the substitution is logged.

### 3.2 Test File Categories

Each discovered test file is classified into exactly one category based on its path:

| Category | Matching condition |
|---|---|
| `unit` | Path contains `/unit/` or the filename contains `unit.test` |
| `integration` | Path contains `/integration/` or the filename contains `integration.test` |
| `e2e` | Path contains `/e2e/` or the filename contains `e2e.test` |
| `other` | No category-specific pattern matched |

Classification applies to the lower-cased path.

### 3.3 Coverage Report Paths

Coverage reports are sought at language-specific paths relative to the project root:

| Language | Candidate paths |
|---|---|
| JavaScript | `coverage/lcov-report/index.html`, `coverage/index.html` |
| Python | `htmlcov/index.html`, `coverage/index.html` |
| Java | `target/site/jacoco/index.html` |
| Go | `coverage.html`, `coverage.out` |
| Ruby | `coverage/index.html` |
| Rust | `target/debug/coverage/index.html` |

The first existing path wins. If no path exists, coverage is reported as not found.

### 3.4 Coverage Percentage Extraction

When a coverage report is found, the step attempts to extract a numeric percentage from
the report content by matching patterns such as `85.5%`, `Coverage: 85.5%`, or
`Statements: 85.5%`. The first match in the range 0–100 is used. If no match is found,
the percentage is reported as `null`.

### 3.5 Issue Types

| Identifier | Condition |
|---|---|
| `no_tests` | No test files found anywhere in the project |
| `no_coverage_report` | Test files exist but no coverage report is found |
| `low_coverage` | Coverage percentage is below the threshold (default 80%) |

### 3.6 Partition-Rotate Strategy

AI analysis operates on one partition of the total test file population per run. The
partition state is persisted to a cache file
(`.ai_workflow/.step_cache/step_06_partition.json`) between runs.

**Active candidates** are the subset of unique relative test file paths that are either
present in the current run's modified file list or in the full test file list. Modified
files are always prioritised.

The current partition is a contiguous slice of active candidates, determined by the
persisted index. When the step completes its AI analysis successfully, the index
advances to the next partition. On the next run, the following partition is reviewed.
After the last partition is processed the index wraps to the beginning.

Each partition is further subdivided into **slices** of up to 5 files each. All slices
within a partition are dispatched to the AI in parallel. Each slice carries its own
cache key and produces its own AI response section.

---

## 4. Constructor Dependencies

Step 06 receives all collaborators at construction time via a single options map.
All dependencies have fallback defaults.

| Dependency key | Role | When absent |
|---|---|---|
| `fileOps` | File system operations: read files, glob patterns, check file existence | Falls back to a new `FileOperations` instance |
| `backlog` | Writes step summary to the workflow artifact store | Falls back to a new `Backlog` instance |
| `techStack` | Detects the project's primary programming language | Falls back to a new `TechStackDetector` instance |
| `aiHelper` | Executes AI API requests (see AI Prompt Contract §6.1) | Falls back to a new `AiHelper` instance |
| `aiCache` | Caches AI responses by key (see AI Prompt Contract §6.2) | Falls back to a new `AiCache` instance |

**Note:** There is no `logger` constructor option. The step uses the global logger
singleton (see §2 logging deviation).

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Source | Description |
|---|---|---|
| `context` | Orchestrator | Full workflow context object (or legacy: project root path string) |
| `context.projectRoot` | Caller | Absolute path to the project root; defaults to process working directory |
| `context.categorizedFiles.test` | Preferably Step 00 `contextUpdate` | Changed test file paths from Step 00's classification; merged with the filesystem scan |
| `context.config.tech_stack.test_runner` / `context.techStack.testRunner` | Optional | Test framework name for prompt context |
| `options.modifiedFiles` | Optional (legacy convention) | File paths to prioritise in partition selection |
| `options.projectKind` | Optional | Project kind for project-kind role overlay in AI prompts |

### 5.2 Execution Phases

Step 06 runs the following phases in order:

#### Phase 1 — Detect Primary Language

Query the technology stack detector for the project's primary programming language.
If detection fails, default to `'javascript'`.

#### Phase 2 — Discover Test Files

Run the test file glob patterns for the detected language against the project root,
excluding `node_modules`, `.git`, `coverage`, `dist`, `build`, `__pycache__`, and
`target`. Collect all matching file paths and remove duplicates. Apply the `isTestFile`
filter as a secondary safeguard.

**Bash fallback.** If the result set is empty and the detected language is not `'bash'`,
repeat the discovery with Bash patterns. If Bash patterns find files, use those and log
the substitution.

**Step 00 merge.** After the filesystem scan, merge in the test file paths from
`context.categorizedFiles.test` (when present and non-empty). The merged set is
de-duplicated. This ensures files modified in the current run but not matched by the
filesystem glob (e.g. due to naming conventions) are still included.

#### Phase 3 — Early Return When No Tests Found

When the resolved test file list is empty:

1. Log a warning.
2. Generate a minimal report indicating no test files were found.
3. Write the report to the backlog.
4. Return `{ success: true, testFiles: [], issues: [{ type: 'no_tests', … }] }`.

No further phases execute.

#### Phase 4 — Categorise Test Files

Classify each discovered test file into one of the four categories (§3.2). Count files
per category.

#### Phase 5 — Count Test Lines

For each test file, read its content and count the lines. Sum across all files to
produce a total line count. Unreadable files are silently skipped.

#### Phase 6 — Analyse Coverage

Search the language-specific coverage report paths (§3.3) in order. For the first
existing report file found, attempt to extract a coverage percentage (§3.4). Return the
path, percentage (or `null`), and a `found` boolean.

#### Phase 7 — Identify Issues

Produce an issue list based on the collected data (§3.5):
- `no_tests` — if the test file list is empty (not reached in the normal path, handled
  by Phase 3).
- `no_coverage_report` — if test files exist but no coverage report was found.
- `low_coverage` — if a coverage percentage was found and it is below 80%.

#### Phase 8 — Generate Initial Report and Save to Backlog

Produce a structured Markdown report containing:
- Summary: total test files, total lines, coverage report status, issue count.
- Test distribution: counts per category (unit, integration, e2e, other).
- Coverage analysis: percentage and status message when a report is found; a warning
  when test files exist but no report was found.
- Issues section: grouped by type, capped at 10 entries per type.
- Recommendations: critical note when no tests exist; recommendations when coverage is
  missing.

Write this initial report to the backlog summary entry for step `6`, section
`'Test Review'`.

#### Phase 9 — AI Test Quality Review (Partition-Rotate)

This phase is wrapped in its own error handler. Any exception is caught, logged as a
warning, and the phase is silently skipped without affecting the overall result.

When AI is available:

1. Initialise `aiHelper`. If unavailable, log a warning and skip the rest of this phase.
2. Initialise `aiCache`.
3. Resolve unique relative test file paths from the discovered list.
4. Load the partition cache from `.ai_workflow/.step_cache/step_06_partition.json`.
5. Determine active candidates (§3.6) using modified files from options.
6. Get the current partition from the cache.
7. Log the partition index, total partitions, file count, and partition label.
8. Read file contents for the current partition's files only. Each file is read
   independently; unreadable files are silently skipped. Contents are truncated to
   5 000 characters per file.
9. Load the YAML AI helpers configuration and build the `test_engineer` persona role
   overlay from the project kinds configuration (optional; silently skipped on error).
10. Divide the partition into slices of up to 5 files each.
11. Dispatch all slices **in parallel**. For each slice:
    a. Assemble a file contents section: each file formatted as a fenced code block
       labelled with the relative path and the detected language.
    b. Attempt to build the prompt from the `step5_test_review_prompt` YAML template,
       populating: project name, project description, primary language, test framework,
       test and coverage commands, file counts, and file contents.
    c. If the YAML template is unavailable, fall back to the built-in test review prompt
       builder using the slice's file list and language.
    d. Prepend the project-kind role override when one was resolved (optional).
    e. Execute the AI request with cache key
       `step_06|v2|p{partitionIndex}|s{sliceIndex}|{language}|{file1},{file2},…`
       and persona `test_engineer`.
12. Collect non-empty AI response sections.
13. Concatenate them with horizontal rule separators, prefixed by a partition header
    line identifying the partition index and label.
14. Overwrite the backlog summary entry with the initial report extended by the AI
    review section.
15. Advance the partition cache index to the next partition.

### 5.3 Skip Conditions

| Condition | Outcome |
|---|---|
| No test files found | Returns `{ success: true, testFiles: [], issues: [{type:'no_tests',…}] }` |

This is not a standard `skipped: true` result. It is a success result with an issue
record. The step does not have formal skip conditions in the sense of §4.1 of the step
contract.

### 5.4 Error Handling

The outer execute body is wrapped in a try/catch. If an unhandled exception occurs in
Phases 1–8, the error is logged and **re-thrown**. This deviates from the step
contract. Callers must catch.

Phase 9 (AI review) is wrapped in its own separate try/catch. Any exception in Phase 9
is caught, logged as a warning, and silently discarded. Phase 9 failure never affects
the outer result.

---

## 6. Data Shapes

### 6.1 StepResult (success, tests found)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `testFiles` | String[] | All discovered test file paths |
| `categories` | Object | Test files grouped by category (§6.2) |
| `statistics` | Object | Test statistics (§6.3) |
| `coverage` | Object | Coverage analysis result (§6.4) |
| `issues` | Object[] | Issue records (§6.5) |

### 6.2 Test Categories

| Field | Type | Description |
|---|---|---|
| `unit` | String[] | Paths of unit test files |
| `integration` | String[] | Paths of integration test files |
| `e2e` | String[] | Paths of end-to-end test files |
| `other` | String[] | Paths of test files in no specific category |

### 6.3 Test Statistics

| Field | Type | Description |
|---|---|---|
| `totalTests` | Integer | Total test file count |
| `unitTests` | Integer | Unit test file count |
| `integrationTests` | Integer | Integration test file count |
| `e2eTests` | Integer | End-to-end test file count |
| `otherTests` | Integer | Other test file count |
| `totalLines` | Integer | Total lines across all test files |
| `averageLinesPerTest` | Integer | Average lines per test file (0 when no files) |

### 6.4 Coverage Result

| Field | Type | Description |
|---|---|---|
| `found` | Boolean | Whether a coverage report file was located |
| `path` | String or null | Relative path to the coverage report; null when not found |
| `percentage` | Number or null | Extracted coverage percentage; null when unextractable or not found |

### 6.5 Issue Record

| Field | Type | Description |
|---|---|---|
| `type` | String | Issue type identifier (§3.5) |
| `message` | String | Human-readable issue description |

### 6.6 StepResult (no tests found)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `testFiles` | `[]` | Empty array |
| `issues` | Object[] | Single issue record with type `no_tests` |

---

## 7. Constraints and Rules

1. **Logging deviation.** Step 06 is declared as a CONTEXT step but uses the global
   logger singleton, not `this.logger`. This violates the CONTEXT step logging contract
   in `step_contract.md §2.2`. Implementations must not add `this.logger` to fix this
   without also updating all logger calls in the execute body.

2. **Error handling deviation.** Step 06 re-throws unhandled exceptions (Phases 1–8)
   rather than returning `{ success: false, error }`. This violates step contract §5
   rule 4. Callers must wrap `execute` in their own error handler.

3. The step **must** perform a full filesystem scan on every run. The filesystem scan
   is the primary source of test files. The Step 00 categorised list is merged in as a
   supplement, not a replacement. Using only the Step 00 list would miss unmodified
   test files that are still relevant to coverage and quality review.

4. The bash fallback **must** be logged when activated. Silent substitution of the
   language is not permitted.

5. `aiCache.init()` **must** be called before any `withCache` call, as required by the
   AI Prompt Contract §6.2.

6. File content injection **must** comply with the AI Prompt Contract §3: each file
   read is wrapped in its own error handler, unreadable files are silently skipped, and
   content is truncated to 5 000 characters per file. Each slice must format its file
   contents as fenced code blocks labelled with the relative file path.

7. The partition index **must** be advanced after a successful AI review. If the AI
   review is skipped (AI unavailable) or fails (exception in Phase 9), the partition
   index must not advance. The same partition will be retried on the next run.

8. All slices within a partition **must** be dispatched in parallel. Serial slice
   execution is not permitted; it negates the performance benefit of the
   partition-rotate strategy.

9. The AI cache key **must** encode the partition index, slice index, language, and
   exact file list. A key that does not encode all of these may serve a cached response
   built for a different set of files, producing incorrect review output.

10. Phase 9 failure **must never** cause the step to return `success: false` or to
    re-throw. The initial report (Phase 8) represents the step's core deliverable.
    AI review is an enhancement; its absence must not degrade the step result.
