# Step 00: Pre-Analysis — Functional Specification

**Step identifier:** `step_00`
**Step name:** Pre-Analysis
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md)

---

## 1. Purpose

Step 00 is the **mandatory opening step** of every workflow run. It runs before any
documentation, testing, or code-quality steps and establishes the shared context that all
subsequent steps consume.

Step 00 answers three questions:

1. **What changed?** — which files were modified, and of which kind?
2. **What is being worked on?** — the project's type and technology stack.
3. **Is the test infrastructure healthy?** — optionally, before spending workflow time.

The answers are captured in a structured `contextUpdate` payload that the orchestrator
merges into the shared workflow context for downstream steps.

---

## 2. Step Kind Conformance

Step 00 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot, context?) → Promise<StepResult>` |
| Can be skipped | **No** — Step 00 must always execute |
| AI-integrated | **No** — does not call the AI API |
| Estimated duration | ≤ 10 seconds under normal conditions |

---

## 3. Domain Concepts

### 3.1 File Category

Every file in a repository is assigned exactly one category. Categories are mutually
exclusive and assigned in priority order (highest priority first):

| Category | Identifier | Priority | Matching Rule |
|---|---|---|---|
| Workflow Artifact | `workflow_artifact` | 1 (highest) | Path begins with the workflow artifacts directory (e.g. `.ai_workflow/`) |
| Documentation | `documentation` | 2 | Path begins with `docs/`, is the project root readme or changelog, or has a `.md` extension |
| Test | `test` | 3 | Path is under a `test` or `tests` directory, or the filename contains `test` / ends with `.test` or `.spec` |
| Source | `source` | 4 | Path is under `src/` or `lib/`, or has a recognised source-code file extension |
| Config | `config` | 5 (lowest / default) | Any configuration file format (yaml, json, toml, etc.) or any file not matching the above rules |

A file is assigned the **first** matching category in the priority order above. If no
rule matches, the file is treated as config.

Test file detection applies to any of the common source-code file extensions, not just a
single language. A file named `widget.test.py` and one named `widgetSpec.ts` are both
classified as test files.

### 3.2 Change Scope

The overall set of changed files is characterised by a single **change scope** value. The
scope is determined from category counts after excluding workflow artifacts.

Scope values and their conditions (evaluated in order; first match wins):

| Scope | Condition |
|---|---|
| `no-changes` | Total modified file count is zero |
| `documentation-only` | Documentation > 0; all other counts are zero |
| `tests-only` | Test > 0; all other counts are zero |
| `source-code` | Source > 0; test, documentation, and config counts are all zero |
| `configuration` | Config > 0; source, test, and documentation counts are all zero |
| `full-stack` | Source > 0 **and** test > 0 **and** documentation > 0 |
| `code-and-tests` | Source > 0 **and** test > 0 |
| `code-and-docs` | Source > 0 **and** documentation > 0 |
| `mixed-changes` | Any other combination |

### 3.3 Project Kind

A **project kind** describes the broad role of the repository (e.g. a Node.js API, a React
SPA, a Python application). Step 00 resolves project kind through a two-stage lookup:

1. **Configuration-sourced** (preferred): read the kind value from the project configuration
   file. When a kind is declared there, it is accepted without further detection and is
   assigned 100% confidence.
2. **Auto-detected** (fallback): when no configuration-declared kind is present, inspect the
   repository's file patterns to infer the kind. The result includes a confidence score.

Both paths produce a project kind record with the fields: kind identifier, human-readable
description, confidence percentage, and detection source (`"config"` or `"auto-detected"`).

### 3.4 Tech Stack

A **tech stack record** describes the implementation technology of the repository:

| Field | Description |
|---|---|
| Primary language | The dominant programming language |
| Build system | Build tool used by the project (absent if none detected) |
| Test framework | Testing library or runner (absent if none detected) |
| Package file | Path to the package manifest (absent if none found) |

---

## 4. Constructor Dependencies

Step 00 receives all external collaborators at construction time via a single dependency
map. All dependencies are optional; the step degrades gracefully when any is absent.

| Dependency key | Role | Behaviour when absent |
|---|---|---|
| `gitOps` | Queries git for commit count, modified files, and status output | Step will fail at runtime when trying to read git state |
| `projectDetection` | Auto-detects project kind from file patterns | Project kind is not determined |
| `techStackDetection` | Detects primary language, build system, and test framework | Tech stack is not reported |
| `projectKindConfig` | Reads the configured project kind from `.workflow-config.yaml` | Config-sourced detection is skipped; auto-detection is used directly |
| `backlogManager` | Writes analysis results to the workflow artifact store | Backlog write is silently skipped |
| `sdkSmokeTest` | Boolean flag; when truthy, triggers the AI SDK smoke test | Smoke test is not performed |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the root of the project under analysis |
| `context` | Optional workflow context object. When present and the `modifiedFiles` field is a non-empty array, it is used as a fallback source for the modified files list (see §5.2.4). |

### 5.2 Execution Sequence

Step 00 executes the following operations in order:

#### 5.2.1 — Query Git Working-Tree State

Obtain three values from the git operations service:

- **Commits ahead**: the number of local commits not yet pushed to the remote.
- **Total changes**: the count of files modified in the working tree.
- **Modified files list**: the list of file paths modified in the working tree.

#### 5.2.2 — Commit-History Fallback

When the working tree is clean (total changes = 0) **and** the caller-supplied context
contains a non-empty `modifiedFiles` list, Step 00 substitutes the context list for the
empty working-tree list.

This handles the case where a push has already been made before the workflow runs: the
working tree appears clean, but the orchestrator has computed the files changed across the
relevant commits. Downstream steps should not see an empty change set in this situation.

When this fallback is activated, the fact must be logged.

#### 5.2.3 — File Classification

Apply the file category rules (§3.1) to every file in the modified files list:

- Each file is assigned exactly one category.
- Workflow-artifact files are **excluded** from the category counts and categorised lists.
- The output is:
  - `counts` — a count of files per category (excluding workflow artifacts).
  - `categorizedFiles` — the list of file paths per category (excluding workflow artifacts).

#### 5.2.4 — Change Scope Detection

Apply the change scope rules (§3.2) to the category counts to produce a single scope value.

#### 5.2.5 — Project Kind Detection

Resolve the project kind using the two-stage lookup (§3.3):
- Query the project kind configuration first.
- Fall back to auto-detection when configuration yields no result.

This stage is skipped when neither `projectKindConfig` nor `projectDetection` is provided.

#### 5.2.6 — Tech Stack Detection

Detect the primary language, build system, and test framework. This stage is skipped when
`techStackDetection` is not provided.

#### 5.2.7 — SDK Smoke Test (Conditional)

When the `sdkSmokeTest` constructor flag is truthy, execute the AI SDK smoke test before
proceeding.

- On failure: Step 00 **immediately** returns `{ success: false, error: … }`. The
  remaining stages are not executed. This is the only condition under which Step 00 exits
  early with failure.
- On success: execution continues normally.

The purpose of this early check is to surface AI SDK connectivity problems before the
workflow spends significant time on earlier steps only to fail at the first AI call (which
typically occurs around step 7).

#### 5.2.8 — Build Analysis Record

Assemble all collected values into an analysis record (see §6.1).

#### 5.2.9 — Log Summary

Format and emit a human-readable summary of the analysis to the workflow log.

#### 5.2.10 — Persist to Backlog

When a `backlogManager` is provided, write two artifacts:

- **Step issues entry** (`step 0, "Pre_Analysis"`): the full analysis in structured
  Markdown, including file counts, tech stack, project kind, smoke test result (if run),
  and the raw modified-files list from git.
- **Step summary entry** (`step 0, "Pre_Analysis"`): a one-line human-readable sentence
  summarising modified files, commits ahead, and change scope.

Backlog writes are best-effort. A failure to write must not cause Step 00 to return
`success: false`.

#### 5.2.11 — Return Result

Return a `StepResult` as defined in §6.2.

### 5.3 Error Handling

Step 00 must not allow any unhandled exception to propagate out of `execute`. All errors
are caught and returned as `{ success: false, error: <message> }`.

---

## 6. Data Shapes

### 6.1 Analysis Record

The intermediate value assembled during execution and included in the result.

| Field | Type | Description |
|---|---|---|
| `commitsAhead` | Integer | Number of local commits not yet pushed |
| `modifiedFiles` | Integer | Count of modified files (after commit-history fallback) |
| `modifiedFilesList` | String[] | File paths used for classification |
| `changeScope` | String | Change scope identifier (§3.2) |
| `fileCounts` | Object | Count per category (keys from FILE_CATEGORY, excluding workflow artifacts) |
| `categorizedFiles` | Object | File path lists per category (same structure as `fileCounts`) |
| `projectKind` | Object or null | Project kind record (§3.3); null when detection was skipped |
| `techStack` | Object or null | Tech stack record (§3.4); null when detection was skipped |
| `smokeTest` | Object or null | Smoke test result (`{ success, status, details }`); null when not run |
| `timestamp` | Integer | Unix timestamp (milliseconds) at the moment the analysis was assembled |

### 6.2 StepResult

| Field | Type | Present when | Description |
|---|---|---|---|
| `success` | Boolean | Always | `true` when Step 00 completed without error |
| `error` | String | `success` is `false` | Description of the failure |
| `analysis` | Analysis record | `success` is `true` | The full analysis (§6.1) |
| `contextUpdate` | Object | `success` is `true` | Subset of analysis values the orchestrator must merge into the workflow context (§6.3) |

### 6.3 ContextUpdate

The `contextUpdate` field is the primary output mechanism. The orchestrator merges these
fields into the shared workflow context so that downstream steps need not re-query git or
repeat project detection.

| Field | Type | Description |
|---|---|---|
| `projectType` | String or null | Project kind identifier (`projectKind.kind`); null when unknown |
| `modifiedFiles` | String[] | Full list of file paths that were classified (from `modifiedFilesList`) |
| `categorizedFiles` | Object | File lists per category |
| `fileCounts` | Object | File counts per category |
| `changeScope` | String | Change scope identifier |
| `commitsAhead` | Integer | Commits ahead of remote |

---

## 7. Step Metadata

Step 00 exposes a metadata record used by the orchestrator for scheduling and reporting.

| Field | Value | Description |
|---|---|---|
| `id` | `0` | Step sequence number |
| `name` | `"Pre-Analysis"` | Human-readable step name |
| `category` | `"analysis"` | Broad step category |
| `estimatedDuration` | `10` | Expected execution time in seconds |
| `canSkip` | `false` | Step 00 must never be skipped |
| `dependencies` | `[]` | No preceding steps required |

---

## 8. Constraints and Rules

1. Step 00 **must** run before any other step in every workflow execution. No step may
   declare it as an optional dependency.

2. Step 00 **must not** be skipped, regardless of change scope or project kind.

3. The `contextUpdate` returned by a successful Step 00 run is the **single source of
   truth** for modified-file information throughout the rest of the workflow. Downstream
   steps must not re-query git independently to build their own file lists.

4. Workflow-artifact files (e.g. files inside `.ai_workflow/`) **must** be excluded from
   category counts and change scope calculations. They are infrastructure noise, not
   meaningful changes.

5. Step 00 does **not** call the AI API. It is not subject to the
   [AI Prompt Contract](./ai_prompt_contract.md).

6. When the `sdkSmokeTest` flag is enabled, a smoke-test failure **must** cause an
   immediate early exit. The backlog write (§5.2.10) must not be attempted in this case.

7. The commit-history fallback (§5.2.2) must be logged when activated. Silent substitution
   of the file list is not permitted.

---

## 9. Backlog Artifact Format

When a `backlogManager` is provided, Step 00 writes a structured Markdown artifact. The
artifact includes:

**Repository Analysis section:**
- Commits ahead of remote
- Count of modified files
- Change scope identifier
- File breakdown: count per category (documentation, tests, source, config)

**Tech Stack section** *(present only when a tech stack was detected)*:
- Primary language
- Build system
- Test framework
- Package manifest path

**Project Kind section** *(present only when a project kind was detected)*:
- Type description and identifier
- Detection confidence percentage

**Test Infrastructure Pre-Validation section** *(present only when smoke test ran)*:
- Status and details
- Note that early validation prevents wasted workflow execution time

**Modified Files List section** *(present only when git status output is available)*:
- Verbatim git status output, formatted as a code block
