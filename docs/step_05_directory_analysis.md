# Step 05: Directory Structure Analysis — Functional Specification

**Step identifier:** `step_05`
**Step name:** Directory Structure Validation
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 05 validates the directory structure of the project and organises misplaced
documentation. It combines deterministic structural checks with two complementary AI
analyses: a directory-architecture review and a requirements-engineering perspective.

Step 05 answers three questions:

1. **Are documentation files in the right place?** — by detecting Markdown files in
   the project root that should be under `docs/`.
2. **Is the directory structure consistent with documentation?** — by comparing
   existing directories against declared critical directories and checking whether
   each directory is mentioned in the project's README or Copilot instructions.
3. **What structural improvements does an architecture expert recommend?** — via two
   sequential AI analyses using the `architecture_reviewer` persona.

---

## 2. Step Kind Conformance

Step 05 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **No** — Step 05 always executes when invoked |
| AI-integrated | **Yes** — two AI calls via `aiHelper` and `aiCache` |
| Conforms to AI Prompt Contract | **Yes** — directory tree and issue list are injected into the prompt (§5 Phase 5) |

**Logging convention.** Step 05 uses the **global logger singleton** imported at module
level, consistent with the PROJECT step logging contract.

**Error handling deviation.** Step 05 re-throws unhandled exceptions from `execute`
rather than returning `{ success: false, error }`. This deviates from the step contract
rule (§5 rule 4) that forbids propagating exceptions. Callers must be prepared to catch.

---

## 3. Domain Concepts

### 3.1 Documentation Categories

Each Markdown file found in the project root is assigned exactly one documentation
category based on the filename. Pattern matching is case-insensitive and applied in the
order listed; the first matching pattern wins.

| Category | Identifier | Target Directory | Name Pattern |
|---|---|---|---|
| Workflow | `workflow` | `docs/workflow-automation` | WORKFLOW, EXECUTION, ORCHESTRAT, PIPELINE, AUTOMATION |
| Test | `test` | `docs/testing` | TEST, COVERAGE, SPEC |
| Bug Fix | `bugfix` | `docs/reports/bugfixes` | BUGFIX, FIX, ISSUE, BUG |
| Implementation | `implementation` | `docs/reports/implementation` | IMPLEMENTATION, COMPLETE, SUMMARY, MIGRATION, REFACTOR |
| Analysis | `analysis` | `docs/reports/analysis` | ANALYSIS, REPORT, VALIDATION, REVIEW, AUDIT |
| Guide | `guide` | `docs/guides` | GUIDE, HOWTO, TUTORIAL, QUICK, REFERENCE |
| Architecture | `architecture` | `docs/architecture` | ARCHITECTURE, DESIGN, PATTERN, STRUCTURE |
| Report | `report` | `docs/reports` | DELIVERABLE, CHECKLIST, PHASE, RECOMMENDATION |
| Uncategorized | `uncategorized` | `docs/misc` | *(no pattern matched)* |

### 3.2 Root-Allowed Files

The following files are permitted in the project root and must not be treated as
misplaced:

`README.md`, `CHANGELOG.md`, `LICENSE.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`

Any other Markdown file found directly in the project root is considered misplaced.

### 3.3 Excluded Directories

The following directory names are excluded from all structural scans:

`node_modules`, `.git`, `coverage`, `dist`, `build`, `.vscode`, `.idea`, `venv`,
`.venv`, `__pycache__`, `.pytest_cache`, `.jest-cache`, `.ai_workflow`, `.ai_cache`,
`.nyc_output`, `.next`, `.nuxt`, `.svelte-kit`, `out`

A directory is excluded if any path segment matches an excluded name.

### 3.4 Issue Types

Structural validation produces issues of the following types:

| Identifier | Meaning |
|---|---|
| `missing_critical` | A directory declared as critical is not present in the filesystem |
| `undocumented` | An existing directory is not mentioned in any documentation file |
| `doc_mismatch` | A directory is documented but does not exist on the filesystem |

### 3.5 Critical Directories

Critical directories are the set of directories that must be present. They are resolved
in priority order:

1. **Configuration-sourced** (preferred): read from the project configuration file's
   `structure` field, combining `src_dirs`, `test_dirs`, and `docs_dirs`. Duplicates
   are removed.
2. **Default set** (fallback when configuration is absent or unreadable): always
   includes `.github`, plus any of `src`, `docs`, `lib`, `bin`, `tests`, `test` that
   already exist on disk.

### 3.6 Documentation Reference Check

A directory is considered documented when its base name appears as a substring in at
least one of the project's documentation files. The documentation files checked are
`README.md` and `.github/copilot-instructions.md`. Only files that can be read are
checked; unreadable files are silently skipped.

---

## 4. Constructor Dependencies

Step 05 receives all collaborators at construction time via a single options map.
All dependencies have fallback defaults.

| Dependency key | Role | When absent |
|---|---|---|
| `fileOps` | File system operations: list directory, read file | Falls back to a new `FileOperations` instance |
| `backlog` | Writes step summary to the workflow artifact store | Falls back to a new `Backlog` instance |
| `gitOps` | Git operations service (available but not used in the current execute path) | Falls back to a new `GitAutomation` instance |
| `config` | Reads project configuration to determine critical directories | Falls back to a new `Config` instance |
| `aiHelper` | Executes AI API requests (see AI Prompt Contract §6.1) | Falls back to a new `AiHelper` instance |
| `aiCache` | Caches AI responses by key (see AI Prompt Contract §6.2) | Falls back to a new `AiCache` instance |
| `techStack` | Detects the project's primary programming language | Falls back to a new `TechStackDetector` instance |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Source | Description |
|---|---|---|
| `projectRoot` | Orchestrator | Absolute path to the project root directory |
| `options` | Orchestrator | Optional hints. Recognised keys: `language`, `projectDescription`, `scope`, `modifiedCount`, `primaryLanguage`, `projectKind` |

### 5.2 Execution Phases

Step 05 runs the following phases in order:

#### Phase 1 — Organise Misplaced Documentation

1. List all Markdown (`.md`) files directly in the project root.
2. Filter out files that are in the root-allowed list (§3.2).
3. For each remaining (misplaced) file:
   - Determine its category by matching the filename against the category patterns (§3.1).
   - Record the computed target directory.
4. Return the count of misplaced files and the count of files actually moved.

**Implementation note:** The current implementation computes target directories but does
not physically move files. The `organizedDocs` count is always zero. The categorisation
result is available in the returned record for informational purposes only.

#### Phase 2 — Validate Directory Structure

1. Recursively enumerate all directories under `projectRoot`, producing relative paths.
2. Filter out excluded directories (§3.3).
3. Resolve critical directories (§3.5).
4. Read documentation files for the reference check (§3.6).
5. Apply all three validation checks:
   - **Missing critical**: identify which critical directories are absent from the
     filesystem.
   - **Undocumented**: identify which existing (non-excluded) directories are not
     mentioned in any documentation file.
   - **Doc mismatch**: identify which critical directories are mentioned in documentation
     but do not exist on disk.

If the configuration cannot be loaded, fall back to the default critical directory set.
If the structure validation itself fails, return an empty issue list with zero counts
rather than propagating the error.

#### Phase 3 — Count Directories

Count the total number of non-excluded directories under `projectRoot`. This is the
full recursive count after applying exclusion rules, used for reporting and AI prompt
context.

#### Phase 4 — Generate Initial Report and Save to Backlog

Assemble the results from Phases 1–3 into a structured Markdown report containing:
- A summary table (total directories, misplaced docs, organised files, structure
  issues).
- A critical issues section when any critical directories are missing.
- An issues detail section (grouped by type, capped at 10 entries per type).
- An organised documentation section when files were moved.
- A passing confirmation when there are no issues and no misplaced docs.

Write this initial report to the backlog summary entry for step `5`,
section `'Directory Structure Validation'`.

#### Phase 5 — AI Directory Structure Analysis

Initialise `aiHelper`. If the AI is unavailable, log a warning and skip Phases 5 and 6.

When AI is available:

1. Initialise `aiCache` (call `init()` before any `withCache` call).
2. Detect the primary programming language using `techStack`.
3. Build the prompt by attempting to load the `step4_directory_prompt` YAML template.
   Populate it with:
   - `project_name` — the project root path
   - `primary_language` — detected or provided language
   - `dir_count` — total directory count
   - `change_scope` — from options, if provided
   - `modified_count` — from options, if provided
   - `missing_critical`, `undocumented_dirs`, `doc_structure_mismatch` — validation counts
   - `structure_issues_content` — up to 20 issue lines formatted as
     `- [{type}] {directory}: {message}`
   - `dir_tree` — up to 50 existing directory paths, one per line
4. If the YAML template cannot be loaded, fall back to a built-in structured prompt
   with the same data inlined.
5. Invoke the AI with cache key `step_05|{totalDirs}|{issueCount}`, using the
   `architecture_reviewer` persona.

#### Phase 6 — AI Requirements Engineering Analysis (Supplementary)

As a second, supplementary AI call:

1. Count how many requirements-related documents (files whose name matches patterns
   such as `requirements`, `FRS`, `BRD`, `SRS`, `func.spec`, `user.stor`) exist under
   `docs/`. Unreadable directories are silently skipped.
2. Build the requirements engineering prompt by loading the `requirements_engineer_prompt`
   YAML template. Populate it with:
   - `project_name`, `project_description`, `primary_language`
   - `source_files` — comma-separated list of existing directories
   - `requirements_docs_count` — count from step above
   - `stakeholder_count` — hardcoded to `'1'`
3. If the template cannot be loaded, skip this supplementary analysis silently.
4. Cache key: `step_05_req|{projectRoot}|{totalDirs}`. Persona: `architecture_reviewer`.

#### Phase 7 — Save Enriched Report to Backlog

When either or both AI calls produced non-empty content, overwrite the backlog summary
entry (step `5`, section `'Directory Structure Validation'`) with the initial report
extended by:
- An `## AI Recommendations` section containing the directory analysis result.
- A `## Requirements Engineering Analysis` section containing the supplementary result
  (present only when the supplementary analysis returned content).

If neither AI call produced content, the initial report written in Phase 4 remains
unchanged.

### 5.3 Skip Conditions

Step 05 has no skip conditions. It always executes all phases when invoked.

### 5.4 Error Handling

The outer `execute` body is wrapped in a try/catch. If any unhandled exception occurs,
the error is logged and **re-thrown**. This deviates from the step contract; callers
must catch.

Individual sub-operations (structure validation, AI calls) have their own internal
error handling and degrade gracefully rather than propagating.

---

## 6. Data Shapes

### 6.1 StepResult (success)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `totalDirs` | Integer | Count of non-excluded directories |
| `misplacedDocs` | Integer | Count of misplaced Markdown files found in project root |
| `organizedDocs` | Integer | Count of files actually moved (currently always 0) |
| `issues` | Object[] | Structural issues (§6.2) |
| `missingCritical` | Integer | Count of missing critical directories |
| `undocumented` | Integer | Count of undocumented directories |
| `docMismatch` | Integer | Count of documented-but-missing directories |
| `existingDirs` | String[] | Relative paths of all non-excluded directories |

### 6.2 Structural Issue Record

| Field | Type | Description |
|---|---|---|
| `type` | String | Issue type identifier (§3.4) |
| `directory` | String | Relative path of the affected directory |
| `message` | String | Human-readable description |

### 6.3 Misplaced Doc Record

| Field | Type | Description |
|---|---|---|
| `file` | String | Relative path of the misplaced file |
| `category` | String | Assigned documentation category (§3.1) |
| `targetDir` | String | Computed target directory path (§3.1) |

---

## 7. Constraints and Rules

1. **Error handling deviation.** Step 05 re-throws unhandled exceptions rather than
   returning `{ success: false, error }`. This violates step contract §5 rule 4.
   Callers must wrap `execute` in their own error handler.

2. Root-allowed files (§3.2) **must** be excluded from misplaced-doc detection. Any
   file in the root-allowed list must never be reported as misplaced or moved.

3. Excluded directories (§3.3) **must** be excluded from structural validation. Issues
   must not reference paths inside excluded directories.

4. Critical directory resolution **must** prefer the configuration-sourced list. The
   default set is only used when the configuration file cannot be loaded.

5. `aiCache.init()` **must** be called before any `withCache` call, as required by the
   AI Prompt Contract §6.2.

6. The two AI calls are sequential. The supplementary requirements analysis (Phase 6)
   must not begin until the primary directory analysis (Phase 5) has completed.

7. Both AI calls use the same `architecture_reviewer` persona. They must not share a
   cache key. The primary key encodes directory and issue counts; the supplementary key
   encodes the project root path and directory count.

8. AI calls are optional enhancements. A failure in either Phase 5 or Phase 6 **must
   not** prevent the initial report (Phase 4) from being written, nor cause `execute`
   to return `success: false`. Individual AI errors are caught and ignored silently.

9. The directory tree injected into the AI prompt (Phase 5, `dir_tree`) is capped at
   50 entries. The issue list is capped at 20 entries. These caps prevent the prompt
   from exceeding model context limits.

10. **File content injection note.** Step 05 does not inject the contents of source or
    documentation files. Instead, it injects the directory tree (path list) and issue
    summary. This is appropriate because the analysis subject is the project's structure,
    not specific file contents. The AI Prompt Contract's file content injection
    requirement (§3) applies when specific files are referenced; path lists are
    structural metadata, not file contents.
