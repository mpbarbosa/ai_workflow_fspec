# Step 0b: Bootstrap Documentation — Functional Specification

**Step identifier:** `step_0b`
**Step name:** Bootstrap Documentation
**Step kind:** `CONTEXT` (see [Step Contract §2.2](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 0b audits a project's documentation coverage and, when that coverage is insufficient,
generates the missing documentation files using an AI `technical_writer` persona.

The step runs early in the workflow — before documentation analysis or consistency checks —
so that downstream steps operate on a project with at least a baseline documentation
structure rather than encountering entirely absent files.

Step 0b answers three questions:

1. **Does the project need documentation bootstrapping?** — based on file counts, README
   size, and the presence of a `docs/` directory and CHANGELOG.
2. **Which documentation files are missing, and how critical are they?** — partitioned into
   critical, important, and optional tiers.
3. **Can the missing files be generated now?** — using AI, subject to a state cache that
   avoids redundant token expenditure when the project's documentation has not changed since
   the last successful generation run.

---

## 2. Step Kind Conformance

Step 0b conforms to the **CONTEXT** step kind as defined in `step_contract.md §2.2`.

| Property | Value |
|---|---|
| Kind identifier | `"context"` |
| Execute signature | `execute(context) → Promise<StepResult>` |
| Can be skipped | **Yes** — when documentation coverage is already adequate |
| AI-integrated | **Yes** — calls the AI API via `aiHelper` using the `technical_writer` persona |
| Estimated duration | Variable; depends on number of missing files and AI response latency |

**Logging convention.** Step 0b receives its logger via the constructor options map and
stores it as `this.logger`. This is consistent with the CONTEXT step convention.

**Error handling.** Step 0b wraps all execution in a top-level error handler. On failure it
writes an error entry to the backlog, then returns `{ success: false, error: <message> }`.
This conforms to the step contract rule that prohibits propagating unhandled exceptions.

**AI caching.** Step 0b does **not** use the `aiCache` / `withCache` pattern described in
the AI Prompt Contract. Instead it uses a dedicated state cache (`stateCache`) that tracks
a fingerprint of the project's existing documentation files. The `aiHelper` is called
directly without a response-level cache. The state cache prevents token expenditure when
the documentation set has not changed since the last generation run.

---

## 3. Domain Concepts

### 3.1 Documentation Coverage Check

Step 0b evaluates project documentation coverage through four independent conditions. A
bootstrap is needed if **any** condition is true:

| Condition | Threshold |
|---|---|
| README is absent or too small | README size < 500 bytes |
| Insufficient documentation files | Documentation file count < 2, **or** `docs/` directory is absent |
| CHANGELOG is absent | No `CHANGELOG.md` found in project root |

When all four conditions are false, the project is considered adequately documented and the
step skips.

### 3.2 Tracked Documentation File Types

Step 0b tracks a fixed set of canonical documentation files. Any file from this set that
is not found on disk is classified as missing:

| File path | Category |
|---|---|
| `README.md` | Critical |
| `CHANGELOG.md` | Important |
| `CONTRIBUTING.md` | Important |
| `LICENSE` | Important |
| `docs/API.md` | Optional |
| `docs/ARCHITECTURE.md` | Optional |
| `docs/GETTING_STARTED.md` | Optional |

Matching is case-insensitive.

### 3.3 Priority Tiers

Missing documentation files are classified into three priority tiers:

| Tier | Files | Meaning |
|---|---|---|
| Critical | `README.md` | The project cannot be understood without this file |
| Important | `CHANGELOG.md`, `CONTRIBUTING.md`, `LICENSE` | Standard project governance files that should exist |
| Optional | All `docs/*.md` files | Useful additions; absence is not blocking |

### 3.4 Source File Extensions

When counting source files and determining the primary programming language, Step 0b
recognises the following file extensions: `.js`, `.ts`, `.py`, `.sh`, `.go`, `.java`,
`.rs`, `.rb`.

The primary language is determined by the extension with the highest file count across the
project directory.

### 3.5 Documentation State Cache

The state cache prevents redundant AI token expenditure. It stores a fingerprint of the
existing documentation file set. Before invoking the AI, Step 0b:

1. Reads the current documentation files and their contents to produce a fingerprint.
2. Compares the fingerprint to the stored value.
3. If the fingerprint matches (the documentation has not changed since the last run that
   produced no new files), skips AI generation with `cachedSkip: true`.
4. After a generation run that produces zero files, persists the current fingerprint so the
   next run can skip.
5. After a generation run that produces one or more files, invalidates the cache so the
   next run re-evaluates.

### 3.6 Gitignore Exclusions

Before scanning the project directory, Step 0b attempts to read `.gitignore` from the
project root. Simple bare directory and file names (no wildcards, no negations, no nested
paths) are extracted and used as scan exclusions. This prevents scanning generated or
dependency directories that are excluded from source control.

---

## 4. Constructor Dependencies

Step 0b receives all external collaborators at construction time via a single options map.
All dependencies have default values; the step instantiates them itself when not provided.

| Dependency key | Role | When absent |
|---|---|---|
| `logger` | Logs progress, warnings, and errors during execution | Falls back to a new Logger instance |
| `fileOps` | Reads files and lists directory contents recursively | Falls back to a new FileOperations instance |
| `backlog` | Writes gap-analysis reports and error entries to the workflow artifact store | Falls back to a new Backlog instance |
| `aiHelper` | Executes AI API requests | Falls back to a new AiHelper instance (using `promptsDir` option) |
| `stateCache` | Fingerprints the documentation set to prevent redundant AI calls | Falls back to a new Step0bStateCache instance (using `cacheDir` and `cacheTtlSeconds` options) |
| `dryRun` | Boolean flag; when `true`, execution previews actions without reading files or calling AI | Defaults to `false` |
| `projectRoot` | Base directory for file operations | Defaults to the process working directory |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Type | Description |
|---|---|---|
| `context` | WorkflowContext object | Full workflow context. `context.projectRoot` overrides the constructor `projectRoot` when present. `context.projectDescription` and `context.projectName` are used in the AI prompt when provided. |

### 5.2 Execution Phases

Step 0b runs the following phases in order:

#### Phase 1 — Dry-Run Check

When the `dryRun` flag is `true`, the step logs a preview of what would be performed
(scan, identify, generate) and returns immediately with `{ success: true, dryRun: true }`.
No file operations or AI calls are made.

#### Phase 2 — Gather Project Statistics

Scan the project directory recursively, applying gitignore exclusions (§3.6). From the
resulting file list, compute:

- **Documentation file count** (`docCount`): files ending in `.md` or named `LICENSE`
  (case-insensitive).
- **Source file count** (`sourceCount`): files matching any tracked source extension (§3.4).
- **Primary language** (`primaryLanguage`): the language with the most source files.
- **README size in bytes** (`readmeSize`): size of `README.md` if it exists; zero otherwise.
- **Has CHANGELOG** (`hasChangelog`): whether any file named `changelog.md`
  (case-insensitive) exists.
- **Has docs directory** (`hasDocsDir`): whether any file begins with `docs/` or equals
  `docs`.

#### Phase 3 — Evaluate Coverage

Apply the bootstrap conditions (§3.1) to the statistics from Phase 2.

If no bootstrap is needed:
- Write a skip summary to the backlog.
- Return `{ success: true, skipped: true, reason: 'documentation coverage adequate' }`.

Execution stops here for adequately-documented projects.

#### Phase 4 — Identify Missing Documentation

List all existing documentation files (`.md` files and `LICENSE`) relative to the project
root. Compare them against the tracked file set (§3.2) using case-insensitive matching.
Compute the list of missing files and their priority tier (§3.3).

#### Phase 5 — Generate Gap Analysis Report

Format a structured Markdown report describing the documentation statistics and the missing
files (by tier). Write the report to the backlog under step `0b`, section `Bootstrap_Docs`.

#### Phase 6 — AI Documentation Generation

Runs only when one or more documentation files are identified as missing.

**6a — State cache check.** Read the existing documentation files and their contents.
Compare the fingerprint of this set to the stored state cache value. If the fingerprint
matches (unchanged since the last zero-result run), return immediately with
`{ success: true, …, cachedSkip: true }`. Skip AI generation to avoid redundant token
expenditure.

**6b — AI initialisation.** Initialise the AI helper. If the AI is not available, log a
warning and proceed to the result without generating any files.

**6c — Load configuration.** Attempt to read `ai_helpers.yaml` from the package's
`.workflow_core/config/` directory to load the `technical_writer_prompt` template
configuration. When unavailable, an inline fallback template is used. Separately, attempt
to read `.workflow-config.yaml` from the project root to obtain the project description
and confirmed primary language; fall back to context values and auto-detected values when
unavailable.

**6d — Build prompt.** Construct the AI prompt using the `technical_writer` persona. The
prompt includes:
- Project name (from context, or derived from the project root directory name)
- Project description (from configuration or context)
- Primary programming language
- Counts of existing source and documentation files
- The list of missing documentation files

When the YAML configuration is available, the prompt is assembled from the
`technical_writer_prompt.role_prefix` and `task_template` fields, with placeholder
substitution. Otherwise the inline fallback template is used.

**6e — Execute AI request.** Submit the prompt to the AI API using the `technical_writer`
persona. The step calls `aiHelper.executeRequest` directly (not via `aiCache.withCache`).

**6f — Parse and write generated files.** Parse the AI response to extract
filename–content pairs. Each successful entry is written to the project root as a new
documentation file. Generated filenames are accumulated in the `generated` list.

**6g — Update state cache.** If the AI ran but produced zero files, persist the current
documentation fingerprint so the next run can skip. If files were generated, invalidate
the cache so the next run re-evaluates the documentation set.

### 5.3 Skip Conditions

| Reason | Condition |
|---|---|
| `documentation coverage adequate` | Coverage check (Phase 3) determined no bootstrap is needed |
| `cachedSkip` (field, not `reason`) | State cache fingerprint matches; AI generation deliberately omitted |

The `documentation coverage adequate` skip returns `{ success: true, skipped: true, reason: '…' }`.

The state cache short-circuit returns a full success result (not a canonical `skipped`
shape) with `cachedSkip: true`.

---

## 6. Data Shapes

### 6.1 ProjectStats

Computed during Phase 2.

| Field | Type | Description |
|---|---|---|
| `docCount` | Integer | Count of documentation files (`.md` files + `LICENSE`) |
| `sourceCount` | Integer | Count of source files matching tracked extensions |
| `readmeSize` | Integer | Size of `README.md` in bytes; zero when absent |
| `hasChangelog` | Boolean | Whether a `CHANGELOG.md` file exists |
| `hasDocsDir` | Boolean | Whether a `docs/` directory exists |
| `primaryLanguage` | String | Dominant programming language by file count |

### 6.2 CategorizedMissingDocs

| Field | Type | Description |
|---|---|---|
| `critical` | String[] | Paths of critical missing files (e.g. `README.md`) |
| `important` | String[] | Paths of important missing files (e.g. `CHANGELOG.md`) |
| `optional` | String[] | Paths of optional missing files (e.g. `docs/API.md`) |

### 6.3 StepResult (success, not skipped)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `missingDocs` | String[] | All missing documentation file paths |
| `categorized` | CategorizedMissingDocs | Missing files by priority tier (§6.2) |
| `stats` | ProjectStats | Project statistics collected in Phase 2 (§6.1) |
| `generated` | String[] | Paths of files actually written to disk by the AI |
| `cachedSkip` | Boolean | `true` when AI generation was skipped due to state cache hit |
| `duration` | Integer | Wall-clock time in milliseconds from start of execute to return |

### 6.4 StepResult (skip — coverage adequate)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `reason` | `"documentation coverage adequate"` | Human-readable explanation |

### 6.5 StepResult (failure)

| Field | Type | Description |
|---|---|---|
| `success` | `false` | — |
| `error` | String | Error message |
| `duration` | Integer | Wall-clock time in milliseconds |

---

## 7. Constraints and Rules

1. Step 0b **must** write a backlog entry regardless of whether bootstrap is needed. When
   coverage is adequate, a skip summary is written. When bootstrapping is performed, a
   gap-analysis report is written. On failure, an error entry is written.

2. Step 0b **must** check the state cache before invoking the AI. Bypassing the cache and
   repeating an AI call when the documentation set is unchanged wastes tokens and produces
   no new output.

3. When the AI is unavailable (initialisation fails), Step 0b **must not** fail. It must
   log a warning and return a success result with an empty `generated` list.

4. When `dryRun` is `true`, Step 0b **must not** perform any file operations, AI calls, or
   backlog writes. It must return immediately with a dry-run indicator.

5. The state cache **must** be updated after every AI generation attempt:
   - Zero files generated → persist current fingerprint.
   - One or more files generated → invalidate the cache.

6. The `projectRoot` provided via the `context` argument at execute time **must** take
   precedence over the `projectRoot` supplied at construction time.

7. Step 0b does **not** use `aiCache.withCache`. AI responses for documentation generation
   are not persisted in an AI response cache. The state cache (§3.5) serves a different
   purpose: it prevents the AI from being invoked at all when the documentation set is
   unchanged, rather than returning a previously-stored AI response.

8. Gitignore parsing **must** be lenient. Any line with wildcards, glob characters,
   negations, or path separators must be skipped. Only simple bare names (with an optional
   trailing `/`) are extracted as exclusions. A parse failure must not abort execution.
