# Step 0f: Commit Artifacts — Functional Specification

**Step identifier:** `step_0f`  
**Step name:** Commit Artifacts  
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))  
**Version:** 2.0.0  
**Status:** Draft  
**Related:** [`step_contract.md`](./step_contract.md)

---

## 1. Purpose

Step 0f commits any uncommitted workflow artifact files to the repository at the
conclusion of a workflow run. It ensures that logs, summaries, metrics, and other
outputs written to the workflow artifacts directory during the run are persisted to
version control before the orchestrator exits.

Step 0f is the designated artifact commit step. Other steps write artifacts freely
to the workflow artifacts directory; Step 0f is the one responsible for committing
them.

---

## 2. Step Kind Conformance

Step 0f conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot) → Promise<StepResult>` |
| Can be skipped | **Yes** — skips gracefully when no artifact files are found |
| AI-integrated | **No** — does not call the AI API |
| Estimated duration | ≤ 15 seconds under normal conditions |

Step 0f conforms fully to the step contract error-handling rule: all exceptions are
caught inside `execute` and surfaced as `{ success: false, error: … }`. No exception
propagates to the caller.

---

## 3. Domain Concepts

### 3.1 Workflow Artifact File

A **workflow artifact file** is any file whose path is recognised as belonging to the
workflow artifacts directory (e.g. `.ai_workflow/`). The recognition check is delegated
to the `validateArtifactPath` utility, which encapsulates the matching rule. Step 0f
uses this check to separate artifact files from other changed files in the working tree.

Non-artifact files found in the working tree are **never** included in the commit.
Step 0f has a narrow, well-defined scope: it touches only workflow artifacts.

### 3.2 Git Working-Tree Status Categories

When the git operations service returns the working-tree status, it reports files in
three categories:

| Category | Description |
|---|---|
| Staged | Files already added to the index, ready to be committed |
| Unstaged | Files modified in the working tree but not yet staged |
| Untracked | New files not yet tracked by git |

Step 0f collects files from all three categories and considers the combined list when
identifying artifact files to commit.

### 3.3 AutoCommit

The **AutoCommit** collaborator encapsulates the full commit sequence: staging eligible
files, building a conventional commit message, and executing the commit. It also
enforces its own filtering rules — for example, it may reject files that fail a
secondary validation or skip the commit when the auto-commit feature is disabled at
the configuration level.

When `dryRun` mode is active, AutoCommit stages and validates files but does not
produce an actual git commit. The result reports what would have been committed.

---

## 4. Constructor Dependencies

Step 0f receives all collaborators at construction time via a single options map.

| Dependency key | Role | Behaviour when absent |
|---|---|---|
| `gitOps` | Queries working-tree status to discover changed files | A default git operations instance is created internally |
| `autoCommit` | Stages and commits the identified artifact files | A default AutoCommit instance is created using `gitOps` and `dryRun` |
| `dryRun` | Boolean flag; when `true`, the commit is simulated only | Defaults to `false` (real commits are made) |

When neither `gitOps` nor `autoCommit` is supplied, Step 0f constructs its own
instances. The internally constructed AutoCommit is always initialised with
`enabled: true`.

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the root of the project under workflow |

### 5.2 Execution Phases

Step 0f executes the following operations in order:

#### Phase 1 — Query Working-Tree Status

Request the current git status of the project from the git operations service. The
status yields three lists: staged files, unstaged files, and untracked files. Collect
the file paths from all three lists into a single combined list.

#### Phase 2 — Identify Artifact Files

Apply the artifact-path validation rule (§3.1) to every file in the combined list.
If no file in the combined list is a workflow artifact file, Step 0f immediately
returns a skip result:

```
{ success: true, skipped: true, reason: 'no_artifact_files' }
```

No further phases are executed after a skip.

#### Phase 3 — Commit via AutoCommit

Pass the filtered list of artifact file paths to the AutoCommit collaborator. AutoCommit
handles staging, commit-message generation, and the git commit operation. The result
includes:

- Whether a commit was actually made (`committed` flag).
- The list of files that were committed.
- When no commit was made, a reason code (e.g. `no_files`, `filtered`, `disabled`,
  `no_git`).

#### Phase 4 — Build Summary Message

Derive a human-readable summary string from the AutoCommit result:

| Condition | Summary text |
|---|---|
| `committed` is `true` | `"Committed N artifact file(s)"` where N is the file count |
| `committed` is `false`, reason = `no_files` | `"No artifact files to commit"` |
| `committed` is `false`, reason = `filtered` | `"No eligible artifact files after filtering"` |
| `committed` is `false`, reason = `disabled` | `"Auto-commit is disabled"` |
| `committed` is `false`, reason = `no_git` | `"Git not available"` |
| `committed` is `false`, unknown reason | `"Skipped: {reason}"` |

Emit the summary to the workflow log.

#### Phase 5 — Return Result

Return a step result as defined in §6.2.

### 5.3 Skip Conditions

| Reason | Condition |
|---|---|
| `no_artifact_files` | The combined git status list contains no workflow artifact files |

A skip result always carries `success: true`.

### 5.4 Error Handling

All exceptions raised during Phases 1–5 are caught. On any unhandled error, Step 0f
returns:

```
{ success: false, error: <exception message> }
```

---

## 6. Data Shapes

### 6.1 StepResult (committed or skipped)

| Field | Type | Present when | Description |
|---|---|---|---|
| `success` | Boolean | Always | `true` when Step 0f completed without error |
| `skipped` | `true` | Skip only | Present and `true` when no artifact files were found |
| `reason` | String | Skip only | `'no_artifact_files'` |
| `committed` | Boolean | Non-skip success | `true` when AutoCommit produced a git commit |
| `files` | String[] | Non-skip success | Paths of files committed; empty array when nothing was committed |
| `summary` | String | Non-skip success | Human-readable outcome message (§5.2 Phase 4) |
| `error` | String | Failure | Exception message when `success` is `false` |

### 6.2 AutoCommit Result (internal)

The value returned by AutoCommit and used to derive the step result:

| Field | Type | Description |
|---|---|---|
| `committed` | Boolean | `true` when a commit was made |
| `files` | String[] or absent | Files included in the commit |
| `reason` | String or absent | Reason code when `committed` is `false` |

---

## 7. Step Metadata

Step 0f exposes a metadata record used by the orchestrator for scheduling and reporting.

| Field | Value | Description |
|---|---|---|
| `id` | `'0f'` | Step sequence identifier |
| `name` | `'Commit Artifacts'` | Human-readable step name |
| `canSkip` | `true` | Skips when no artifact files are present |
| `dependencies` | `[]` | No preceding steps required by the step itself |

---

## 8. Constraints and Rules

1. Step 0f **must** only commit files whose paths are confirmed as workflow artifact
   files by the `validateArtifactPath` rule. It must never stage or commit non-artifact
   files, regardless of their presence in the working tree.

2. When `dryRun` is `true`, Step 0f **must not** produce a real git commit. The result
   must accurately reflect what would have been committed so that callers can inspect it
   without side effects.

3. Step 0f does **not** call the AI API. It is not subject to the
   [AI Prompt Contract](./ai_prompt_contract.md).

4. A git status query failure **must** be caught and returned as
   `{ success: false, error: … }`. It must not propagate as an unhandled exception.

5. An AutoCommit result where `committed` is `false` is **not** a failure. A non-commit
   outcome (e.g. no files eligible, feature disabled) is a normal operating condition
   and must be returned with `success: true`. Only unhandled exceptions produce
   `success: false`.

6. The step **must** log the summary message (§5.2 Phase 4) at info level regardless of
   whether a commit was made. Silent completion is not permitted.
