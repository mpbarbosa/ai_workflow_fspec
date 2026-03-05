# Step 01: Documentation Analysis — Functional Specification

**Step identifier:** `step_01`
**Step name:** Documentation Analysis
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 01 analyses and updates project documentation in response to recent code changes. It
combines structural validation (file presence, version consistency, inline documentation
coverage) with AI-powered content review using the `documentation_expert` persona.

Step 01 operates on the change set established by Step 00. It does not re-derive the full
list of changed files from scratch; instead it accepts the authoritative list produced by
Step 00 and focuses only on files relevant to documentation.

---

## 2. Step Kind Conformance

Step 01 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **Yes** — multiple conditions produce a graceful skip |
| AI-integrated | **Yes** — calls the Copilot API via `aiHelper` and `aiCache` |
| Dependencies | Step 00 output (via `options.modifiedFiles`) is preferred but not required |

**Dual calling convention.** Step 01 accepts two execute signatures for compatibility:

- **Context-style** (preferred): `execute({ projectRoot, modifiedFiles, … })` — the first
  argument is the workflow context object.
- **Legacy / test style**: `execute(projectRoot, options)` — separate path and options.

Both conventions produce identical behaviour. The step detects which style is in use at
runtime.

**Error handling deviation.** Step 01 re-throws unhandled exceptions from `execute`
rather than returning `{ success: false, error }`. This deviates from the step contract
rule that forbids propagating exceptions. Callers should be prepared to catch.

---

## 3. Domain Concepts

### 3.1 Changed-File Classification

Every changed file is assigned to exactly one documentation-impact category. The
categories are:

| Category | Matching Rule |
|---|---|
| `documentation` | File has a `.md` extension **or** path contains `docs/` |
| `tests` | File has a `.test.*` extension **or** path contains `test/` |
| `config` | File has a `.json`, `.yaml`, or `.yml` extension **or** path contains `config` |
| `source` | File has a `.js`, `.mjs`, `.ts`, or `.tsx` extension |

Files whose path begins with `.ai_workflow/` or `.workflow_core/` are **excluded** from
classification entirely; they are internal workflow infrastructure and must not appear in
prompts.

A file that matches multiple rules is assigned by first match in the order above
(`documentation` before `tests` before `config` before `source`).

Files matching none of the above rules are not assigned to any category and are not counted
in the totals.

### 3.2 AI Gate Conditions

Step 01 applies a gate before invoking the AI to avoid unnecessary API calls:

| Condition | Skip? |
|---|---|
| Total changed files = 0 | Yes |
| Only documentation files changed **and** `skipDocsOnly` option is `true` | Yes |
| No source files changed **and** `requireSource` option is `true` | Yes |
| Any other combination | No — proceed with AI analysis |

### 3.3 Incremental Detection

When incremental mode is enabled (default), Step 01 narrows the documentation file list
to only those that have actually changed since the last successful run, using a persistent
change-tracking store. This avoids re-analysing documentation files that are unmodified.

When incremental detection finds zero changed documentation files, the step skips with
reason `docs_unchanged`.

### 3.4 Structural Validation Checks

Three independent validation checks are performed before the AI is consulted:

| Check | What is tested |
|---|---|
| **File count** | A README file is present in the documentation set; exactly one README exists; at least one Markdown file is present |
| **Version references** | Version strings found in each documentation file match the version declared in the project manifest (e.g. `package.json`). Version strings in the form `X.Y.Z` or `vX.Y.Z` are matched. |
| **Inline documentation coverage** | Changed source files with code-documentation extensions (`.ts`, `.tsx`, `.js`, `.mjs`) are checked for a module-level documentation block and for undocumented exported symbols. |

All three checks are independent; a failure in one does not prevent the others from
running.

**File count check:** Identifies missing or duplicate README files and an empty
documentation set.

**Version reference check:** Skipped entirely if no project manifest exists or if the
manifest cannot be parsed. When it runs, each documentation file is checked independently.
Multiple version mismatches within one file are reported separately.

**Inline documentation coverage:** For each qualifying source file:
- The file must have a module-level documentation block (e.g. a `@module` or `@fileoverview` tag).
- Every exported declaration (class, function, constant, interface, type, enum) must have a documentation block immediately preceding it.

### 3.5 Parallel AI Analysis

When parallel mode is enabled (default), Step 01 dispatches documentation files to the AI
in parallel batches rather than serially. The batch strategy and maximum concurrency are
configurable. The default strategy is `BALANCED`; the default maximum concurrency is `4`.

Each batch receives a separate AI request. The `documentation_expert` persona is used for
all requests.

---

## 4. Constructor Dependencies

Step 01 accepts all collaborators via a single options map at construction time. All
dependencies have built-in fallback defaults; the step can instantiate them itself when not
provided.

| Dependency key | Role | When absent |
|---|---|---|
| `gitOps` | Queries git for the list of modified files (fallback source) | Falls back to a new git operations instance |
| `fileOps` | Reads file contents (for validation and prompt construction) | Falls back to a new file operations instance |
| `backlog` | Writes the step summary to the workflow artifact store | Falls back to a new backlog instance |
| `aiCache` | Caches and retrieves AI responses by cache key | Falls back to a new AI cache instance |
| `promptBuilder` | Constructs AI prompts | Falls back to a new prompt builder instance |
| `aiHelper` | Executes AI API requests | Falls back to a new AI helper instance |
| `incrementalProcessor` | Detects which documentation files have actually changed | Falls back to a new incremental processor instance |
| `parallelProcessor` | Dispatches documentation files to AI in parallel batches | Falls back to a new parallel processor instance |
| `configManager` | Provides project configuration (language, kind, prompts directory) | Optional; tech stack and project kind are omitted from prompts |
| `enableParallel` | Boolean; enables parallel AI dispatch | Defaults to `true` |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Source | Description |
|---|---|---|
| `projectRoot` | Caller | Absolute path to the project root |
| `options.modifiedFiles` | Preferably step_00 `contextUpdate` | Authoritative list of changed file paths. When present and non-empty, this list is used directly. When absent, the step queries git. |
| `options.enableIncremental` | Caller | Enables incremental detection. Default: `true`. |
| `options.skipDocsOnly` | Caller | When `true`, skip AI analysis if only documentation files changed. |
| `options.requireSource` | Caller | When `true`, skip AI analysis if no source files changed. |
| `options.parallelStrategy` | Caller | Batch strategy for the parallel processor. Default: `'BALANCED'`. |
| `options.maxConcurrency` | Caller | Maximum concurrent AI requests. Default: `4`. |

### 5.2 Execution Phases

Step 01 runs seven sequential phases:

#### Phase 1 — Detect Changes

Obtain the list of changed files. Prefer `options.modifiedFiles` (the authoritative list
from Step 00) over a fresh git query. If the resolved list is empty, skip immediately with
reason `no_changes`.

#### Phase 2 — Classify Changes

Apply the file classification rules (§3.1) to every changed file, producing per-category
lists and counts. Workflow artifacts are excluded at this stage.

#### Phase 3 — AI Gate

Apply the gate conditions (§3.2). If any condition is met, skip with reason `not_needed`
and include the classification in the skip result.

#### Phase 4 — Incremental Detection

When `options.enableIncremental !== false` and the documentation category is non-empty:
- Query the incremental processor for the subset of documentation files that have changed
  since the last run.
- If the subset is empty, skip with reason `docs_unchanged`.
- Otherwise, continue with only the changed subset.

#### Phase 5 — Structural Validation

Run the three validation checks (§3.4) against the narrowed documentation file list and
the classified source files. Validation runs regardless of whether the AI is available.

The AI helper is initialised at this point and the AI response cache is prepared before
validation completes, so they are ready for Phase 6. The `workingDirectory` for the AI
session is set to `projectRoot` to ensure the target project's AI configuration is used
rather than the workflow engine's own configuration.

#### Phase 6 — Parallel AI Analysis

Runs when:
- `enableParallel` is `true`, **and**
- At least one documentation file remains to process.

For each batch of documentation files dispatched by the parallel processor:

1. **Collect file contents.** Read the actual content of each documentation file in the
   batch plus all files from classified categories (documentation, source, tests, config).
   De-duplicate the combined list. For each file, read and wrap the content in a labelled
   block. Respect the total character budget (see [AI Prompt Contract §3](./ai_prompt_contract.md)).
   Skip unreadable files silently.

2. **Build the prompt.** Attempt to load the `doc_analysis_prompt` template from the
   shared AI helpers configuration. If successful, populate it with:
   - `project_name` — project kind identifier, or the project root path if unavailable
   - `primary_language` — from project configuration, or `"unknown"`
   - `changed_files` — comma-separated list of all classified changed files
   - `doc_files` — comma-separated list of files in this batch
   - `file_contents` — the labelled file content blocks assembled in step 1

   If the template cannot be loaded, fall back to the built-in documentation analysis
   prompt builder and append the file contents section manually.

3. **Prepend project-kind role (optional).** Attempt to load the project-kind
   configuration. If a `documentation_specialist` role is defined for the project's kind,
   prepend it to the prompt as a role declaration.

4. **Execute via cache.** The cache key is `documentation_expert|{file1},{file2},…`.
   If a cached response exists, return it. Otherwise, submit the prompt to the AI API
   using the `documentation_expert` persona and cache the response.

When the AI is unavailable (initialisation fails), each batch is skipped with reason
`ai_unavailable`.

#### Phase 7 — Save to Backlog

Write a structured Markdown summary to the workflow backlog under step `1`, section
`"Documentation Analysis"`. The summary includes:
- Changed file counts per category
- Validation results (pass/fail, all issues listed)
- Parallel analysis statistics (files processed, total time, per-file average, speedup
  factor if available)

### 5.3 Skip Conditions

| Reason | Condition |
|---|---|
| `no_changes` | Changed file list is empty after resolution |
| `not_needed` | AI gate (§3.2) determined analysis is unnecessary |
| `docs_unchanged` | Incremental detection found no changed documentation files |

All skip results return `{ success: true, skipped: true, reason: <value> }`.

---

## 6. Data Shapes

### 6.1 StepResult (success, not skipped)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `classification` | Object | File classification from Phase 2 (§6.2) |
| `validation` | Object | Validation results from Phase 5 (§6.3) |
| `analysis` | Object or null | Parallel AI analysis statistics from Phase 6 (§6.4); null when parallel analysis did not run |
| `filesProcessed` | Integer | Count of documentation files that entered Phase 6 |

### 6.2 Classification

| Field | Type | Description |
|---|---|---|
| `documentation` | String[] | Documentation file paths |
| `source` | String[] | Source file paths |
| `tests` | String[] | Test file paths |
| `config` | String[] | Configuration file paths |
| `counts.documentation` | Integer | Count of documentation files |
| `counts.source` | Integer | Count of source files |
| `counts.tests` | Integer | Count of test files |
| `counts.config` | Integer | Count of config files |
| `counts.total` | Integer | Total changed files (before classification exclusions) |

### 6.3 Validation Result

| Field | Type | Description |
|---|---|---|
| `success` | Boolean | `true` when `totalIssues` is zero |
| `totalIssues` | Integer | Sum of issues across all three checks |
| `fileCount` | Object | `{ success, issues[] }` — file count check result |
| `versionRefs` | Object | `{ success, issues[] }` — version reference check result; each issue has `file` and `mismatches[]` |
| `sourceDoc` | Object | `{ success, issues[], totalIssues, fileResults[] }` — inline documentation coverage result |

### 6.4 Analysis Statistics

| Field | Type | Description |
|---|---|---|
| `stats.processed` | Integer | Number of documentation files processed by AI |
| `stats.totalTime` | Integer | Total wall-clock time in milliseconds for parallel execution |
| `stats.speedup` | Number or null | Measured speedup ratio versus estimated serial time; null when not available |

### 6.5 StepResult (skip)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `reason` | String | One of: `no_changes`, `not_needed`, `docs_unchanged` |
| `classification` | Object | Present when reason is `not_needed`; absent otherwise |

---

## 7. Constraints and Rules

1. Step 01 **must** prefer `options.modifiedFiles` over a git query when the option is
   present and non-empty. Performing a redundant git query when Step 00's list is available
   is a contract violation.

2. Workflow artifact directories (`.ai_workflow/`, `.workflow_core/`) **must** be excluded
   from file classification. They must never appear in AI prompts.

3. The AI response cache key **must** encode the persona and the exact set of files in each
   batch. Using a different key risks serving a cached response built for a different set
   of files.

4. File content injection **must** comply with the [AI Prompt Contract §3](./ai_prompt_contract.md):
   each file read in a `try/catch`, total character budget respected, `buildFileContentBlock`
   used for each file.

5. The AI session's working directory **must** be set to `projectRoot` before the first AI
   call. This ensures the target project's AI configuration (e.g. custom instructions) is
   used, not the workflow engine's own configuration.

6. The structural validation (Phase 5) **must** run regardless of AI availability. It
   provides value without the AI and its results are always included in the return value
   and the backlog.

7. Incremental detection **must** be honoured. When the incremental processor reports zero
   changed documentation files, the step must skip. Bypassing the incremental check and
   re-processing unchanged files wastes tokens and time.

8. Parallel mode statistics (speedup, total time) are informational. A missing or zero
   speedup value must not be treated as a failure.
