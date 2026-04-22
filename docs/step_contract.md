# Step Contract — Functional Specification

<<<<<<< HEAD
**Module:** `steps/step_contract`
**Version:** 1.0.0
=======
**Module:** `steps/step_contract`
**Version:** 1.0.1
>>>>>>> 108d19c (docs(docs): update docs [skip ci])
**Status:** Draft

---

## 1. Purpose

This specification defines the **step contract** — the set of rules that every workflow step must satisfy so the orchestrator can dispatch, execute, and interpret results from any step in a uniform way.

A step is a discrete unit of work within an AI-assisted workflow. The contract governs:

- How a step declares its execution mode (its "kind")
- What arguments the orchestrator passes when invoking a step
- How the step signals success, failure, or intentional skip

---

## 2. Step Kinds

Every step implementation must declare exactly one **step kind**. The kind is a static, immutable property on the step class/type itself (not on an instance). The orchestrator reads this property at dispatch time to determine how to invoke the step.

Two kinds are defined:

---

### 2.1 Project Step (`PROJECT`)

**Identifier:** `"project"`

**Purpose:** Steps that inspect, validate, or analyse files within a project directory.

**Execute signature:**

```
execute(projectRoot, options?) → Promise<StepResult>
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `projectRoot` | Absolute path string | Yes | Root directory of the project being analysed |
| `options` | Key–value map | No | Additional orchestrator-supplied hints |

**Constructor contract:**

The step receives shared orchestrator dependencies at construction time via a single dependency map argument. The step extracts only the dependencies it needs.

```
constructor(deps = {})
  deps.gitOps  — Git operations service
  deps.*       — Any other shared service the orchestrator provides
```

**Logging:** Uses the **global logger singleton** provided by the runtime. The step does not manage a logger instance; it imports and uses the singleton directly. All output is automatically routed to the run log file.

**Known steps of this kind:** 00, 01, 02, 02_5, 03, 04, 05, 07, 08, 09, 0f, 10, 11_5, 11_6

---

### 2.2 Context Step (`CONTEXT`)

**Identifier:** `"context"`

**Purpose:** Steps that perform workflow-level actions — such as git operations, linting, version updates, or summary generation — and require richer context than a bare filesystem path.

**Execute signature:**

```
execute(context) → Promise<StepResult>
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `context` | WorkflowContext object | Yes | Full workflow context (see §4.2) |

**Constructor contract:**

The step is self-contained and receives all dependencies through a single options map at construction time. A logger is always available via this map.

```
constructor(options = {})
  options.logger  — Logger instance (falls back to console if absent)
  options.*       — Any other service options the step requires
```

**Logging:** Uses `this.logger`, injected by the orchestrator from the global singleton at construction time. The step must not import or instantiate its own logger.

**Known steps of this kind:** 06, 0b, 11, 12, 13, 14, 15, 16, 17

---

### 2.3 Analysis Step (`ANALYSIS`)

**Identifier:** `"analysis"` *(observed in practice; not yet formally defined in `step_contract.js`)*

**Purpose:** Steps that perform deep AI-powered analysis of source code and require both a
project root path and structured dependency injection.

**Execute signature:**

```
execute(projectRoot, options?) → Promise<StepResult>
```

This is identical to the PROJECT step signature. The ANALYSIS kind signals to the
orchestrator that the step performs heavyweight AI analysis (e.g., TypeScript review,
debugging analysis) rather than lightweight project inspection.

**Note:** Steps 18 (`step_18_debugging`) and 19 (`step_19_typescript_review`) declare
`STEP_KIND.ANALYSIS` in their exported `STEP_DEFINITION` metadata object, but do **not**
declare `static stepKind` on the class itself. This differs from the PROJECT and CONTEXT
convention. The `ANALYSIS` constant is not yet defined in `step_contract.js`. This is a
known specification gap requiring resolution.

**Known steps of this kind:** 18, 19

---

## 3. Orchestrator Dispatch Rules

The orchestrator selects a dispatch path based on the step kind:

| Step Kind | Invocation |
|---|---|
| `PROJECT` | `step.execute(projectRoot)` |
| `CONTEXT` | `step.execute({ projectRoot, ...context })` |
| `ANALYSIS` | `step.execute(projectRoot)` (same as PROJECT; heavyweight AI analysis) |

The orchestrator must read the kind from the **class/type declaration**, not from an instance, before constructing the step.

---

## 4. Data Types

### 4.1 StepResult

Returned by every `execute` call. Represents the outcome of the step.

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | Boolean | Yes | `true` if the step completed successfully |
| `skipped` | Boolean | No | `true` if the step was intentionally not executed |
| `reason` | String | No | Human-readable explanation for a skip or failure |
| `error` | String | No | Error message when `success` is `false` |
| `summary` | Key–value map | No | Step-specific result payload for downstream consumers |

**Rules:**

- If `skipped` is `true`, `success` should also be `true` unless the skip itself is considered a failure.
- `reason` should be present whenever `skipped` is `true` or `success` is `false`.
- `error` is only meaningful when `success` is `false`.
- The shape of `summary` is step-defined; the orchestrator treats it as opaque.

---

### 4.2 WorkflowContext

Passed to Context Steps as the sole argument to `execute`.

| Field | Type | Required | Description |
|---|---|---|---|
| `projectRoot` | Absolute path string | Yes | Root directory of the project |
| `workflowDir` | String | No | Workflow artifacts directory. Default: `.ai_workflow` |
| `auto` | Boolean | No | `true` when running in non-interactive (automated) mode |
| `gitStats` | Object | No | Git change statistics: `categories` (documentation/tests/code/config counts), `summary`, `insertions`, `deletions`, `gitLog`, `diffSummary`, `diffSample` |
| `modifiedFiles` | String[] | No | Array of modified file paths relative to `projectRoot` |
| `modifiedCount` | Number | No | Count of modified files (length of `modifiedFiles`) |
| `versionConfig` | Object | No | Version configuration loaded from `.workflow-config.yaml` |
| `branch` | String | No | Current git branch name (also available as `currentBranch`) |
| `results` | Object | No | Accumulated step results map, keyed by step ID |
| `stepResults` | Object | No | Alias / structured variant of `results` |
| `startTime` | Number | No | Workflow start timestamp (`Date.now()` at launch) |

Additional fields may be present; a step should ignore fields it does not need.

---

## 5. Step Implementation Contract (Summary)

A conforming step implementation must:

1. Declare a static `stepKind` property equal to one of the defined kind identifiers (`"project"`, `"context"`, or `"analysis"`).
2. Implement an `execute` method matching the signature for its declared kind.
3. Return a `StepResult`-shaped value (always asynchronously / as a promise or equivalent).
4. Never throw an unhandled exception from `execute`; wrap errors and surface them via `StepResult.error` and `StepResult.success = false`.
5. Follow the logging convention for its kind:
   - **Project Steps** use the global logger singleton directly.
   - **Context Steps** use `this.logger` injected via the constructor.
6. AI-integrated steps (those that call the Copilot API) must additionally satisfy the **[AI Prompt Contract](./ai_prompt_contract.md)**, including file content injection, prohibited prompt techniques, and anti-hallucination requirements.

---

## 6. Step Kind Identifiers (Enumeration)

| Name | Value | Description |
|---|---|---|
| `PROJECT` | `"project"` | Project Step execution contract |
| `CONTEXT` | `"context"` | Context Step execution contract |
| `ANALYSIS` | `"analysis"` | Analysis Step — heavyweight AI analysis; same execute signature as PROJECT. *(Observed in `STEP_DEFINITION` exports of steps 18 and 19; not yet defined in `step_contract.js` — specification gap.)* |

The set of valid identifiers is closed. Implementations must not introduce additional kind values without revising this specification.

---

## 7. Step Specifications

Individual step behaviour is documented in dedicated specification files:

| Step | Name | Kind | Specification |
|---|---|---|---|
| `step_00` | Pre-Analysis | PROJECT | [step_00_pre_analysis.md](./step_00_pre_analysis.md) |
| `step_01` | Documentation Analysis | PROJECT · AI | [step_01_documentation_analysis.md](./step_01_documentation_analysis.md) |
| `step_02` | Doc/Code Consistency Analysis | PROJECT · AI | [step_02_consistency_analysis.md](./step_02_consistency_analysis.md) |
| `step_02_5` | Documentation Optimization | PROJECT | [step_02_5_documentation_optimization.md](./step_02_5_documentation_optimization.md) |
| `step_03` | Script Reference Validation | PROJECT · AI | [step_03_script_references.md](./step_03_script_references.md) |
| `step_04` | Configuration Validation | PROJECT · AI | [step_04_config_validation.md](./step_04_config_validation.md) |
| `step_05` | Directory Structure Analysis | PROJECT · AI | [step_05_directory_analysis.md](./step_05_directory_analysis.md) |
| `step_06` | Test Review | CONTEXT · AI | [step_06_test_review.md](./step_06_test_review.md) |
| `step_07` | Test Generation | PROJECT · AI | [step_07_test_generation.md](./step_07_test_generation.md) |
| `step_08` | Test Execution | PROJECT · AI | [step_08_test_execution.md](./step_08_test_execution.md) |
| `step_09` | Dependency Validation | PROJECT · AI | [step_09_dependency_validation.md](./step_09_dependency_validation.md) |
| `step_0b` | Bootstrap Documentation | CONTEXT · AI | [step_0b_bootstrap_documentation.md](./step_0b_bootstrap_documentation.md) |
| `step_0f` | Commit Artifacts | PROJECT | [step_0f_commit_artifacts.md](./step_0f_commit_artifacts.md) |
| `step_10` | Code Quality Analysis | PROJECT · AI | [step_10_code_quality.md](./step_10_code_quality.md) |
| `step_11` | Context Health Analysis | CONTEXT · AI | [step_11_context_analysis.md](./step_11_context_analysis.md) |
| `step_11_5` | AWS LBS Validation | PROJECT · AI | [step_11_5_aws_lbs_validation.md](./step_11_5_aws_lbs_validation.md) |
| `step_11_6` | AWS Serverless Review | PROJECT · AI | [step_11_6_aws_serverless_review.md](./step_11_6_aws_serverless_review.md) |
| `step_12` | Git Finalization | CONTEXT · AI | [step_12_git_finalization.md](./step_12_git_finalization.md) |
| `step_13` | Markdown Lint | CONTEXT · AI | [step_13_markdown_lint.md](./step_13_markdown_lint.md) |
| `step_14` | Prompt Engineering Review | CONTEXT · AI | [step_14_prompt_engineering.md](./step_14_prompt_engineering.md) |
| `step_15` | UX Analysis | CONTEXT · AI | [step_15_ux_analysis.md](./step_15_ux_analysis.md) |
| `step_16` | Version Update | CONTEXT · AI | [step_16_version_update.md](./step_16_version_update.md) |
| `step_17` | Workflow Summary | CONTEXT | [step_17_workflow_summary.md](./step_17_workflow_summary.md) |
| `step_18` | Debugging Analysis | ANALYSIS · AI | [step_18_debugging.md](./step_18_debugging.md) |
| `step_19` | TypeScript Review | ANALYSIS · AI | [step_19_typescript_review.md](./step_19_typescript_review.md) |
