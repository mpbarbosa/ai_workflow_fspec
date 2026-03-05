# Step 02.5: Documentation Optimization — Functional Specification

**Step identifier:** `step_02_5`
**Step name:** Documentation Optimization
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))
**AI-integrated:** Optionally — via `aiAnalyzer` sub-component (see §4)
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md)

---

## 1. Purpose

Step 02.5 identifies and removes structural redundancy from a project's documentation set.
It analyses Markdown files in the designated documentation directory and detects:

- **Exact duplicates** — files with identical content.
- **Redundant pairs** — files with content similarity above a configurable threshold.
- **Outdated files** — files identified as stale by git history or version reference
  analysis.

When optimisations are needed, the step consolidates exact duplicates and archives
outdated files, then generates a structured report. When the documentation set is absent
or too small, the step skips without performing any analysis.

---

## 2. Step Kind Conformance

Step 02.5 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot) → Promise<StepResult>` |
| Can be skipped | **Yes** — skips when the documentation directory is missing or contains fewer than `minFiles` Markdown files |
| AI-integrated | **Optionally** — Phase 4 (AI edge case analysis) runs only when an `aiAnalyzer` sub-component is injected |

**Logging note.** Step 02.5 stores `this.logger` as an instance field (accepting an
injected logger or falling back to the global singleton). The PROJECT step contract calls
for the global singleton to be used directly without storing it as an instance field. This
is an internal implementation detail that does not affect the observable behaviour of the
step or the format of its output.

**Error handling.** Fatal errors from Phase 1 (heuristics) or Phase 6 (optimisation
execution) are caught at the top level and returned as `{ success: false, error, errors,
executionTime }`. Errors from Phases 2 (git history) and 3 (version analysis) are
non-fatal; they are logged as warnings and execution continues.

**AI note.** The optional AI component in this step is `aiAnalyzer` — a higher-level
sub-component that encapsulates its own AI interaction. Step 02.5 does **not** use the
standard `aiHelper` / `aiCache` pair and is **not** subject to the
[AI Prompt Contract](./ai_prompt_contract.md). The prompt contract governs steps that
call the Copilot API directly via those two services.

---

## 3. Domain Concepts

### 3.1 Documentation Directory

The directory scanned for Markdown files. Defaults to `docs/` relative to `projectRoot`.
Only files with the `.md` extension are processed.

### 3.2 Exact Duplicates

Two or more Markdown files whose content is byte-for-byte identical. When optimisations
are applied, all but one copy are moved to the archive directory. Consolidation is
performed by the `ConsolidationManager` sub-component.

### 3.3 Redundant Pairs

Pairs of files whose content similarity score meets or exceeds the `similarityThreshold`
(default 0.8). Similarity analysis is performed by the `HeuristicsAnalyzer`
sub-component. Redundant pairs are candidates for manual review or AI-assisted
deduplication; they are not automatically archived.

### 3.4 Outdated Files

Files identified as stale through either of two independent signals (results are merged
as a union):

- **Git history signal**: the file's last-modified commit is older than
  `outdatedThresholdMonths` (default 12). Determined by the `GitAnalyzer` sub-component.
- **Version reference signal**: the file contains version strings (in `X.Y.Z` format)
  that do not match the project's declared version. Determined by the `VersionAnalyzer`
  sub-component.

### 3.5 Archive Directory

A directory where consolidated and outdated files are moved when optimisations are applied.
Defaults to `.ai_workflow/archive/docs` relative to `projectRoot`. A timestamp-named
subdirectory is created per optimisation run, producing a unique path per execution.

### 3.6 Confidence Thresholds

Two thresholds control optimisation decisions:

| Threshold | Default | Purpose |
|---|---|---|
| `confidenceAuto` | 0.9 | Minimum similarity score for automatic consolidation |
| `confidenceAi` | 0.5 | Minimum similarity score floor for AI edge-case analysis |

### 3.7 Excluded Files

The following file name patterns are excluded from analysis by default:
`CHANGELOG.md`, `LICENSE*`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`. Callers may extend
this list via configuration.

### 3.8 Dry-Run Mode

When `dryRun` is `true` (default: `false`), Phase 6 logs its planned actions but does
not move or delete any files.

---

## 4. Constructor Dependencies

Step 02.5 accepts all collaborators via a single options map at construction time.

| Dependency key | Role | Behaviour when absent |
|---|---|---|
| `logger` | Logger instance for diagnostic output | Falls back to the global logger singleton |
| `fileOps` | File system operations (read, write, list, glob, stat) | Defaults to a new `FileOperations` instance |
| `gitOps` / `gitAutomation` | Git history queries for last-modified dates (passed to `GitAnalyzer`) | When absent, `GitAnalyzer` may degrade gracefully |
| `heuristics` | Content similarity analysis and duplicate detection | Defaults to a new `HeuristicsAnalyzer` instance |
| `gitAnalyzer` | Git-history-based staleness analysis | Defaults to a new `GitAnalyzer` instance |
| `versionAnalyzer` | Version-reference-based staleness analysis | Defaults to a new `VersionAnalyzer` instance |
| `consolidation` | Duplicate consolidation and file archiving | Defaults to a new `ConsolidationManager` instance |
| `reporting` | Report generation and writing | Defaults to a new `ReportingManager` instance |
| `aiAnalyzer` | AI-powered edge-case analysis for redundant pairs (opt-in) | `null`; Phase 4 is skipped entirely when absent |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the project root |

Additional configuration (thresholds, flags, directory paths) is supplied through the
`userConfig` parameter of the internal orchestration method, but the orchestrator-facing
`execute(projectRoot)` interface derives all configuration from defaults combined with
`projectRoot`.

### 5.2 Configuration Validation

Before executing, user-supplied configuration is merged with defaults and validated. If
validation fails (e.g. `docsDir` is empty, `similarityThreshold` is outside `[0, 1]`, or
`outdatedThresholdMonths` is negative), the step returns immediately:

```
{ success: false, errors: ["<validation error message>", …] }
```

### 5.3 Skip Conditions

After successful configuration validation, two skip conditions are evaluated:

| Reason | Condition |
|---|---|
| Documentation directory missing | The resolved documentation directory does not exist on disk |
| Documentation set too small | Fewer than `minFiles` (default 5) Markdown files exist in the directory |

Skip results return `{ success: true, skipped: true, reason: "<message>" }`.

### 5.4 Execution Phases

When not skipped, Step 02.5 runs seven sequential phases:

#### Phase 1 — Heuristics Analysis

Scan all discovered Markdown files using the `HeuristicsAnalyzer`. Produce:

- `exactDuplicates` — file paths with identical content.
- `redundantPairs` — file path pairs with similarity ≥ `similarityThreshold`.

A failure in Phase 1 propagates to the top-level error handler and terminates the step.

#### Phase 2 — Git History Analysis

Pass the file list to the `GitAnalyzer` using `outdatedThresholdMonths`. Produce a set
of `outdatedFiles` identified by git commit age. A failure in Phase 2 is non-fatal: a
warning is logged and execution continues with an empty git-history result.

#### Phase 3 — Version Reference Analysis

Detect the project's current version from the project root. Pass the file list to the
`VersionAnalyzer` to identify files containing stale version strings. Merge these
results with the git-history outdated set (union, de-duplicated). A failure in Phase 3
is non-fatal: a warning is logged and execution continues.

#### Phase 4 — AI Edge Case Analysis (Optional)

Runs only when `aiAnalyzer` is injected **and** `redundantPairs` is non-empty.

For each redundant pair, the `aiAnalyzer` evaluates whether the pair is truly redundant
or represents intentional divergence. Results are used to update each pair's similarity
score. Pairs processed by AI are marked `aiAnalyzed: true`. A failure in Phase 4 is
non-fatal.

#### Phase 5 — Summary Display

Aggregate all collected results and emit a log summary: total files analysed, exact
duplicate count, redundant pair count, outdated file count, and total files that would be
optimised (`exactDuplicates.length + outdatedFiles.length`).

#### Phase 6 — Optimisation Execution (Conditional)

Runs only when `filesOptimized > 0` (at least one exact duplicate or outdated file).

- **Consolidate exact duplicates:** group by content hash and move all but one copy to
  the archive directory.
- **Archive outdated files:** move to the archive directory.

When `dryRun` is `true`, these operations are logged but not applied to the file system.
A failure in Phase 6 propagates to the top-level error handler and terminates the step.

#### Phase 7 — Report Generation (Conditional)

Runs only when Phase 6 executed. Generates a Markdown optimisation report and writes it
to `<archiveDir>/<timestamp>/optimization_report.md`.

### 5.5 Result Assembly

After all phases complete, the step returns:

| Field | Type | Description |
|---|---|---|
| `success` | `true` | All phases completed (or were skipped) without fatal error |
| `summary` | Object | Aggregated results (§6.2) |
| `optimizationResults` | Object or `null` | Consolidation and archive results; `null` when no files were optimised |
| `reportResult` | Object or `null` | Report generation result; `null` when no files were optimised |
| `executionTime` | Integer | Total wall-clock time in seconds |
| `errors` | Object[] | Non-fatal phase errors collected during the run |

---

## 6. Data Shapes

### 6.1 Configuration Object

| Field | Type | Default | Description |
|---|---|---|---|
| `docsDir` | String | `"docs"` | Documentation directory (relative to `projectRoot`) |
| `archiveDir` | String | `".ai_workflow/archive/docs"` | Archive directory (relative to `projectRoot`) |
| `outdatedThresholdMonths` | Integer | `12` | Age in months after which a file is considered outdated by the git signal |
| `similarityThreshold` | Number | `0.8` | Content similarity score at or above which files are considered redundant |
| `confidenceAuto` | Number | `0.9` | Similarity score for automatic action without confirmation |
| `confidenceAi` | Number | `0.5` | Similarity score floor for AI edge-case analysis |
| `dryRun` | Boolean | `false` | When `true`, no files are moved or deleted |
| `minFiles` | Integer | `5` | Minimum Markdown file count required to proceed |
| `excludePatterns` | String[] | (see §3.7) | File name patterns excluded from analysis |

### 6.2 Aggregated Results

| Field | Type | Description |
|---|---|---|
| `totalFiles` | Integer | Total Markdown files discovered in the documentation directory |
| `exactDuplicates` | String[] | Paths of exact duplicate files |
| `redundantPairs` | Object[] | Pairs of similar files with similarity score |
| `outdatedFiles` | String[] | Paths of files identified as outdated (union of both signals) |
| `edgeCases` | Object[] | AI edge-case analysis results; empty when Phase 4 was skipped |
| `filesOptimized` | Integer | `exactDuplicates.length + outdatedFiles.length` |
| `errors` | Object[] | Non-fatal phase errors, each with `{ phase, error }` |

### 6.3 StepResult (success)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | No fatal error occurred |
| `summary` | Object | Aggregated results (§6.2) |
| `optimizationResults` | Object or `null` | Phase 6 output; `null` when no files were optimised |
| `reportResult` | Object or `null` | Phase 7 output; `null` when no files were optimised |
| `executionTime` | Integer | Total elapsed time in seconds |
| `errors` | Object[] | Non-fatal phase errors |

### 6.4 StepResult (failure)

| Field | Type | Description |
|---|---|---|
| `success` | `false` | A fatal error occurred |
| `error` | String | Error message from the failing phase |
| `errors` | Object[] | Non-fatal errors collected before the fatal failure |
| `executionTime` | Integer | Time elapsed before the failure in seconds |

### 6.5 StepResult (skipped)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `reason` | String | Human-readable description of the skip condition |

---

## 7. Constraints and Rules

1. The documentation directory **must** be resolved to an absolute path (by joining
   `projectRoot` with `docsDir`) before any file system operation. The step must not
   operate on relative paths directly.

2. Files matching `excludePatterns` **must** be excluded from all analysis phases. These
   files are infrastructure documents that do not benefit from duplication detection.

3. The `minFiles` guard **must** be enforced before Phase 1 begins. Running similarity
   analysis on one or two files produces meaningless results and wastes time.

4. Failures in Phase 2 (git history) and Phase 3 (version analysis) **must** be treated
   as non-fatal. An unavailable git binary or missing project manifest should not prevent
   heuristics results from being used.

5. When `dryRun` is `true`, Phase 6 **must not** perform any destructive file operations
   (no moves, no deletions). A dry-run execution that modifies files on disk is a contract
   violation.

6. The `aiAnalyzer` is an **optional, higher-level** dependency. It is not the same as
   the `aiHelper` / `aiCache` pair used by standard AI-integrated steps. This step is
   **not subject** to the [AI Prompt Contract](./ai_prompt_contract.md). The AI
   interaction is fully encapsulated within the `aiAnalyzer` component.

7. The archive directory **must** be resolved to an absolute path before Phase 6 executes.
   A relative archive path causes file-move operations to resolve against the process
   working directory rather than the project root, producing incorrect archive locations.

8. Non-fatal phase errors **must** be accumulated in the `errors` array and included in
   the final result. Swallowing non-fatal errors silently makes diagnosis of partial
   failures impossible.
