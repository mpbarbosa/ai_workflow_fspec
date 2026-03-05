# Step 12: Git Finalization — Functional Specification

**Step identifier:** `step_12`  
**Step name:** Git Finalization  
**Step kind:** `CONTEXT` (see [Step Contract §2.2](./step_contract.md))  
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))  
**Version:** 2.0.0  
**Status:** Draft  
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 12 is the **git finalization step** of the workflow. It runs near the end of the
pipeline, after analysis and quality steps have completed, and is responsible for
committing all workflow-produced changes to the repository and pushing them to the
remote origin.

Step 12 answers four questions:

1. **What is the current state of the repository?** — which files have changed, what
   type of commit is appropriate, and whether any submodules are present.
2. **Is a new commit required?** — if nothing has changed, the step still handles any
   commits that are already ahead of the remote.
3. **What should the commit message say?** — derived first by heuristic, then refined
   by the AI `git_specialist` persona when the AI is available.
4. **Has the new version been tagged and pushed?** — after committing, the step creates
   a semver git tag from the project's version metadata and pushes it to the remote.

---

## 2. Step Kind Conformance

Step 12 conforms to the **CONTEXT** step kind as defined in `step_contract.md §2.2`.

| Property | Value |
|---|---|
| Kind identifier | `"context"` |
| Execute signature | `execute(context) → Promise<StepResult>` |
| Can be skipped | **No** — the step always runs, though it may short-circuit when there is nothing to commit or when dry-run mode is enabled |
| AI-integrated | **Yes** — calls the AI API via `aiHelper` and `aiCache` |
| Conforms to AI Prompt Contract | **Yes** — prompt is constructed from a YAML template or a structured fallback; no prohibited techniques are used |

**Error handling deviation.** Step 12 does **not** conform to the step contract rule
requiring that `execute` never throw. When an unhandled error occurs, the `catch`
block logs the error and **re-throws** it rather than returning
`{ success: false, error: … }`. Callers must be prepared to catch exceptions from
this step.

This deviation is documented in §7 Constraints and Rules.

---

## 3. Domain Concepts

### 3.1 Change Categories

Every changed file is classified into exactly one category by file extension. Categories
are matched in order; the first matching rule wins.

| Category | Matching Rule | Weight |
|---|---|---|
| `documentation` | Extension is `.md`, `.txt`, `.rst`, or `.adoc` | 1 |
| `tests` | Extension matches `*.test.*` or `*.spec.*` for common source extensions | 3 |
| `scripts` | Extension is `.sh`, `.bash`, `.zsh`, `.ps1`, `.cmd`, or `.bat` | 2 |
| `code` | Extension is `.js`, `.ts`, `.py`, `.go`, `.java`, `.rb`, `.php`, `.c`, `.cpp`, `.rs`, `.swift`, or `.kt` | 5 |
| `config` | Extension is `.json`, `.yaml`, `.yml`, `.toml`, `.ini`, `.xml`, `.conf`, or `.config` | 1 |
| `other` | Any file not matching the above rules | 1 |

### 3.2 Commit Type Inference

The commit type and scope are inferred from the category counts of changed files.
Inference is evaluated in priority order; the first matching rule wins.

| Commit Type | Scope | Condition |
|---|---|---|
| `feat` | `implementation+tests` | Code count > 0 **and** test count > 0 |
| `feat` | `implementation` | Code count > 0 (no tests) |
| `test` | `testing` | Test count > 0, code count = 0 |
| `docs` | `documentation` | Documentation count > 0, code and test = 0 |
| `chore` | `automation` | Script count > 0 |
| `chore` | `configuration` | Config count > 0 |
| `chore` | `general` | No other category matches |

### 3.3 Impact Score

A weighted score summarising the magnitude of the change set. Computed by multiplying
each category's file count by its weight (§3.1) and summing the products.

### 3.4 Submodule States

Submodules are categorised into three states:

| State | Description |
|---|---|
| `initialized` | Submodule is present and at the expected commit |
| `uninitialized` | Submodule path exists but has not been initialised (`-` prefix in status output) |
| `conflict` | Submodule has merge conflicts (`U` prefix in status output) |

### 3.5 Dry-Run Mode

When constructed with `dryRun: true`, the step logs a preview of the git operations
it would perform but executes none of them. It returns `{ success: true, dryRun: true }`
immediately after the preview log.

### 3.6 Conventional Commit Format

Commit messages follow the Conventional Commits specification:

```
<type>(<scope>): <description>

<body>

<footer>
```

The footer, when present, includes the workflow automation version.

---

## 4. Constructor Dependencies

Step 12 receives all collaborators via a single options map at construction time. All
dependencies have fallback defaults.

| Dependency key | Role | When absent |
|---|---|---|
| `logger` | Logs step progress and errors | Falls back to `console` |
| `executor` | Executes git commands in a subprocess | Required for git operations; step throws at runtime if absent |
| `backlogManager` | Writes step summary and issues to the workflow artifact store | Backlog writes are silently skipped |
| `gitAutomation` | Optional pre-built git automation service | Not used directly by execute; available for subclasses |
| `aiHelper` | Executes AI API requests for commit message generation | A new `AiHelper` instance is created if absent |
| `aiCache` | Caches AI responses | A new `AiCache` instance is created if absent |
| `dryRun` | Boolean; when `true`, no git operations are performed | Defaults to `false` |
| `interactiveMode` | Boolean; reserved for interactive confirmation flows | Defaults to `false` |
| `aiEnabled` | Boolean; legacy flag for explicit AI enablement | Defaults to `false` |
| `projectRoot` | Fallback project root when not supplied via context | Falls back to the process working directory |
| `promptsDir` | Directory path for AI prompt templates | Passed to the `AiHelper` constructor |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Type | Description |
|---|---|---|
| `context` | WorkflowContext object | Full workflow context. `context.projectRoot` takes precedence over the constructor `projectRoot` option. |

Relevant context fields:

| Field | Description |
|---|---|
| `projectRoot` | Absolute path to the project under analysis |
| `results` | Array of prior step results. Used to check whether step 07 reported untested files. |

### 5.2 Dry-Run Short-Circuit

When `dryRun` is `true`, the step logs a human-readable preview of the git operations
it would perform and returns `{ success: true, dryRun: true, message: 'Dry run completed' }`
without executing any git commands.

If a `backlogManager` is present, a dry-run summary entry is written to the backlog.

### 5.3 Execution Phases

Step 12 runs the following phases in order:

#### Phase 1 — Analyse Git State

Query the repository for:

- **Current branch** name.
- **Commits ahead and behind** the upstream tracking branch. If there is no upstream
  branch, both values default to zero and the failure is silently ignored.
- **Working-tree status** — all modified, staged, untracked, and deleted files.
- **Change categories** — classify all changed files using the rules in §3.1.
- **Inferred commit type and scope** — determined from the category counts (§3.2).
- **Submodule presence** — detected by reading `.gitmodules` configuration. Failures
  in this check (e.g. when no `.gitmodules` exists) are silently ignored.

#### Phase 1b — Test-Coverage Warning (Conditional)

If the prior step results contain a step-07 entry whose `untestedFiles` list is
non-empty, a warning is logged listing the count of untested files. This is a soft
warning only — execution continues regardless.

#### Phase 2 — Process Submodules (Conditional)

Runs only when Phase 1 detected submodules.

1. Retrieve all submodule details.
2. Log a human-readable submodule summary.
3. Warn if any submodule is in conflict state.
4. For each uninitialised submodule, run a recursive submodule update (init + update).

Submodule processing failures are caught and logged as warnings. The step continues
even if submodule processing fails.

#### Phase 3 — Check for Changes

If the total changed file count is zero, enter the **no-changes path**:

- If `commitsAhead > 0`: push existing commits to the remote and write a backlog summary
  reflecting the push result. Return `{ success: true, noChanges: true, branch, pushed }`.
- If `commitsAhead = 0`: write a backlog summary indicating the repository is up to date.
  Return `{ success: true, noChanges: true, branch }`.

If total changes > 0, continue to Phase 4.

#### Phase 4 — Stage Changes

Stage all changes with a broad staging command. Then force-stage the workflow artifact
directory (`.ai_workflow/`) explicitly, because parts of that directory may be listed
in `.gitignore` and would otherwise be silently excluded by the broad staging.

If submodules are present, first stage and commit changes within each submodule before
staging the parent repository. Submodule commit failures are caught and logged as
warnings; they do not prevent the parent staging from proceeding.

#### Phase 5 — Generate Commit Message

1. Read project metadata (name, description, version) from `package.json`. Falls back
   to empty strings when the file is absent or unreadable.
2. Build a **heuristic commit message** using the Conventional Commits format (§3.6),
   populated with the inferred type, scope, category counts, and project version.
3. Attempt to **refine the message with AI**:
   a. Initialise the AI helper. If initialisation fails, skip AI refinement.
   b. Initialise the AI response cache.
   c. Gather git context (diff summary, diff sample, recent commit log, changed files)
      scoped to project-relevant files, excluding artifact and dependency directories.
   d. Attempt to load the `step11_git_commit_prompt` template from the shared AI
      helpers YAML configuration file. If successful, populate the template with
      project metadata and git context variables.
   e. If the template cannot be loaded, fall back to a hardcoded structured prompt
      (role + task + approach).
   f. Submit the prompt via the AI cache. The cache key encodes the commit type,
      modified count, and total changes. The `git_specialist` persona is used.
   g. If the AI response contains non-empty content, use it as the commit message.
      Otherwise, use the heuristic message.
4. AI refinement failures are caught and logged as warnings. The heuristic message
   is always used as the fallback.

#### Phase 6 — Commit Changes

Write the commit message to a temporary file and commit using that file to avoid
shell-parsing problems with multiline content. The `--no-verify` flag is passed to
bypass pre-commit hooks (linting was already performed in earlier steps).

If the commit command reports that there is nothing to commit, the condition is
treated as non-fatal and `committed = false` is recorded (the push phase will still
run if `commitsAhead > 0`).

Any other commit error is re-thrown after including the stderr output in the error
message. The temporary file is removed in all cases.

#### Phase 6b — Tag Current Version (Conditional)

Read the project version from `package.json`. If the version is a valid semver string,
create a git tag (`v{version}`). Silently skip tagging when:

- The version is absent or not a valid semver.
- The tag already exists.

If tagging succeeds, push the tag to the remote. Tag-push failures are logged as
warnings but do not abort the step.

#### Phase 7 — Push to Remote

Push the current branch to `origin`.

Before pushing:

1. Stash all changes (staged + unstaged + untracked). Stash failures are non-fatal.
2. Pull and rebase from the remote branch to avoid a non-fast-forward rejection.
3. Restore the stash. Stash-pop failures are logged as warnings.
4. Push the branch.
5. Reopen log file descriptors (the stash/pull/push sequence may atomically rename
   log files, orphaning open file handles).

If `commitsAhead = 0`, the push is skipped and `{ pushed: false, reason: 'up-to-date' }`
is returned.

Push failures are caught and returned as `{ pushed: false, error: … }`. They do not
cause the step to return `success: false`.

#### Phase 8 — Generate Report and Save to Backlog

Format a Markdown report (§6.3) from the collected git state and push result.

If a `backlogManager` is present, write two artifacts:

- **Step issues entry** (`step 12, "Git_Finalization"`): the full formatted Markdown report.
- **Step summary entry** (`step 12, "Git_Finalization"`): a one-line summary describing
  the commit and push outcome.

Return the final `StepResult` (§6.2).

### 5.4 Skip Conditions

Step 12 does not formally skip. The no-changes path (§5.3 Phase 3) returns early with
`success: true` but is not marked as `skipped: true` in the result shape.

---

## 6. Data Shapes

### 6.1 GitState (internal)

Assembled during Phase 1 and threaded through subsequent phases.

| Field | Type | Description |
|---|---|---|
| `branch` | String | Current branch name |
| `commitsAhead` | Integer | Commits ahead of the upstream tracking branch |
| `commitsBehind` | Integer | Commits behind the upstream tracking branch |
| `status` | Object | Parsed git status: `{ modified[], staged[], untracked[], deleted[] }` |
| `categories` | Object | File counts per category (§3.1) |
| `commitType` | String | Inferred commit type identifier (§3.2) |
| `commitScope` | String | Inferred commit scope identifier (§3.2) |
| `totalChanges` | Integer | Total count of all changed files |
| `modifiedCount` | Integer | Count of files in the `modified` list |
| `hasSubmodules` | Boolean | Whether the repository contains submodules |

### 6.2 StepResult

| Field | Type | Present when | Description |
|---|---|---|---|
| `success` | `true` | Always (on normal exit) | Step completed without fatal error |
| `dryRun` | `true` | Dry-run mode | Indicates no real git operations were performed |
| `noChanges` | `true` | Working tree was clean | No new commit was created |
| `branch` | String | `success` is `true` | The branch that was operated on |
| `commitMessage` | String | A commit was generated | The commit message used |
| `pushed` | Boolean | A push was attempted | `true` when push succeeded |
| `report` | String | Full execution path | Markdown-formatted git finalization summary |

**Error path:** when the step encounters an unhandled error, it logs the error and
**re-throws** rather than returning `{ success: false, error: … }`. See §7 rule 1.

### 6.3 Backlog Report Structure

The Markdown report written to the backlog includes:

- **Git Finalization Summary section**: branch, commits ahead and behind, inferred commit
  type and scope, modified file count, total changes.
- **Change Breakdown section**: file counts per category.
- **Commit Message section**: the full commit message used, formatted as a code block.
- **Push Status line**: whether the push succeeded or failed.

---

## 7. Constraints and Rules

1. **Error handling deviation.** Step 12 does **not** comply with the step contract rule
   (§5, rule 4) that forbids propagating exceptions from `execute`. When an unhandled
   error occurs, the step re-throws after logging. Callers **must** wrap their call to
   `execute` in a try/catch or equivalent.

2. The AI prompt template key used is **`step11_git_commit_prompt`** (note: the key
   name references step 11, not step 12). This is not a typo in the implementation;
   it is the key defined in the shared AI helpers YAML. Implementations must not rename
   the key without updating the YAML file.

3. The force-staging of `.ai_workflow/` (Phase 4) **must** be attempted even when the
   broad staging has already run. Skipping the force-add would silently omit workflow
   artifacts that are listed in `.gitignore`.

4. Submodule changes **must** be staged and committed inside each submodule **before**
   staging the parent repository. Staging the parent first would record stale submodule
   pointers.

5. The stash-pull-pop sequence in Phase 7 **must** reopen log file descriptors after
   each git operation. The pull/rebase may atomically rename log files, which would
   orphan open file handles and cause log lines to be lost.

6. The temporary commit-message file **must** always be deleted after the commit
   attempt, regardless of success or failure. Leaving the file behind is a resource
   leak.

7. Backlog writes are best-effort. A failure to write to the backlog **must not** cause
   the step to return failure or re-throw an error.

8. The commit cache key encodes the commit type, modified count, and total changes. A
   cached AI response for the same key may be served on subsequent runs. Callers that
   require a fresh commit message on every run should disable the AI cache.
