# Step 16: Version Update — Functional Specification

**Step identifier:** `step_16`
**Step name:** Version Update
**Step kind:** `CONTEXT` (see [Step Contract §2.2](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 16 calculates the appropriate semantic version bump for the project based on the
current change set, increments the version, and propagates the new version string to
every file in the project that references it.

Step 16 answers four questions:

1. **What is the current version?** — discovered by reading project metadata files in
   a defined priority order.
2. **What bump type is required?** — determined first by heuristic thresholds applied
   to git change statistics, then optionally refined by the AI `devops_engineer` persona.
3. **Which files need updating?** — all project files containing the old version string
   are found by scanning the entire project, not just the modified-file list. Special
   handling is applied for JS version-config files and service-worker files.
4. **Is the project version consistent after the update?** — an optional `check:version`
   script (if declared in the project's package manifest) is run to verify consistency,
   with automatic remediation of any remaining mismatches.

Step 16 is positioned after the analysis and quality steps and before git finalization.

---

## 2. Step Kind Conformance

Step 16 conforms to the **CONTEXT** step kind as defined in `step_contract.md §2.2`.

| Property | Value |
|---|---|
| Kind identifier | `"context"` |
| Execute signature | `execute(context) → Promise<StepResult>` |
| Can be skipped | **Yes** — multiple conditions produce a graceful skip |
| AI-integrated | **Yes** — calls the AI API via `aiHelper` and `aiCache` for bump-type verification |
| Conforms to AI Prompt Contract | **Yes** — no file content from disk is embedded in the AI prompt; the prompt uses git statistics and a file list only; no `@workspace` or IDE-feature references are used |

**Error handling.** Step 16 **conforms** to the step contract error-handling rule.
All exceptions within `execute` are caught and returned as
`{ success: false, error: <message> }`. The step does not re-throw.

---

## 3. Domain Concepts

### 3.1 Semantic Version Format

Versions follow the format `X.Y.Z` where each component is a non-negative integer.
An optional pre-release suffix is supported: `X.Y.Z-prerelease` (e.g. `1.0.1-alpha`).
The pre-release suffix is preserved through version bumps.

### 3.2 Bump Types

Three bump types are defined, consistent with the Semantic Versioning specification:

| Type | Effect | Typical Trigger |
|---|---|---|
| `major` | `X+1.0.1` | Breaking changes; large deletions or high file-modified count |
| `minor` | `X.Y+1.0` | New features; new files added or significant insertions |
| `patch` | `X.Y.Z+1` | Bug fixes, documentation, refactoring, tests |

### 3.3 Heuristic Bump Type Determination

Before consulting the AI, the step determines a bump type from git statistics using
fixed thresholds:

| Threshold | Value | Bump Type |
|---|---|---|
| Deletions exceed | 500 | `major` |
| Modified files exceed | 20 | `major` |
| New files added (> 0) **or** insertions exceed | 100 | `minor` |
| Otherwise | — | `patch` |

Thresholds are evaluated in order: `major` takes priority over `minor` over `patch`.

### 3.4 Metadata Files

Version detection searches the following files in priority order. The first file
that yields a parseable version wins.

| Priority | File | Parsing method |
|---|---|---|
| 1 | `package.json` | Parse JSON; read `version` field |
| 2 | `pyproject.toml` | Regex scan for `version = "X.Y.Z"` pattern |
| 3 | `setup.py` | Regex scan for `version = "X.Y.Z"` pattern |
| 4 | `Cargo.toml` | Regex scan for `version = "X.Y.Z"` pattern |
| 5 | `.workflow-config.yaml` | Regex scan for `version: X.Y.Z` pattern |

### 3.5 Dry-Run Mode

When constructed with `dryRun: true`, the step logs a preview of the operations it
would perform but writes no files and makes no external calls. It returns
`{ success: true, dryRun: true, message: 'Version update dry run completed' }`.

### 3.6 JS Version-Config Files

Source files that export both a `VERSION` constant and a `BUILD_DATE` constant are
treated as version-config files. When updating these files, both the version string
and the build date (in `YYYY-MM-DD` format) are replaced simultaneously. The set of
version-config files is loaded from the `version.config_files` list in
`.workflow-config.yaml` and from the `context.versionConfig.configFiles` field.

### 3.7 Service-Worker Files

Source files that define a `CACHE_NAME` constant in the pattern
`const CACHE_NAME = 'name-vX.Y.Z-YYYYMMDD'` are treated as service-worker files.
When updating these files, the version component and the date suffix are both replaced.
The date suffix is refreshed to today's date in `YYYYMMDD` format to force cache
invalidation on next deployment. The set of service-worker files is loaded from the
`version.service_worker_files` list in `.workflow-config.yaml` and from
`context.versionConfig.serviceWorkerFiles`.

### 3.8 Version Consistency Check

If the project declares a `check:version` script in its package manifest, the step
runs it after performing version updates. The script validates that all version
references across the project are consistent with the new version.

If the script exits with a non-zero code, the step attempts to **auto-fix** reported
issues by parsing the script's text output and applying targeted corrections:

- Files with stale version strings have those strings replaced with the new version.
- Missing JS constant files are created with a default version-constant export.
- Files that are neither JS nor have identifiable stale versions are flagged for
  manual attention and are not modified.

After auto-fixing, the consistency check is re-run to confirm resolution.

### 3.9 Full-Project Version Scan

Rather than restricting updates to the modified-file list, Step 16 scans the entire
project directory tree for files that contain the old version string as a substring.
This ensures that version references in documentation, configuration files, and other
locations outside the active change set are not missed.

The scan skips the following directories and file types:

- **Directories:** `node_modules`, `.git`, `dist`, `build`, `coverage`, `.ai_workflow`,
  `.workflow_core`, `__pycache__`.
- **Extensions:** binary and minified file extensions (images, fonts, archives,
  lock files, minified scripts and stylesheets).

---

## 4. Constructor Dependencies

Step 16 receives all collaborators via a single options map at construction time.

| Dependency key | Role | When absent |
|---|---|---|
| `logger` | Logs step progress and phase transitions | Falls back to a new `Logger` instance |
| `fileOps` | Reads and writes files during version detection, update, and config loading | Falls back to a new `FileOperations` instance |
| `backlog` | Writes the step summary to the workflow artifact store | Falls back to a new `Backlog` instance |
| `aiHelper` | Executes AI API requests for bump-type verification | Falls back to a new `AiHelper` instance |
| `aiCache` | Caches and retrieves AI responses by cache key | Falls back to a new `AiCache` instance |
| `dryRun` | Boolean; when `true`, no files are written and no scripts are run | Defaults to `false` |
| `projectRoot` | Fallback project root when not supplied via context | Falls back to the process working directory |
| `promptsDir` | Directory path for AI prompt templates | Passed to `AiHelper` constructor |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Type | Description |
|---|---|---|
| `context` | WorkflowContext object | Full workflow context |

Relevant context fields:

| Field | Description |
|---|---|
| `projectRoot` | Absolute path to the project. Takes precedence over the constructor `projectRoot` option when present. |
| `modifiedFiles` | Authoritative list of changed file paths from Step 00. Used as the starting set for version-update candidates and as a guard condition (skip if empty). |
| `gitStats` | Git change statistics: `{ modifiedCount, addedCount, deletedCount, insertions, deletions, categories }`. Used by the heuristic bump-type determination. |
| `versionConfig` | Optional version configuration loaded from `.workflow-config.yaml`: `{ files[], configFiles[], serviceWorkerFiles[] }`. Merged with YAML-loaded lists. |

### 5.2 Skip Conditions

| Reason | Condition |
|---|---|
| `no modified files` | `context.modifiedFiles` is empty or absent |
| `no version found` | No parseable version string was found in any metadata file |

Both skip conditions are handled after a dry-run check. All skip results return
`{ success: true, skipped: true, reason: <value> }`.

### 5.3 Execution Phases

Step 16 runs five main phases:

#### Phase 1 — Detect Current Version

Read each metadata file from §3.4 in priority order until a valid version is found.
For `package.json`, parse the JSON and read the `version` field. For all other files,
apply the version-pattern regex and return the first match.

If no version is found across all metadata files, skip with reason `no version found`.

#### Phase 2 — Determine Bump Type

1. Apply the heuristic thresholds (§3.3) to `context.gitStats` to produce an initial
   bump type.
2. Attempt to **verify and refine with AI**:
   a. Initialise the AI helper. If unavailable, retain the heuristic result.
   b. Initialise the AI response cache.
   c. Gather git context by running git commands directly: `git log --oneline -n 10`,
      `git diff --shortstat HEAD`, and `git diff HEAD`. Parse insertions and deletions
      from the shortstat output. Limit the raw diff to the first 80 lines.
   d. Attempt to load expertise, approach, and output-format blocks from the
      `version_manager_prompt` key in the shared AI helpers YAML. These blocks are
      incorporated into the prompt structure. If the YAML cannot be loaded, generic
      text is used.
   e. Build a structured prompt containing: the current version, the heuristic
      recommendation, change statistics per category, the recent git log, the diff
      statistics, and the diff sample.
   f. The cache key encodes: `step_16|{currentVersion}|{heuristicBumpType}|{modifiedCount}|{insertions}|{deletions}`.
   g. Submit via the AI cache using the `devops_engineer` persona.
   h. Parse the AI response for a bump type using a three-tier extractor: first look
      for `Bump Type: <type>`, then for `recommend… <type>`, then for a bare one-word
      response. If a valid bump type is extracted, it replaces the heuristic value.
3. AI verification failures are caught and logged as warnings. The heuristic result is
   always the fallback.

#### Phase 3 — Calculate New Version

Apply the resolved bump type to the current version using standard semver arithmetic
(§3.2), preserving any pre-release suffix. Log the transition as
`{currentVersion} → {newVersion} ({bumpType})`.

#### Phase 4 — Update Versions in Files

Version update is performed in three sub-phases:

**Phase 4a — Core file update:**

1. Build the candidate file set by combining:
   - All metadata files from §3.4 (resolved to absolute paths).
   - All paths from `context.modifiedFiles` (resolved to absolute paths).
   - Additional files from `context.versionConfig.files` (resolved to absolute paths).
   - Results from the full-project version scan (§3.9), which searches the entire
     project directory tree for files containing the old version string.
2. For each candidate file:
   - Read the file. If the old version string is not present, record a skip.
   - For `package.json`, update only the `version` field by parsing and re-serialising
     the JSON. For all other files, replace all occurrences of the old version string.
   - If the replacement produces no change, record a skip.
   - Otherwise, write the updated content (unless dry-run) and record a success.
   - Handle `ENOENT` (file not found) as a skip, not an error.

**Phase 4b — JS version-config files:**

Update files identified as version-config files (§3.6). For each qualifying file,
replace both the `VERSION` constant value and the `BUILD_DATE` constant value. The
build date defaults to today in `YYYY-MM-DD` format.

**Phase 4c — Service-worker files:**

Update files identified as service-worker files (§3.7). For each qualifying file,
replace the version component within the `CACHE_NAME` string and refresh the date
suffix to today's date in `YYYYMMDD` format.

**Phase 4 (consistency check — optional):**

Run the `check:version` script if present (§3.8). If it exits non-zero, attempt
auto-fix and re-run. Auto-fixed files are merged into the updates list for reporting.

After all sub-phases, compute update statistics: count of successfully updated files,
skipped files, and failed files.

#### Phase 5 — Generate Report and Save to Backlog

Format a Markdown report (§6.3) from the version transition, update statistics, and
file update list. Save to the backlog under step `16`, section `"Version_Update"` with
a success indicator.

Return the final `StepResult` (§6.2).

---

## 6. Data Shapes

### 6.1 FileUpdateResult (internal)

Produced for each candidate file processed in Phase 4.

| Field | Type | Description |
|---|---|---|
| `file` | String | Relative file path |
| `success` | Boolean | `true` when the file was written successfully |
| `skipped` | Boolean | `true` when the file did not contain the old version or was not found |
| `autoFixed` | Boolean | `true` when the file was updated via consistency auto-fix |
| `error` | String | Error message when `success` is `false` and `skipped` is `false` |
| `reason` | String | Human-readable skip reason (e.g. `'file not found'`) |

### 6.2 StepResult (success)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `oldVersion` | String | The version detected in Phase 1 |
| `newVersion` | String | The incremented version |
| `bumpType` | String | The bump type used (`major`, `minor`, or `patch`) |
| `stats` | Object | `{ updated, skipped, failed }` counts |
| `updates` | Object[] | Full list of `FileUpdateResult` objects from all Phase 4 sub-phases |
| `consistencyCheck` | Object | `{ ran, exitCode, output, autoFix? }` from the consistency check |
| `duration` | Integer | Wall-clock time in milliseconds for the full execute call |

### 6.3 StepResult (skip)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `reason` | String | One of: `no modified files`, `no version found` |

### 6.4 StepResult (dry-run)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `dryRun` | `true` | — |
| `message` | String | `"Version update dry run completed"` |

### 6.5 StepResult (error)

| Field | Type | Description |
|---|---|---|
| `success` | `false` | — |
| `error` | String | Error message from the caught exception |
| `duration` | Integer | Wall-clock time in milliseconds up to the point of failure |

### 6.6 Backlog Report

The Markdown report written to the backlog includes:

- **Version Update section**: previous version, new version, bump type.
- **Update Statistics section**: count of updated, skipped, and failed files.
- **Files Updated section**: list of successfully updated file paths.
- **Failed Updates section** *(present only when failures > 0)*: list of failed paths
  and error messages.
- **Metadata section**: step version, analysis method, bump type, and next-step
  guidance.

---

## 7. Constraints and Rules

1. Step 16 **must** attempt the full-project version scan (§3.9) on every run. Relying
   solely on `context.modifiedFiles` would miss version references in documentation,
   templates, and configuration files that were not part of the current change set.

2. The bump-type heuristic **must** be computed before the AI call and **must** be
   used as the fallback if the AI is unavailable or returns an unrecognisable response.
   The AI call is a verification, not the primary source of truth.

3. The `package.json` `version` field **must** be updated by modifying only the
   `version` property in the parsed JSON, not by a raw string replacement on the entire
   file. Raw replacement risks corrupting version strings in dependency declarations
   that happen to match the current project version.

4. The consistency check (§3.8) is best-effort. A failure to run or a non-zero exit
   code from `check:version` **must not** cause the step to return `success: false`.
   Auto-fix results are informational; if auto-fix cannot resolve all issues, a warning
   is logged and the step continues.

5. JS version-config file detection (§3.6) **must** require both a `VERSION` export
   and a `BUILD_DATE` export to be present. Files with only one of the two constants
   **must not** be treated as version-config files and must go through the standard
   replacement path instead.

6. Service-worker file detection (§3.7) requires the presence of a `CACHE_NAME`
   constant. The CACHE_NAME update **must** refresh the date suffix to today's date;
   preserving the old date suffix would defeat the cache-busting purpose.

7. AI verification failures **must not** cause the step to fail. The heuristic bump
   type is always available as a fallback. The step **must** log a warning when AI
   verification fails, but **must not** re-throw.

8. The step **must** catch all exceptions from `execute` and return
   `{ success: false, error: <message> }`. Step 16 conforms to this requirement.
   Before returning the error result, the step writes a step-issues entry to the
   backlog describing the error.

9. In dry-run mode, **no** file writes and **no** script executions may occur. The
   step must log a preview of the operations it would perform and return immediately.
   The `check:version` script must not be invoked in dry-run mode.
