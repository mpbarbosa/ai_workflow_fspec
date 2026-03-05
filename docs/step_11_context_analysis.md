# Step 11: Context Analysis — Functional Specification

**Step identifier:** `step_11`
**Step name:** Context Analysis
**Step kind:** `CONTEXT` (see [Step Contract §2.2](./step_contract.md))
**AI-integrated:** No
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md)

---

## 1. Purpose

Step 11 performs a mid-workflow health check. It synthesises accumulated step results, the
current git working-tree state, and elapsed time into a structured context report — and
records strategic recommendations for the operator.

Step 11 answers four questions:

1. **How far through the workflow are we?** — completion rate and qualitative status.
2. **How risky are the current changes?** — impact score based on file count and kind.
3. **Are there outstanding problems?** — aggregated critical issues and warnings from prior
   steps.
4. **How long has the workflow been running?** — elapsed duration, formatted for humans.

The answers are written to the workflow backlog and returned in the step result for
downstream steps to consume.

---

## 2. Step Kind Conformance

Step 11 conforms to the **CONTEXT** step kind as defined in `step_contract.md §2.2`.

| Property | Value |
|---|---|
| Kind identifier | `"context"` |
| Execute signature | `execute(context) → Promise<StepResult>` |
| Can be skipped | **No** — executes unconditionally |
| AI-integrated | **No** — does not call the AI API |

**Logging deviation.** The CONTEXT step contract (`step_contract.md §2.2`) requires that
Context Steps use `this.logger`, injected via the constructor. Step 11 instead imports and
uses the **global logger singleton** directly. The constructor does not accept or store a
`logger` option. This deviates from the prescribed logging convention for CONTEXT steps.

**Error handling deviation.** The step contract requires that all exceptions be caught and
returned as `{ success: false, error }`. Step 11 instead **re-throws** any unhandled
exception from `execute`. Callers must be prepared to catch.

---

## 3. Domain Concepts

### 3.1 Completion Rate

The proportion of prior steps that succeeded (not skipped) relative to the total step
count, expressed as a percentage (0–100, rounded to the nearest integer).

### 3.2 Completion Status

A qualitative label derived from the completion rate:

| Rate | Status |
|---|---|
| ≥ 90% | `excellent` |
| ≥ 75% | `good` |
| ≥ 50% | `moderate` |
| < 50% | `poor` |

### 3.3 Impact Score

A numeric score (0–7) that quantifies how significant the current working-tree changes
are. Points are accumulated as follows:

| Condition | Points |
|---|---|
| Modified files > 10 | +3 |
| Modified files > 5 and ≤ 10 | +2 |
| Modified files > 0 and ≤ 5 | +1 |
| A dependency manifest is modified | +2 |
| A script file is modified | +1 |
| A configuration file is modified | +1 |

### 3.4 Change Impact Level

A qualitative label derived from the impact score:

| Score | Level |
|---|---|
| ≥ 5 | `high` |
| ≥ 3 | `medium` |
| < 3 | `low` |

### 3.5 Issue Aggregation

Prior step results are classified into three buckets. A result object counts toward
exactly one bucket:

| Bucket | Condition |
|---|---|
| `critical` | Result has `success === false` |
| `warnings` | Result has `success === true` and a non-empty `warnings` array |
| `passed` | Result has `success === true` and no warnings |

`total` is the sum of the `critical` and `warnings` counts.

### 3.6 Dependency Manifest Detection

A file (from the combined modified + staged + untracked list) is classified as a
dependency manifest when its path contains any of: `package.json`, `requirements.txt`,
`go.mod`, `Cargo.toml`.

### 3.7 Script File Detection

A file is classified as a script when its path ends with `.sh` or contains `scripts/`.

### 3.8 Configuration File Detection

A file is classified as a configuration file when its path ends with `.yaml`, `.yml`,
`.json`, `.toml`, or contains `.config`.

---

## 4. Constructor Dependencies

Step 11 receives collaborators via a single options map at construction time. All
dependencies are optional; the step instantiates defaults when absent.

| Dependency key | Role | Behaviour when absent |
|---|---|---|
| `fileOps` | File system operations (available for use within the step) | Defaults to a new `FileOperations` instance |
| `backlog` | Writes the context report to the workflow artifact store | Defaults to a new `Backlog` instance |
| `git` | Queries git working-tree status and branch information | Defaults to a new `GitAutomation` instance |
| `startTime` | Workflow start timestamp (milliseconds) for duration calculation | Defaults to `Date.now()` at construction time |

---

## 5. Execute Behaviour

### 5.1 Inputs

The sole argument to `execute` is the **WorkflowContext** object (see
`step_contract.md §4.2`). The following fields are consumed:

| Field | Description |
|---|---|
| `projectRoot` | Absolute path to the project root — passed to git queries |
| `results` | Array of prior step result objects — used for completion and issue aggregation |
| `startTime` | Workflow start timestamp in milliseconds — preferred over the constructor-injected value for duration calculation |
| `completedSteps` | When present as a number, used directly as the completed count; otherwise computed from `results` |
| `totalSteps` | When present as a number, used directly as the total count; otherwise derived as `max(results.length, 11)` |

### 5.2 Execution Phases

Step 11 runs six sequential phases:

#### Phase 1 — Workflow Completion Analysis

Compute the number of completed steps and the total step count. A step result counts as
completed when `success === true` and `skipped` is not `true`. Derive the completion rate
(§3.1) and completion status (§3.2).

#### Phase 2 — Git Repository State Analysis

Query the git service for the current working-tree status at `projectRoot`:

- Whether the directory is a git repository
- Current branch name
- Count of modified, staged, and untracked files
- Commits ahead of remote

When the git query fails for any reason, or when the directory is not a git repository,
the phase produces `{ isGitRepo: false }` and execution continues normally.

#### Phase 3 — Change Impact Assessment

Using the combined list of modified, staged, and untracked files from Phase 2, check for:
dependency manifest modifications (§3.6), script file modifications (§3.7), and
configuration file modifications (§3.8). Compute the impact score (§3.3) and impact level
(§3.4).

When Phase 2 found no git repository, the modified-file count is treated as zero and all
flag checks return `false`.

#### Phase 4 — Issue Aggregation

Apply the classification rules (§3.5) to every entry in the `results` array, producing
counts for `total`, `critical`, `warnings`, and `passed`.

#### Phase 5 — Duration Calculation

Calculate elapsed time in seconds as `round((endTime − startTime) / 1000)`, where
`endTime` is the current clock reading and `startTime` is drawn from the workflow context
field (preferred) or the constructor-injected value (fallback). Format the result as a
human-readable string (examples: `"47s"`, `"2m 15s"`, `"1h 3m 0s"`).

#### Phase 6 — Report Generation and Persistence

Assemble the context data from all prior phases into a Markdown report (§6.3) and write
it to the workflow backlog under step `11`, section `"Context Analysis"`.

### 5.3 Success Determination

The step result's `success` field is `true` when `issues.critical === 0`; otherwise
`false`. A `false` result does not indicate a step execution error — it signals that one
or more prior steps have critical failures requiring operator attention.

---

## 6. Data Shapes

### 6.1 Git Status Record

| Field | Type | Description |
|---|---|---|
| `isGitRepo` | Boolean | Whether the project root is a git repository |
| `branch` | String | Current branch name; `"unknown"` when not determinable |
| `modifiedFiles` | Integer | Count of modified working-tree files |
| `untrackedFiles` | Integer | Count of untracked files |
| `stagedFiles` | Integer | Count of staged files |
| `commitsAhead` | Integer | Commits ahead of remote |

When `isGitRepo` is `false`, the remaining fields are absent.

### 6.2 StepResult

| Field | Type | Present when | Description |
|---|---|---|---|
| `success` | Boolean | Always | `true` when `issues.critical === 0` |
| `completionRate` | Integer | Always | Workflow completion percentage (0–100) |
| `completionStatus` | String | Always | One of: `excellent`, `good`, `moderate`, `poor` |
| `completedSteps` | Integer | Always | Count of successfully completed prior steps |
| `totalSteps` | Integer | Always | Total step count used for rate calculation |
| `gitStatus` | Object | Always | Git status record (§6.1) |
| `changeImpact` | String | Always | One of: `low`, `medium`, `high` |
| `impactScore` | Integer | Always | Raw impact score (0–7) |
| `issues` | Object | Always | Aggregated issue counts: `total`, `critical`, `warnings`, `passed` |
| `duration` | Integer | Always | Workflow elapsed time in seconds |

### 6.3 Backlog Report Format

The Markdown report written to the backlog contains five sections:

**Workflow Completion section:**
- Completion rate percentage
- Steps completed / total steps
- Qualitative status with emoji indicator

**Change Impact Assessment section:**
- Impact level (`HIGH`, `MEDIUM`, or `LOW`) with emoji
- Numeric impact score

**Issues & Warnings section:**
- Total issue count, critical count, warnings count, passed count
- Aggregate status indicator with emoji

**Workflow Metrics section:**
- Elapsed duration as a formatted string

**Recommendations section:**
- Conditionally present; recommendations are generated for: outstanding critical issues,
  an incomplete workflow (< 100% completion), and high-impact changes.

---

## 7. Constraints and Rules

1. Step 11 **must** be positioned after all substantive workflow steps whose results it is
   expected to aggregate. Placing it earlier means prior results are absent and the
   completion rate and issue counts will be inaccurate.

2. The `results` field of the workflow context is the **sole** authoritative source for
   step outcomes. Step 11 must not re-query external systems to determine whether prior
   steps succeeded.

3. Git queries in Phases 2 and 3 reflect the **current** working-tree state at the time
   Step 11 runs. This may differ from the state captured by Step 00 if git operations were
   performed during the workflow run.

4. When any git query fails, the step **must** degrade gracefully by returning
   `{ isGitRepo: false }` for that query. Git errors must not propagate out of the git
   analysis phases.

5. The step **does not** call the AI API and is **not** subject to the
   [AI Prompt Contract](./ai_prompt_contract.md).

6. **Logging deviation:** Step 11 uses the global logger singleton rather than
   `this.logger`. This deviates from the CONTEXT step contract (§2.2). Implementors should
   note this when extending or replacing the step.

7. **Error propagation deviation:** Step 11 re-throws unhandled exceptions from `execute`
   rather than returning `{ success: false, error }`. This deviates from the step contract
   requirement (§5, rule 4). Callers are responsible for wrapping the call in error
   handling.
