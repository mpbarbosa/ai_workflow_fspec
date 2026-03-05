# Step 17: Workflow Summary — Functional Specification

**Step identifier:** `step_17`  
**Step name:** Workflow Summary  
**Step kind:** `CONTEXT` (see [Step Contract §2.2](./step_contract.md))  
**AI-integrated:** No  
**Version:** 2.0.0  
**Status:** Draft  
**Related:** [`step_contract.md`](./step_contract.md)

---

## 1. Purpose

Step 17 is the **mandatory closing step** of every workflow run. It reads the structured
metrics artifact written by the orchestrator during the run, aggregates the results of all
preceding steps, and produces a comprehensive Markdown summary report that is written to
the workflow artifacts directory.

Step 17 answers three questions:

1. **How did the workflow perform overall?** — step counts, success rate, total and average
   duration, cache efficiency.
2. **Which steps were slow or problematic?** — bottleneck detection, failed step
   identification, and cache quality assessment.
3. **What should be improved?** — a prioritised list of actionable recommendations for the
   next run.

---

## 2. Step Kind Conformance

Step 17 conforms to the **CONTEXT** step kind as defined in `step_contract.md §2.2`.

| Property | Value |
|---|---|
| Kind identifier | `"context"` |
| Execute signature | `execute(context) → Promise<StepResult>` |
| Can be skipped | **No** — Step 17 always executes |
| AI-integrated | **No** — does not call the AI API |
| Estimated duration | < 5 seconds under normal conditions |

**Logging convention deviation.** The step contract for CONTEXT steps specifies that
logging is performed via `this.logger` injected through the constructor. Step 17 instead
uses the **global logger singleton** directly, importing it at the module level. This
deviates from the CONTEXT step convention. The constructor does not accept a `logger`
option. Callers cannot redirect Step 17's log output by injecting a custom logger.

**Constructor deviation.** The step contract for CONTEXT steps specifies
`constructor(options = {})` with service dependencies injected via the options map. Step
17 accepts a single argument that may be either a **string** (the workflow artifacts
directory path) or an **object**. When an object is passed, the constructor ignores all
its fields and defaults the workflow artifacts directory to `".ai_workflow"`. This means
no service dependencies (such as a custom file-system layer) can be injected; the step
always uses the underlying filesystem directly.

**Error handling.** Step 17's `execute` method wraps the `generateSummary` call in a
try/catch and returns `{ success: false, error: <message> }` on failure. This conforms
to the step contract rule that prohibits propagating unhandled exceptions. However, the
internal `generateSummary` method re-throws errors rather than returning a result; callers
of that method directly (not via `execute`) should be prepared to catch.

---

## 3. Domain Concepts

### 3.1 Step Phase Grouping

Step 17 groups the steps of a workflow run into six named phases based on each step's
numeric identifier:

| Phase | Step identifiers | Description |
|---|---|---|
| Initialization | Any step with a hex suffix (e.g. `step_0b`, `step_0f`) | Early setup and bootstrapping steps |
| Documentation | Steps 1–2 | Documentation analysis and consistency checking |
| Validation | Steps 3–5 | Testing, quality, and dependency validation |
| Testing | Steps 6–8 | Git automation, test generation, and build |
| Quality | Steps 9–11 | UX, code quality, and context management |
| Finalization | Steps 12 and above | Git commit, markdown lint, version bump, summary |

Steps are assigned by the **first matching rule** in the order above. Steps with a
non-numeric suffix (hex steps) are placed in Initialization regardless of their position
in the execution sequence.

### 3.2 Performance Thresholds

Step 17 applies fixed thresholds when classifying step performance:

| Threshold | Value | Meaning |
|---|---|---|
| Bottleneck | ≥ 300 seconds | Step took 5 minutes or more — high-priority performance concern |
| Slow step | ≥ 120 seconds (and < 300 s) | Step took 2–5 minutes — medium-priority concern |
| Acceptable cache hit rate | 60% | Minimum rate before a caching recommendation is issued |
| Good cache hit rate | 80% | Threshold for reporting cache quality as "excellent" |

### 3.3 Recommendation Types

The recommendation engine produces entries of the following types:

| Type | Trigger condition | Priority |
|---|---|---|
| `performance` | One or more steps exceeded the bottleneck threshold | High |
| `optimization` | One or more slow steps (between thresholds); or failed steps; or success rate below 80% | High / Critical (varies) |
| `caching` | Cache hit rate is below the acceptable threshold | High |
| `parallelization` | Two or more steps in the validation phase or quality phase could potentially run in parallel | Medium |

Recommendations are sorted in the output report from most critical to least critical
(critical → high → medium → low).

### 3.4 Parallelisation Opportunities

Step 17 identifies two specific phase windows where parallel execution may be possible:

- **Validation phase** (steps 3–5): when two or more steps from this range are present in
  the results.
- **Quality phase** (steps 10–11): when two or more steps from this range are present in
  the results.

This detection is heuristic and informational. Step 17 does not verify actual step
dependency constraints.

---

## 4. Constructor Dependencies

Step 17 manages its own I/O using the host platform's filesystem directly. It does not
accept injected service dependencies.

| Constructor argument | Type | Description |
|---|---|---|
| First argument | String or Object | When a string, treated as the workflow artifacts directory path. When an object (or absent), the workflow artifacts directory defaults to `".ai_workflow"`. |

Derived paths from the workflow directory:

| Internal path | Derivation | Purpose |
|---|---|---|
| Metrics directory | `<workflowDir>/metrics` | Contains `current_run.json` |
| Summaries directory | `<workflowDir>/summaries` | Receives the output report |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Source | Description |
|---|---|---|
| `context` | Orchestrator | Workflow context object. `context.workflowRunId` is used as the summary subdirectory name (preferred over the value in `current_run.json`). `context.dryRun` suppresses writing the report file when `true`. |

### 5.2 Execution Phases

Step 17 runs the following phases in order:

#### Phase 1 — Read Metrics Artifact

Read `<workflowDir>/metrics/current_run.json` from the filesystem. Parse the JSON content
to obtain the full metrics record for the current workflow run.

If the file cannot be read or parsed, a warning is logged and an empty metrics object
(with no step records) is substituted. Execution continues.

#### Phase 2 — Aggregate Step Results

Iterate over every entry in the `steps` map of the metrics record. For each step, extract:
step identifier, step name, status (`success`, `failed`, or `skipped`), start time,
end time, and duration in seconds.

Sort the resulting step result list by start time (ascending).

When the metrics object is empty or contains no step entries, an empty list is produced.
All downstream phases tolerate an empty step list.

#### Phase 3 — Calculate Workflow Metrics

Compute the following aggregate values from the step result list:

| Metric | Computation |
|---|---|
| `totalSteps` | Count of all step results |
| `successfulSteps` | Count of results with status `success` |
| `failedSteps` | Count of results with status `failed` |
| `skippedSteps` | Count of results with status `skipped` |
| `totalDuration` | Sum of all step duration values (seconds) |
| `avgDuration` | `totalDuration / totalSteps`; zero when `totalSteps` is zero |
| `successRate` | `(successfulSteps / totalSteps) × 100`; zero when `totalSteps` is zero |

Workflow-level metadata (`startTime`, `endTime`, `workflowRunId`, `version`, `mode`) is
taken from the metrics artifact. The end time defaults to the current instant when the
artifact does not record one.

#### Phase 4 — Generate Execution Timeline

Produce a timeline record containing:
- The ordered list of steps with their identifiers, names, statuses, durations, and
  timestamps.
- Steps grouped by phase (§3.1), where each phase entry lists only steps assigned to
  that phase.

#### Phase 5 — Calculate Cache Efficiency

Read `cache_hits` and `cache_misses` from the metrics artifact. Compute:
- Hit rate: `cache_hits / (cache_hits + cache_misses)`; zero when no requests recorded.
- Quality rating: `"excellent"` when hit rate ≥ 80%, `"good"` when ≥ 60%,
  `"poor"` otherwise.

#### Phase 6 — Generate Recommendations

Evaluate the thresholds and heuristics described in §3.2–§3.4. Produce a list of
recommendation records. Each record carries a type, priority, title, description,
suggested action, and supporting detail lines.

When no threshold is exceeded and no heuristic fires, an empty list is produced.

#### Phase 7 — Format Summary Report

Assemble a structured Markdown document containing:

1. **Header** — workflow run ID, version, mode, start time, end time, total duration.
2. **Executive Summary** — overall status with emoji indicator, success rate, step counts,
   average duration.
3. **Execution Timeline** — one line per step, with status icon and duration.
4. **Phase Breakdown** — one section per phase, showing step count, success count, and
   cumulative phase duration.
5. **Performance Metrics** — tabular view of total and average duration, step count, and
   success rate.
6. **Recommendations** *(present only when the list is non-empty)* — one section per
   recommendation, sorted from most to least critical.
7. **Footer** — generation timestamp and workflow version.

#### Phase 8 — Write Report File

Unless `context.dryRun` is `true`, create the directory
`<workflowDir>/summaries/<workflowRunId>/` and write the formatted report to
`workflow_summary.md` within it.

When `dryRun` is `true`, the report is generated and included in the return value but
is not written to disk.

### 5.3 Skip Conditions

Step 17 has **no skip conditions**. It always produces and returns a summary result.

---

## 6. Data Shapes

### 6.1 WorkflowMetrics

The aggregated numeric record produced in Phase 3.

| Field | Type | Description |
|---|---|---|
| `totalSteps` | Integer | Total number of steps in the run |
| `successfulSteps` | Integer | Steps that completed with status `success` |
| `failedSteps` | Integer | Steps that completed with status `failed` |
| `skippedSteps` | Integer | Steps that completed with status `skipped` |
| `totalDuration` | Number | Sum of all step durations (seconds) |
| `avgDuration` | Number | Mean step duration (seconds) |
| `successRate` | Number | Percentage of successful steps (0–100) |
| `startTime` | String | Workflow start timestamp (ISO 8601) from metrics artifact |
| `endTime` | String | Workflow end timestamp (ISO 8601); current instant if absent in artifact |
| `workflowRunId` | String | Unique run identifier |
| `version` | String | Workflow engine version |
| `mode` | String | Execution mode (e.g. `"auto"`) |

### 6.2 StepResult (success)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `summary` | Object | Nested summary payload (§6.3) |
| `message` | String | `"Workflow summary generated successfully"` |

### 6.3 Summary Payload

| Field | Type | Description |
|---|---|---|
| `metrics` | WorkflowMetrics | Aggregated metrics (§6.1) |
| `timeline` | Object | Execution timeline with `steps[]` and `phases{}` |
| `cacheEfficiency` | Object | `{ cacheHits, cacheMisses, totalRequests, hitRate, quality }` |
| `recommendations` | Object[] | Prioritised recommendation records |
| `report` | String | Full Markdown summary as a string |

### 6.4 Recommendation Record

| Field | Type | Description |
|---|---|---|
| `type` | String | Recommendation type (§3.3) |
| `priority` | String | One of: `"critical"`, `"high"`, `"medium"`, `"low"` |
| `title` | String | Short description |
| `description` | String | Detailed explanation |
| `impact` | String | Impact level (matches priority) |
| `suggestion` | String | Actionable suggestion for the next run |
| `details` | String[] | Supporting detail lines |

### 6.5 StepResult (failure)

| Field | Type | Description |
|---|---|---|
| `success` | `false` | — |
| `error` | String | Error message from the thrown exception |
| `message` | String | `"Failed to generate workflow summary"` |

---

## 7. Constraints and Rules

1. Step 17 **must** run as the final step in every workflow execution. It consumes the
   metrics artifact written by preceding steps and therefore cannot produce meaningful
   output if run earlier.

2. Step 17 **must not** be skipped. A summary is always generated and returned, even when
   the metrics artifact is absent or empty.

3. When the metrics artifact cannot be read, Step 17 **must** continue with an empty step
   list rather than failing. The resulting summary will show zero steps, which accurately
   reflects the absence of recorded data.

4. Step 17 does **not** call the AI API. It is exempt from the
   [AI Prompt Contract](./ai_prompt_contract.md).

5. The `workflowRunId` from `context.workflowRunId` **must** take precedence over the
   value stored in the metrics artifact. This prevents stale run IDs (from a previously
   written metrics file) from appearing in new summary reports.

6. When `context.dryRun` is `true`, the report **must** be generated in memory and
   returned in the result, but **must not** be written to the filesystem. No summary
   directory is created.

7. Parallelisation opportunity detection (§3.4) is **heuristic only**. A recommendation
   of type `parallelization` carries no guarantee that the identified steps can safely run
   in parallel; step dependency constraints must be verified separately.

8. The performance threshold values (§3.2) are fixed constants. They are not
   configurable at runtime via constructor arguments or context fields.

9. **Logging convention deviation** (restated for enforcement): Step 17 uses the global
   logger singleton, not an injected instance. This is a known deviation from the CONTEXT
   step convention. Implementations replacing Step 17 must decide whether to maintain this
   deviation for compatibility or adopt the standard CONTEXT logging pattern.
