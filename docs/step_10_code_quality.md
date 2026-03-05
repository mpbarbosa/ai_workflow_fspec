# Step 10: Code Quality Analysis — Functional Specification

**Step identifier:** `step_10`
**Step name:** Code Quality Analysis
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 10 analyses the static code quality of the project by running language-appropriate
linters across all detected source languages and then enriching the linter output with
an AI-powered review using the `code_quality_analyst` persona.

Step 10 answers three questions:

1. **What languages are present, and do they have linters?** — the step auto-detects all
   source languages in the project and resolves the correct linter command for each.
2. **What is the linter output?** — for each language, the appropriate linter is executed
   and its output is parsed into a structured issue count broken down by errors, warnings,
   and affected files.
3. **What does an AI reviewer conclude about code quality?** — a rotating partition of
   source files is submitted to the AI for qualitative review, covering maintainability,
   anti-patterns, and technical debt.

---

## 2. Step Kind Conformance

Step 10 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **Partially** — the step returns `success: true, skipped: true` when no linter is configured or no source files are found; it does not formally skip for other reasons |
| AI-integrated | **Yes** — calls the AI API via `aiHelper` and `aiCache` |
| Conforms to AI Prompt Contract | **Yes** — file contents are injected via `buildFileContentMap` / `formatFileContentMap`; no prohibited prompt techniques are used; the prompt includes an explicit anti-hallucination guard |

**Error handling deviation.** Step 10 does **not** conform to the step contract rule
requiring that `execute` never throw. When an unhandled error occurs in the main
execution body, the `catch` block logs the error and **re-throws** it rather than
returning `{ success: false, error: … }`. AI-phase errors are caught separately and
treated as non-fatal warnings. See §7 for details.

---

## 3. Domain Concepts

### 3.1 Source Languages

The step supports the following languages, each with a default linter command and a
set of recognised source file extensions:

| Language | Default Linter Command | Source Extensions |
|---|---|---|
| `javascript` | `npm run lint` | `.js`, `.jsx`, `.mjs`, `.cjs` |
| `typescript` | `npm run lint` | `.ts`, `.tsx` |
| `python` | `flake8 .` | `.py` |
| `go` | `golint ./...` | `.go` |
| `java` | `mvn checkstyle:check` | `.java` |
| `ruby` | `rubocop` | `.rb` |
| `rust` | `cargo clippy` | `.rs` |
| `bash` | `find . -name "*.sh" … \| xargs shellcheck` | `.sh`, `.bash` |
| `json` | *(native validation)* | `.json` |

Language detection always supplements the tech-stack result with `bash` and `json`,
since those are commonly present but may not be reported by the tech-stack detector.

### 3.2 Linter Command Resolution

The effective linter command for a language is resolved in priority order:

1. **Project config override** — a `lint_commands` map in `.workflow-config.yaml`
   under the `tech_stack` section. If a key matching the language is found, its value
   is used as-is.
2. **Default command** — the built-in default from §3.1.

Languages for which neither source applies are excluded from linting entirely.

### 3.3 JSON Native Validation

When the detected language set includes `json`, source `.json` files are validated by
parsing each file rather than invoking an external linter process. Files matching
`tsconfig*.json` and `*.jsonc` are excluded to avoid false positives from the JSONC
comment syntax. A parse failure is reported as one lint error.

### 3.4 Quality Rating

After running a linter, a per-language quality rating is derived from the **issue rate**
(issues per source file):

| Rating | Issue Rate |
|---|---|
| `excellent` | 0 issues/file |
| `good` | > 0 and ≤ 5 issues/file |
| `moderate` | > 5 and ≤ 20 issues/file |
| `poor` | > 20 issues/file |

### 3.5 Partition-and-Rotate Strategy

The AI review does not process all source files in a single call. Instead:

1. **Active candidates** — source files eligible for review are filtered to those that
   are recently modified or have not yet been reviewed in recent runs.
2. **Current partition** — a slice of the active candidates is selected as the
   partition for this run. The partition advances on each run, cycling through all
   active candidates over time.
3. **Slicing for prompt size** — the partition is subdivided into slices of at most
   `AI_FILES_PER_SLICE = 5` files each. Each slice is submitted as a separate AI
   request.
4. **Rotation** — after a successful AI review, quality scores are updated and the
   partition pointer advances so that the next run reviews a different set of files.

This strategy ensures that large file sets are reviewed progressively without exceeding
prompt size limits.

### 3.6 File Content Injection

File contents are injected into the AI prompt using `buildFileContentMap` and
`formatFileContentMap`. Source files are prioritised before test files. Each file
excerpt is capped at 5,000 characters. The resulting formatted string is passed as the
`file_content_map` variable in the prompt template, or as a direct prompt variable in
the fallback builder.

This satisfies the [AI Prompt Contract §3](./ai_prompt_contract.md) requirement that
file contents must be injected rather than referenced by name.

---

## 4. Constructor Dependencies

Step 10 receives all collaborators via a single options map at construction time.

| Dependency key | Role | When absent |
|---|---|---|
| `executor` | Executes linter commands in a subprocess | Falls back to the global executor module |
| `fileOps` | Reads file contents for AI prompt injection and config reading | Falls back to a new `FileOperations` instance |
| `backlog` | Writes step summary and AI review to the workflow artifact store | Falls back to a new `Backlog` instance |
| `techStack` | Detects all languages present in the project | Falls back to a new `TechStackDetector` instance |
| `aiHelper` | Executes AI API requests for code quality review | Falls back to a new `AiHelper` instance |
| `aiCache` | Caches and retrieves AI responses by cache key | Falls back to a new `AiCache` instance |
| `analysisCache` | Caches linter results to avoid re-running on unchanged inputs | Falls back to a new `AnalysisCache` instance |
| `promptsDir` | Directory path for AI prompt templates | Passed to `AiHelper` constructor; defaults to `null` |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the root of the project under analysis |
| `options` | Optional map of additional orchestrator-supplied hints |

Relevant `options` fields:

| Field | Description |
|---|---|
| `modifiedFiles` | List of modified file paths from Step 00 context. Used to determine active AI review candidates. |
| `projectKind` | Project kind identifier (e.g. `react_spa`). Used to select the project-kind role override for the AI prompt. |
| `projectDescription` | Human-readable project description. Included in the AI prompt. |
| `changeScope` | Change scope string from Step 00. Included in the AI prompt. |
| `sessionLogFile` | Path to the current session log file. When present along with `sessionLogContent`, a supplementary issue-extraction prompt is appended. |
| `sessionLogContent` | Contents of the session log. Used with `sessionLogFile`. |

### 5.2 Execution Phases

Step 10 runs eight sequential phases:

#### Phase 1 — Detect Languages

Run tech-stack detection on `projectRoot`. The result is supplemented with `bash` and
`json` to ensure those languages are always considered. Log the full list of detected
languages.

#### Phase 2 — Load Config Lint Commands

Attempt to read `${projectRoot}/.workflow-config.yaml` and extract a `lint_commands`
map from the `tech_stack` section. If the file is absent or unreadable, the map is
empty and all languages use their default commands.

#### Phase 3 — Build Per-Language Linter Map

For each detected language, resolve the effective linter command (§3.2). Languages
with no resolvable command are excluded. If the resulting map is empty (no linter
configured for any language), proceed to the **no-linter path**:

- Identify the primary language from the tech-stack result.
- Count source files for that language.
- Save a skipped report to the backlog.
- Return `{ success: true, language, sourceFileCount, skipped: true }`.

#### Phase 4 — Run Linters

For each language in the linter map:

1. Count source files by globbing for the language's extensions under `projectRoot`,
   excluding `node_modules`, `dist`, `build`, `.git`, virtual environments, coverage
   output, and test directories.
2. If no source files are found for the language, skip it without running the linter.
3. Otherwise, check the analysis cache for a previously computed linter result for
   this `(projectRoot, language, command)` combination. If cached, use the cached
   result. If not cached, run the linter (or native JSON validation for `json`) and
   store the result.
4. Parse the linter output using the language-appropriate parser:
   - **ESLint** (javascript/typescript): parse the `✖ N problems` summary line and per-file
     issue counts from stylish output.
   - **Flake8** (python): count output lines and group by file path.
   - **Shellcheck** (bash): count `In file line N:` markers; classify by `(error)`,
     `(warning)`, `(info)`, `(style)`, and `(note)` tags.
   - **Generic fallback**: count non-empty output lines.
5. Compute issue rate and quality rating (§3.4).
6. Collect all source file paths into a combined list for the AI phase.

#### Phase 5 — Handle Zero Source Files

If every language was skipped due to having zero source files, save a skipped report
to the backlog using the primary language and return
`{ success: true, language, sourceFileCount: 0, skipped: true }`.

#### Phase 6 — Compute Aggregate Totals

Sum issue counts, error counts, warning counts, info counts, and file counts across
all per-language results to produce aggregate totals.

#### Phase 7 — Generate and Save Multi-Language Report

Format a Markdown report covering:

- **Summary section**: language count, total files, total issues, errors, warnings, info.
- **Per-language sections**: language name, file count, linter command, issue count
  breakdown, issue rate, quality rating.
- **Recommendations section** (present when total issues > 0): prioritised fix advice.

Save the report to the backlog under step `10`, section `"Code Quality"`.

#### Phase 8 — AI Code Quality Review (Optional)

This phase runs only when the AI helper can be initialised successfully.

1. Initialise the AI response cache.
2. De-duplicate the combined source file list from Phase 4.
3. Determine the **active candidates** and select the **current partition** using the
   partition cache (§3.5). The partition is labelled with its index and total count.
4. Read the file contents of all files in the partition. Each file read is attempted
   individually; unreadable files are silently skipped.
5. Subdivide the partition into slices of at most `AI_FILES_PER_SLICE` files.
6. Read the shared AI helpers YAML once. Attempt to load the project-kind role
   override for the `code_quality_auditor` role from the project kinds YAML.
7. **For each slice** (submitted in parallel):
   a. Build the content map for the slice's files (§3.6).
   b. Attempt to build the prompt from the `step9_code_quality_prompt` template,
      populating it with project name, description, language, tech stack, change scope,
      modified count, file count, quality summary, the formatted quality report, file
      list, and file content map.
   c. If a project-kind role override was loaded, prepend it to the prompt.
   d. Conditionally append a supplementary `issue_extraction_prompt` when session log
      content is available.
   e. For front-end project kinds (`react_spa`, `client_spa`, `static_website`),
      append a `front_end_developer_prompt` supplementary section.
   f. If the YAML template fails, fall back to the built-in code quality prompt
      builder.
   g. The cache key encodes: `step_10|v2|p{partitionIndex}|s{sliceIndex}|{languages}|{totalIssues}|{contentHash}`.
      The content hash is an 8-character value derived from the first 80 characters of
      each file in the slice, sorted by path. This ensures cache freshness when file
      contents change.
   h. Submit the request via the AI cache using the `code_quality_analyst` persona
      with a 240-second timeout.
8. Concatenate non-empty AI section results with separators.
9. If any AI content was produced, prepend a partition header to the content and
   append it to the linter report. Save the enriched report to the backlog, replacing
   the earlier linter-only save.
10. Update quality scores in the partition cache for each file in the partition using
    per-file issue counts normalised to project-relative paths. Advance the partition
    pointer for the next run.

AI-phase errors (initialisation failures or per-request errors) are caught and logged
as warnings. They do not cause the step to return failure.

### 5.3 Skip Conditions

| Reason | Condition | Result shape |
|---|---|---|
| No linter configured | No language has a resolvable linter command | `{ success: true, language, sourceFileCount, skipped: true }` |
| No source files | All detected languages have zero qualifying source files | `{ success: true, language, sourceFileCount: 0, skipped: true }` |

---

## 6. Data Shapes

### 6.1 StepResult (success, full execution)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `hasLintErrors` | Boolean | `true` when the aggregate error count is > 0 |
| `perLanguageResults` | Object[] | Per-language result records (§6.2) |
| `aggregateTotals` | Object | Summed totals across all languages (§6.3) |
| `language` | String | Primary language (backward-compatibility alias) |
| `sourceFileCount` | Integer | Total source file count across all languages (backward-compatibility alias) |

### 6.2 Per-Language Result

| Field | Type | Description |
|---|---|---|
| `language` | String | Language name |
| `sourceFileCount` | Integer | Count of qualifying source files |
| `linterCommand` | String | The linter command that was run |
| `linterResults` | Object | Parsed output: `{ totalIssues, errors, warnings, files, fileIssues }` |
| `issueRate` | Number | Issues per source file (rounded to 1 decimal) |
| `qualityRating` | String | One of: `excellent`, `good`, `moderate`, `poor` |
| `skipped` | Boolean | `true` when the language was skipped |

### 6.3 Aggregate Totals

| Field | Type | Description |
|---|---|---|
| `totalIssues` | Integer | Sum of all issues across all languages |
| `errors` | Integer | Sum of all errors |
| `warnings` | Integer | Sum of all warnings |
| `infos` | Integer | Sum of all info-level messages |
| `fileCount` | Integer | Sum of all source files across all languages |

### 6.4 StepResult (skip)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `language` | String | The primary or fallback language used |
| `sourceFileCount` | Integer | Count of source files (0 for the no-files skip path) |

---

## 7. Constraints and Rules

1. **Error handling deviation.** Step 10 does **not** comply with the step contract
   rule (§5, rule 4) that forbids propagating exceptions from `execute`. Errors in the
   main execution body are re-thrown after logging. Callers **must** wrap calls to
   `execute` in a try/catch or equivalent. Errors that occur only in the AI phase
   (Phase 8) are caught and treated as warnings; they do not cause a re-throw.

2. The analysis cache (for linter results) **must** be keyed on
   `(projectRoot, language, linterCommand)`. A cache hit must prevent the linter
   subprocess from being run again for the same inputs.

3. The partition cache **must** be updated after each successful AI review. Failing to
   advance the partition or update quality scores would cause the same files to be
   reviewed repeatedly on consecutive runs.

4. File content injection into AI prompts **must** follow the
   [AI Prompt Contract §3](./ai_prompt_contract.md). Contents are capped per file (5,000
   characters) and the set of files per slice is capped at `AI_FILES_PER_SLICE`. Each
   file read must be attempted independently; a failure to read one file must not
   prevent the prompt from being built with the remaining files.

5. The AI prompt template key is **`step9_code_quality_prompt`** (note: the key name
   references step 9, not step 10). This reflects the shared nature of the YAML
   configuration; the same template is used for code quality analysis regardless of
   which step invokes it. Implementations must not rename the key without updating the
   YAML file.

6. The cache key **must** include a content hash derived from file contents, not just
   file paths. Using only file paths risks serving a stale cached response when file
   contents change.

7. The AI review (Phase 8) **must not** block or fail the step when the AI is
   unavailable. The linter report without AI enrichment is always a valid outcome.

8. Test directories and test file patterns **must** be excluded from source file
   counting (Phase 4). Linting source files is distinct from linting test files; mixing
   them inflates the file count and distorts the issue rate.
