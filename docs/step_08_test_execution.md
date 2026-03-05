# Step 08: Test Execution — Functional Specification

**Step identifier:** `step_08`
**Step name:** Test Execution
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 08 executes the project's test suite, collects coverage metrics, identifies per-module
coverage gaps, and invites the AI to analyse results and recommend targeted improvements.

The step is language-aware: it detects the primary language, selects the appropriate test
command, runs tests with non-interactive output forced, parses the output for test counts,
reads coverage data from standard report files, and then invokes two AI personas — one
focused on test quality and failure analysis, one focused on end-to-end test engineering.

Step 08 always attempts to run tests when a command can be determined. It does not gate on
the change scope established by Step 00.

---

## 2. Step Kind Conformance

Step 08 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **No** — Step 08 does not return `{ skipped: true }` |
| AI-integrated | **Yes** — two AI calls (`test_engineer` persona, two different prompt templates) |

**Error-handling deviation.** Step 08 re-throws unhandled exceptions from `execute` rather
than returning `{ success: false, error }`. This deviates from step contract rule 4
(§5.4 of `step_contract.md`). Callers must be prepared to catch exceptions propagated
from this step.

**No-test-command early return.** When no test command can be determined, Step 08 returns
`{ success: false, language, testResults: {}, message: 'No test command configured' }`.
This is a graceful failure, not a standard skip (`skipped: true`). The step does not
throw in this case; the `success: false` result is a valid, non-exceptional outcome.

**AI Prompt Contract conformance.** Step 08 conforms to
[`ai_prompt_contract.md`](./ai_prompt_contract.md) for prompt construction. Unlike steps
that inject file content, Step 08 injects structured test execution data: counts, durations,
coverage metrics, and truncated output snippets. The model is not asked to retrieve or
infer file contents; all context is explicitly provided as prompt variables.

---

## 3. Domain Concepts

### 3.1 Language Detection

The primary language is detected using the tech-stack detector, which inspects the
project's file patterns and manifests. When detection fails or yields no result, `javascript`
is assumed as the default.

### 3.2 Test Command Resolution

The test command is resolved using a three-tier priority:

1. **Package manifest script (Node.js / TypeScript only).** When the project has a
   `package.json` with a `test` script defined, the command is always `npm test`. The raw
   value of the script field is not used directly, because it would fail when `node_modules/.bin`
   is not in the shell's `PATH`.

2. **Language default.** A built-in mapping provides default commands for:
   JavaScript → `npm test`, TypeScript → `npm test`, Python → `pytest`,
   Go → `go test ./...`, Java → `mvn test`, Ruby → `rspec`, Rust → `cargo test`.

3. **Workflow configuration.** Read `tech_stack.test_command` from `.workflow-config.yaml`.

When all three tiers yield no command, the step returns a graceful failure (§5.2, Phase 1).

### 3.3 Test Execution Environment

Tests are run with `CI=true` and `FORCE_COLOR=0` injected into the process environment.
This forces non-interactive, pipe-friendly output from Jest and similar runners, ensuring
reliable output capture regardless of whether a TTY is attached to the parent process.
Without these flags, some runners emit ANSI cursor-control sequences or interactive
progress displays that can result in zero bytes being captured when the process exits before
flushing those streams.

The default test run timeout is 300,000 milliseconds (5 minutes). This may be overridden
via `options.timeout`.

### 3.4 Runner Crash Detection

When the test runner exits with a non-zero code and produces zero bytes of combined output:

1. The step retries once. For commands that include `jest`, `--ci` is appended to the
   command before retrying. For all other runners, the original command is reused.
2. If the retry also produces zero output, a `runnerCrashed` sentinel flag is set on the
   internal result.

A runner crash is treated the same as "no tests found" (§3.6): a warning condition, not a
failure. The workflow continues and a note is written to the backlog recommending manual
investigation.

### 3.5 Test Output Parsing

Language-specific parsers extract test counts from the runner's combined stdout and stderr.
ANSI colour escape sequences are stripped before parsing.

| Language | Parser behaviour |
|---|---|
| JavaScript, TypeScript | Parses the `Tests:` summary line for passed, failed, skipped, and total counts. Parses the `Test Suites:` line for `suitesFailed` and `suitesTotal`. |
| Python | Parses the final pytest summary line for passed, failed, and skipped counts. Derives total as the sum of the three. |
| All others | Returns all counts as zero (no structured parser). |

### 3.6 Test Status Determination

The step outcome is `success: false` when any of the following holds:

- One or more test cases failed (`failed > 0`)
- One or more test suites failed to run (`suitesFailed > 0`)
- The runner exited with a non-zero code and the run does not qualify as "no tests found"

**"No tests found" condition.** A run is classified as "no tests found" — and treated as
`success: true` with a warning — when all of the following hold:
- `total === 0` and `suitesFailed === 0`
- At least one of: exit code is 0; the output contains a recognised "no tests" phrase
  (such as `"No tests found"`, `"No test files found"`, or `"Your test suite must contain
  at least one test"`); or `runnerCrashed` is set.

### 3.7 Coverage Collection

After the test run, the step searches for coverage report files at language-specific paths.
For Node.js and TypeScript projects the preferred file is `coverage/coverage-summary.json`
(Jest format). When a readable JSON coverage file is found:

- **Aggregate metrics** are extracted from the `total` key: statements, branches, functions,
  and lines, each as a percentage.
- **The raw JSON** is retained for per-file gap analysis.

When no coverage file is readable, the coverage object is empty and gap analysis is skipped.

### 3.8 Coverage Gap Detection

The per-file entries of the coverage JSON are inspected. For each file where any of the
four metrics (statements, branches, functions, lines) falls below the configured threshold,
a gap record is produced. Gap records are sorted by worst minimum metric ascending, so the
most urgently under-covered modules appear first.

The coverage threshold is read from `.workflow-config.yaml` at the path
`validation.testing.min_coverage`. The threshold must be a number between 1 and 100
inclusive. It defaults to `80` when the configuration file is absent, unreadable, or does
not specify the field.

---

## 4. Constructor Dependencies

Step 08 receives all external collaborators at construction time via a single options map.
Each dependency falls back to a freshly instantiated default when absent.

| Dependency key | Role | Behaviour when absent |
|---|---|---|
| `executor` | Runs shell commands (the test command) | Falls back to the executor module |
| `fileOps` | Reads `package.json`, coverage JSON files, and workflow configuration | Falls back to a new file-operations instance |
| `backlog` | Writes step reports to the workflow artifact store | Falls back to a new backlog instance |
| `techStack` | Detects the project's primary language | Falls back to a new tech-stack detector instance |
| `aiHelper` | Executes AI API requests | Falls back to a new AI helper instance |
| `aiCache` | Caches and retrieves AI responses by key | Falls back to a new AI cache instance |
| `configManager` | Provides project configuration, used to read the test framework name | Optional; `null` accepted; test framework defaults to the detected language name |
| `promptsDir` | Directory path for AI prompt templates | Passed to the AI helper; `null` when absent |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the root of the project under test |
| `options.timeout` | Override the default test run timeout in milliseconds. Default: 300,000 (5 minutes). |
| `options.projectDescription` | Human-readable description of the project, injected into the primary AI prompt. |
| `options.projectKind` | Project kind identifier, injected into the e2e AI prompt. |
| `options.modifiedFiles` | List of modified file paths; the count is used in the e2e AI prompt. |

### 5.2 Execution Phases

Step 08 executes the following phases in order:

#### Phase 1 — Language and Command Detection

Detect the primary language (§3.1). Resolve the test command using the three-tier lookup
(§3.2). When no command is found:

1. Log a warning.
2. Generate a minimal test report indicating no test command was configured.
3. Write the report to the backlog under step `8`, section `"Test Execution"`.
4. Return `{ success: false, language, testResults: {}, message: 'No test command configured' }`.

Remaining phases are skipped.

#### Phase 2 — Test Execution

Run the resolved test command in `projectRoot` with the non-interactive environment (§3.3)
and the configured timeout. Apply crash detection and the `--ci` retry logic (§3.4). The
result carries the exit code, combined output, and the optional `runnerCrashed` sentinel.

#### Phase 3 — Output Parsing

Strip ANSI escape codes from the combined output. Apply the language-appropriate parser
(§3.5) to extract test counts. Log the parsed counts at debug level.

#### Phase 4 — Coverage Collection

Search for coverage report files at the language-specific paths (§3.7). Parse the first
readable JSON file to extract aggregate coverage metrics and retain the raw JSON for gap
analysis.

#### Phase 4b — Coverage Gap Analysis

Read the coverage threshold from workflow configuration (§3.8). Iterate the per-file
entries of the coverage JSON and collect gap records for files below the threshold. Sort
by worst minimum metric. Log the count of modules below threshold.

#### Phase 5 — Report Generation and Backlog Write

Compute the step duration (wall-clock time from Phase 1 start to here). Apply the success
logic (§3.6) to determine `success` and `noTestsFound` flags. Generate a structured
Markdown report covering: summary, test counts, coverage aggregate metrics, per-file
coverage gaps, and recommendations when failures are present.

Write the report to the backlog under step `8`, section `"Test Execution"`.

Handle special backlog entries for warning states:
- **Runner crashed**: append a crash-detection note explaining the likely cause (OOM kill
  or process crash before first write) and recommend running the test command manually.
- **No tests found (non-crash)**: append a recommendation to add a test suite, with
  language-appropriate setup instructions.

#### Phase AI-1 — Test Quality Analysis (Conditional)

Runs when the AI helper initialises successfully.

1. **Initialise the response cache.** Call `aiCache.init()` before any `withCache` call.

2. **Build coverage gaps text.** Produce a text block listing each module below the
   threshold with all four coverage percentages. When no gaps exist, use the text
   `'none — all modules meet the threshold'`.

3. **Build the primary prompt.** Load the `step7_test_exec_prompt` template from the AI
   helpers YAML configuration and populate:
   `project_name`, `project_description`, `primary_language`, `test_framework`,
   `test_command`, `test_exit_code`, `tests_total`, `tests_passed`, `tests_failed`,
   `execution_summary` (a single-line summary of counts and duration, including suite
   failures if any), `test_output` (first 2,000 characters of combined runner output, or
   `'none'` when empty), `failed_test_list` (first 1,000 characters of output when
   failures occurred, otherwise `'none'`), `coverage_threshold`, `coverage_gaps`.

   When the template is unavailable, fall back to a built-in structured prompt that embeds
   language, test counts, aggregate coverage, coverage threshold, coverage gaps, duration,
   and exit code.

4. **Execute via cache.** Cache key: `step_08|{language}|{passed}|{failed}|{coverageGaps.length}`.
   Persona: `test_engineer`.

5. **Append to backlog.** If a response was produced, append it to the backlog report
   under a `"AI Recommendations"` heading.

#### Phase AI-2 — E2E Analysis (Conditional, Supplementary)

Runs when the AI helper initialised successfully and the `e2e_test_engineer_prompt` YAML
template is available.

1. **Build the e2e prompt.** Populate the template with: `project_name`,
   `project_description`, `project_type`, `e2e_framework`, `test_command`,
   `browser_targets` (empty string unless provided), `modified_count` (count of files in
   `options.modifiedFiles`).

2. **Execute via cache.** Cache key: `step_08_e2e|{language}|{passed}`.
   Persona: `test_engineer`.

3. **Append to backlog.** If a response was produced, append it to the backlog report
   under an `"E2E Test Engineering Analysis"` heading.

When the AI helper is unavailable, both AI phases are skipped with a warning logged.

### 5.3 Skip Conditions

Step 08 has no conventional skip conditions that return `{ skipped: true }`. When no test
command is found, the step returns a graceful failure (see §5.2, Phase 1 and §2).

---

## 6. Data Shapes

### 6.1 StepResult (normal execution)

| Field | Type | Description |
|---|---|---|
| `success` | Boolean | `true` when no test failures occurred and the runner did not exit abnormally (excluding "no tests found") |
| `noTestsFound` | Boolean | `true` when the runner found no test files (warning state; `success` is `true` in this case) |
| `language` | String | Detected primary language |
| `testResults` | TestCounts | Parsed test counts (§6.2) |
| `coverage` | CoverageMetrics | Aggregate coverage percentages (§6.3); empty object when no coverage data found |
| `duration` | Integer | Step execution duration in milliseconds |
| `exitCode` | Integer | Exit code returned by the test runner process |

### 6.2 TestCounts

| Field | Type | Description |
|---|---|---|
| `total` | Integer | Total number of test cases |
| `passed` | Integer | Number of passing test cases |
| `failed` | Integer | Number of failing test cases |
| `skipped` | Integer | Number of skipped test cases |
| `suitesFailed` | Integer | Number of test suites that failed to load or run (Jest-specific) |
| `suitesTotal` | Integer | Total number of test suites (Jest-specific) |

### 6.3 CoverageMetrics

| Field | Type | Description |
|---|---|---|
| `statements` | Number | Statement coverage percentage |
| `branches` | Number | Branch coverage percentage |
| `functions` | Number | Function coverage percentage |
| `lines` | Number | Line coverage percentage |

All fields are absent (the object is empty) when no readable coverage file is found.

### 6.4 CoverageGap

Produced by the coverage gap analysis (§3.8); not included directly in the `StepResult`
but injected into AI prompt variables and included in the backlog report.

| Field | Type | Description |
|---|---|---|
| `file` | String | File path as keyed in the coverage JSON |
| `statements` | Number | Statement coverage percentage |
| `branches` | Number | Branch coverage percentage |
| `functions` | Number | Function coverage percentage |
| `lines` | Number | Line coverage percentage |
| `min` | Number | Lowest of the four metrics |

### 6.5 StepResult (no test command)

| Field | Type | Description |
|---|---|---|
| `success` | `false` | — |
| `language` | String | Detected primary language |
| `testResults` | `{}` | Empty map |
| `message` | `'No test command configured'` | Human-readable explanation |

---

## 7. Constraints and Rules

1. When `package.json` defines a `test` script, the test command **must** be `npm test`,
   not the raw script value. Using the raw script value causes `command not found` errors
   because `node_modules/.bin` is not in the shell's `PATH`.

2. Tests **must** be executed with `CI=true` and `FORCE_COLOR=0` in the environment.
   Omitting these flags risks capturing zero bytes of output from interactive runners,
   making the results unparseable.

3. The crash-detection retry **must** only append `--ci` to Jest-family commands (those
   whose command string includes `jest`). Adding `--ci` to other runners (pytest, `go
   test`, etc.) may cause the command to fail with an unrecognised flag.

4. The coverage threshold **must** be validated as a number in the range 1–100 before use.
   Values outside this range are ignored and the default of 80 is used.

5. The "no tests found" condition **must** be handled as `success: true` with a warning
   backlog entry. It **must not** cause the step to return `success: false`. The workflow
   should continue when a project simply has no test suite yet.

6. The runner crash condition **must** also be handled as `success: true` with a dedicated
   backlog note. A crashed runner indicates an infrastructure problem that requires manual
   investigation, not a code quality failure that should halt the workflow.

7. Each AI prompt variable that contains test runner output (`test_output`,
   `failed_test_list`) **must** be character-limited before injection. Injecting unbounded
   output risks exceeding context-window limits. (Ref: AI Prompt Contract §3.5.)

8. The AI response cache **must** be initialised (via `aiCache.init()`) before the first
   `withCache` call. Failure to do so causes a runtime error.

9. **Exception propagation deviation.** Callers of `execute` **must** wrap the call in
   error handling. Step 08 does not guarantee that exceptions are converted to
   `{ success: false }` results; unhandled exceptions are re-thrown.
