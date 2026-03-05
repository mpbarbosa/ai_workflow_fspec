# Step 07: Test Generation — Functional Specification

**Step identifier:** `step_07`  
**Step name:** Test Generation  
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))  
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))  
**Version:** 2.0.0  
**Status:** Draft  
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 07 identifies source files that lack corresponding test files and generates those
tests using an AI `test_engineer` persona. It produces a coverage gap report, a
AI-generated test strategy document, and — for a bounded number of untested files — actual
test file content ready to be committed.

Step 07 answers two questions:

1. **Where are the test coverage gaps?** — which source files have no matching test file,
   how are they distributed across directories, and what is the overall coverage
   percentage?
2. **How should the gaps be closed?** — a test strategy recommendation from the AI, plus
   generated test files for up to five of the highest-priority untested files.

---

## 2. Step Kind Conformance

Step 07 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **No** — returns a success result (not a skip) when no source files are found |
| AI-integrated | **Yes** — calls the AI API via `aiHelper` and `aiCache` |

**Logging convention.** Step 07 uses the **global logger singleton** directly, which is
consistent with the PROJECT step convention defined in `step_contract.md §2.1`.

**Error handling deviation.** Step 07's top-level catch block logs the error and then
**re-throws** it rather than returning `{ success: false, error: … }`. This violates
`step_contract.md §5` rule 4, which requires that `execute` never propagate an unhandled
exception. Callers must be prepared to catch exceptions from this step.

The AI generation sub-phase (Phase 7) is wrapped in its own independent try/catch. AI
errors are logged as warnings and do not propagate; the step returns a success result even
when AI generation fails entirely.

**No canonical skip result.** When no source files are found, Step 07 does **not** return
`{ success: true, skipped: true, … }`. Instead it returns a full success result with zero
counts and an empty `generatedFiles` list. This deviates from the pattern used by other
steps that have a natural "nothing to do" condition.

---

## 3. Domain Concepts

### 3.1 Language Detection

Step 07 determines the project's primary programming language by querying the tech-stack
detection service. When detection fails or yields no result, `"javascript"` is used as the
default.

A **bash fallback** applies when the detected language produces no source files: Step 07
repeats the source-file discovery using `bash` patterns. If that also produces no files,
execution continues with an empty source list.

### 3.2 Source File Patterns

Source files are discovered by glob pattern. The patterns used depend on the detected
language:

| Language | Glob patterns |
|---|---|
| JavaScript | `src/**/*.js`, `src/**/*.ts`, `src/**/*.jsx`, `src/**/*.tsx`, `src/**/*.vue`, `lib/**/*.js`, `lib/**/*.ts`, `scripts/**/*.js` |
| TypeScript | `src/**/*.ts`, `src/**/*.tsx`, `src/**/*.vue`, `lib/**/*.ts` |
| Python | `src/**/*.py`, `lib/**/*.py`, `**/*.py` |
| Go | `**/*.go` |
| Java | `src/**/*.java` |
| Ruby | `lib/**/*.rb`, `app/**/*.rb` |
| Rust | `src/**/*.rs` |
| Bash / Shell | `**/*.sh` |

When the detected language does not match any entry above, the JavaScript patterns are
used.

### 3.3 Excluded Directories

The following directories are excluded from both source and test file discovery:
`node_modules`, `.git`, `coverage`, `dist`, `build`, `__pycache__`, `target`, `vendor`,
`.ai_workflow`.

### 3.4 Excluded Source Files

The following individual filenames are excluded from gap analysis (they are entry points
or boilerplate that do not warrant dedicated tests):
`__init__.py`, `index.js`, `main.js`, `config.js`, `constants.js`.

### 3.5 Test File Matching

A source file is considered **tested** when at least one test file in the test file list
corresponds to it by name. Test file correspondence is detected by naming convention,
which varies by language:

| Language | Naming convention |
|---|---|
| Python | `test_<module>.py` |
| Python (alt) | `<module>_test.py` |
| Go | `<module>_test.go` |
| Java | `<module>Test.java` or `<module>Tests.java` |
| Ruby | `<module>_spec.rb` |
| JavaScript / TypeScript | `<module>.test.<ext>` or `<module>.spec.<ext>` |
| JavaScript (Jest) | `__tests__/<module>.*` |

A source file excluded by §3.4, or that is itself a test file, is considered tested and
excluded from the untested list.

### 3.6 Test Output Path Derivation

When the AI generates a test file, the output path is derived from the source file path:

- Leading `src/` in the source path is replaced with `test/`. All other source paths are
  placed under a `test/` prefix.
- The filename is transformed according to language convention:

| Language | Naming convention for generated test |
|---|---|
| Python | `test_<base><ext>` |
| Go | `<base>_test<ext>` |
| Java | `<base>Test<ext>` |
| Ruby | `<base>_spec<ext>` |
| JavaScript / TypeScript (and all others) | `<base>.test<ext>` |

### 3.7 Coverage Percentage

Coverage is the fraction of source files that have a corresponding test file, expressed as
a whole-number percentage:

```
coverage = round((testedCount / totalSourceFiles) × 100)
```

When `totalSourceFiles` is zero, coverage is zero.

### 3.8 AI Test Generation Limits

| Constant | Value | Meaning |
|---|---|---|
| Maximum files to generate | 5 | Only the first 5 untested files (in discovery order) are passed to the AI |
| Maximum source file size | 20,000 characters | Files exceeding this limit are skipped; the AI does not receive their content |

### 3.9 Test Framework Mapping

The AI prompt identifies the test framework by language:

| Language | Framework label in prompt |
|---|---|
| JavaScript | Jest |
| TypeScript | Jest |
| Python | pytest |
| Go | testing (Go stdlib) |
| Java | JUnit 5 |
| Ruby | RSpec |
| Rust | cargo test |

When the language has no entry in this table, the language name itself is used as the
framework label.

---

## 4. Constructor Dependencies

Step 07 receives all external collaborators at construction time via a single options map.
All dependencies fall back to new instances when not provided.

| Dependency key | Role | When absent |
|---|---|---|
| `fileOps` | Reads source file contents, performs glob patterns for file discovery, writes generated test files | Falls back to a new FileOperations instance |
| `backlog` | Writes coverage and generation reports to the workflow artifact store | Falls back to a new Backlog instance |
| `techStack` | Detects the project's primary programming language | Falls back to a new TechStackDetector instance |
| `aiHelper` | Executes AI API requests | Falls back to a new AiHelper instance |
| `aiCache` | Caches and retrieves AI responses by cache key | Falls back to a new AiCache instance |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the root of the project under analysis |
| `options.projectKind` | Project kind identifier used to load the test-engineer role overlay; defaults to `"default"` |
| `options.modifiedFiles` | Array of recently changed file paths; used only in the strategy prompt for context (count only) |

### 5.2 Execution Phases

Step 07 runs the following phases in order:

#### Phase 1 — Language Detection

Query the tech-stack service for the primary language. Log the result. Fall back to
`"javascript"` on detection failure.

#### Phase 2 — Discover Source and Test Files

Use the language-specific glob patterns (§3.2) to discover source files and test files,
applying the excluded directories (§3.3). Deduplicate results.

Apply the **bash fallback** (§3.1): when no source files are found for the primary
language and the primary language is not already `"bash"`, repeat discovery with bash
patterns.

Log the counts of discovered source and test files.

When no source files are found (after the bash fallback): generate a zero-coverage report,
write it to the backlog, and return:

```
{
  success: true,
  totalSourceFiles: 0,
  totalTestFiles: <count>,
  untestedFiles: [],
  coveragePercentage: 0,
  generatedFiles: []
}
```

This is a success result, not a canonical skip (§2, deviation note).

#### Phase 3 — Identify Untested Files

Apply the test-file matching rules (§3.5) to produce the list of source files that have no
corresponding test. Excluded filenames (§3.4) and files that are themselves test files are
always omitted from this list.

Log the count of untested files.

#### Phase 4 — Calculate Coverage

Compute coverage percentage (§3.7). Log the result.

#### Phase 5 — Categorise Untested Files

Group the untested file list by top-level directory. For files at the project root, the
category key is `"root"`.

#### Phase 6 — Generate Coverage Report

Format a Markdown coverage report containing:
- Total source file count
- Total test file count
- Untested file count
- Coverage percentage
- A status label (Excellent / Good / Moderate / Low) based on coverage
- Untested files grouped by directory (up to 10 shown per directory)
- Recommendations when coverage is below 100%

Write the report to the backlog under step `7`, section `"Test Generation"`.

#### Phase 7 — AI Test Generation (Optional, Non-Fatal)

This entire phase is wrapped in a try/catch. Any exception causes a warning log and
returns the step with an empty `generatedFiles` list (the coverage report written in
Phase 6 is preserved).

Initialise the AI helper. When unavailable, log a warning and skip this phase.

When available:

**7a — Initialise cache.** Call `aiCache.init()`.

**7b — Test strategy analysis.** Attempt to load the AI helpers YAML configuration and
the project-kind configuration. Build a `test_strategy_prompt` with:
- `project_name` — derived from the project root directory name
- `coverage_stats` — a formatted string with the coverage percentage and file counts
- `test_files` — a structured summary of all test files grouped by top-level directory
  (file counts and a sample of filenames)
- `modified_count` — number of modified files from `options.modifiedFiles` (zero if
  absent)

Apply the project-kind role overlay for the `test_engineer` role when the configuration
is available.

Submit the strategy prompt via `aiCache.withCache` with the cache key
`step_07_strategy|{projectRoot}|{language}|{untestedCount}` and the `test_engineer`
persona.

When a non-empty strategy response is received, append it to the coverage report and
update the backlog entry.

**7c — Per-file test generation.** Take the first `MAX_FILES_TO_GENERATE` (5) entries
from the untested file list. For each file:

1. Read the source file content. If the file exceeds `MAX_SOURCE_FILE_CHARS` (20,000
   characters), skip it with a warning.
2. Derive the output test file path (§3.6). If a file already exists at that path, skip
   with an informational log.
3. Build a per-file test-generation prompt using the `single_file_test_prompt` YAML
   template when the configuration is available, or the inline fallback template otherwise.
   The prompt includes: source file path, language name, framework label (§3.9), file
   extension, and the full source file content.
4. Submit via `aiCache.withCache` with the cache key `step_07|{sourceFile}|{contentLength}`
   and the `test_engineer` persona.
5. Extract the first fenced code block from the AI response. If no code block is found,
   use the full response text after trimming. If the result is empty, skip with a warning.
6. Write the extracted test code to the derived output path on disk.
7. Add the output path to the `generatedFiles` list.

Log the total count of generated test files after all files are processed.

### 5.3 Skip Conditions

Step 07 has **no canonical skip conditions**. It returns a success result in all cases
where execution completes without an unhandled exception.

When no source files are found, the result shape is identical to a successful run with
zero untested files (see Phase 2 above).

---

## 6. Data Shapes

### 6.1 StepResult (success — source files found)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `totalSourceFiles` | Integer | Count of source files discovered |
| `totalTestFiles` | Integer | Count of test files discovered |
| `untestedFiles` | String[] | Relative paths of source files with no matching test |
| `coveragePercentage` | Integer | Percentage of source files with corresponding tests |
| `categories` | Object | Untested files keyed by top-level directory name |
| `generatedFiles` | String[] | Relative paths of test files written to disk by AI |

### 6.2 StepResult (success — no source files)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `totalSourceFiles` | `0` | — |
| `totalTestFiles` | Integer | Count of test files discovered before the source check |
| `untestedFiles` | `[]` | — |
| `coveragePercentage` | `0` | — |
| `generatedFiles` | `[]` | — |

### 6.3 Test File Summary (used in strategy prompt)

A human-readable string grouping test files by their top-level directory:

```
<total> test files across <N> director[y/ies]:
  <dir1>/ (<count>): <file1>, <file2>, <file3>[, ... (+N more)]
  <dir2>/ (<count>): …
```

When no test files exist, the summary is the string `"none"`.

---

## 7. Constraints and Rules

1. **Error handling deviation.** Step 07 re-throws unhandled exceptions from `execute`
   rather than returning `{ success: false, error: … }`. This violates the step contract.
   Callers **must** wrap invocations of Step 07 in their own try/catch.

2. The AI test-generation phase (Phase 7) **must** be treated as non-fatal. Any AI error,
   missing configuration, or individual file failure must be absorbed with a warning log.
   The coverage report and untested-file data must always be returned regardless of AI
   availability.

3. The AI cache **must** be initialised (`aiCache.init()`) before any `withCache` call.
   Cache keys must encode enough context to avoid returning stale results: the project
   root, source file path, and content length for per-file prompts; the project root,
   language, and untested file count for the strategy prompt.

4. The per-file test generation limit (§3.8) of **5 files** **must** be enforced. The step
   must not submit more than 5 individual file prompts in a single run, regardless of how
   many untested files exist.

5. Files whose content exceeds **20,000 characters** (§3.8) **must** be skipped with a
   warning. The AI must not receive truncated source code as the basis for test generation.
   Note: the inline fallback prompt template does truncate content at this limit, but the
   preferred path is to skip the file entirely before building the prompt.

6. When a test file already exists at the derived output path, that file **must not** be
   overwritten. The step must check for file existence before writing.

7. The test strategy prompt **must** be submitted before any per-file generation prompt. It
   provides a holistic overview of the project's test coverage and informs the subsequent
   per-file prompts indirectly (via the updated backlog entry that downstream reviewers and
   operators read).

8. The AI must be invoked with the **`test_engineer` persona** for all requests in this
   step — both the strategy prompt and the per-file generation prompts.

9. The bash language fallback (§3.1) **must** be applied only when the primary detected
   language yields zero source files. It must not be applied when source files were found
   but all were excluded (by §3.3 or §3.4).

10. Step 07 does not inject existing test file contents into AI prompts. The strategy
    prompt uses only a structured file-count summary (§6.3). Implementors adding full
    content injection must do so in accordance with the
    [AI Prompt Contract §3](./ai_prompt_contract.md).
