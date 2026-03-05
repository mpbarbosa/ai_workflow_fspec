# Step 11.5: AWS LBS Validation — Functional Specification

**Step identifier:** `step_11_5`
**Step name:** AWS LBS Validation
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))
**AI-integrated:** Conditionally — opt-in only (see §4)
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 11.5 validates the structure and quality of `aws_lbs_backend_setup` projects — a
specialised project kind for serverless AWS backends. It examines three aspects:

1. **Shell script best practices** — shebang presence and strict-mode setup.
2. **Lambda function structure** — required files per function directory.
3. **AWS configuration schema** — presence of recognised configuration keys in
   `aws-config.json`.

For any other project kind, the step skips immediately without performing file I/O.

When an `aiHelper` dependency is explicitly injected at construction time, the step also
performs an optional AI-powered architectural review using the `devops_engineer` persona.
The AI analysis enriches the backlog report but does not affect the validation outcome or
the `success` field.

Step 11.5 is positioned between Step 11 (context analysis) and Step 12 (git finalisation)
so that git finalisation remains the last executed step.

---

## 2. Step Kind Conformance

Step 11.5 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **Yes** — skips for any project kind other than `aws_lbs_backend_setup` |
| AI-integrated | **Conditionally** — AI is opt-in; only active when `aiHelper` is explicitly injected |
| AI persona | `devops_engineer` |

**Success field quirk.** The step result's `success` field is computed as
`totalIssues === 0 OR shellIssues.length === 0`. This means `success` is `true` whenever
all shell scripts are clean, even when Lambda structure or AWS configuration issues exist.
This is a known behaviour; callers that need a strict all-passing outcome must inspect
`summary.passed` instead.

---

## 3. Domain Concepts

### 3.1 Project Kind Gate

The step targets exclusively the `aws_lbs_backend_setup` project kind. Any other kind —
including an absent or empty value — causes an immediate skip. The kind is resolved in
order from:

1. `options.projectKind` (caller-supplied override), when present.
2. The `projectKindConfig` service, when injected (asynchronous lookup).
3. Empty string (fallback when neither source is available).

### 3.2 Shell Script Best Practices

A shell script (a file ending in `.sh` or `.bash`) is compliant when it satisfies both
rules:

| Rule | Requirement |
|---|---|
| Shebang | The first line of the file must begin with `#!` |
| Strict mode | The file content must contain `set -euo pipefail` in any recognised form |

Each violated rule produces one issue string associated with the script's path.

### 3.3 Lambda Function Structure

Lambda function directories are identified by the path pattern
`src/lambda/<function-name>/index.js`. A valid Lambda function directory must contain
both:

- `index.js` — the handler entry point
- `package.json` — the function's dependency manifest

A missing file produces one issue entry per absence.

### 3.4 AWS Configuration Schema

An `aws-config.json` file is searched at two candidate locations in order:

1. `<projectRoot>/aws-config.json`
2. `<projectRoot>/src/aws-config.json`

The first readable file is used. The configuration is valid when the parsed JSON object
contains at least one of the following keys: `region`, `stackName`, `apiId`, or `mapName`.
Any single key from this set is sufficient.

When no configuration file is found at either location, the validation is treated as
failed with reason `"aws-config.json not found"`.

### 3.5 File Discovery

All validations operate on a flat list of relative file paths produced by a recursive
scan of `projectRoot`. The following directories are always excluded:
`node_modules`, `.git`, `dist`, `build`, `coverage`.

---

## 4. Constructor Dependencies

Step 11.5 receives all collaborators via a single dependency map at construction time.

| Dependency key | Role | Behaviour when absent |
|---|---|---|
| `fileOps` | File system operations for reading the AI helpers configuration | Defaults to a new `FileOperations` instance |
| `backlog` | Writes the validation report to the workflow artifact store | Defaults to a new `Backlog` instance |
| `projectKindConfig` | Resolves the project kind when not supplied via `options` | `null`; kind resolution falls back to an empty string |
| `aiHelper` | Executes AI architectural review requests (opt-in) | `null`; the AI phase is skipped entirely |
| `aiCache` | Caches AI responses; activated only when `aiHelper` is injected | Defaults to a new `AiCache` instance when `aiHelper` is present; `null` otherwise |

**AI dependency note.** Unlike other AI-integrated steps, `aiHelper` and `aiCache` are
never instantiated by default. The step conforms to the [AI Prompt Contract](./ai_prompt_contract.md)
only when the AI phase is active (i.e. `aiHelper` is explicitly injected).

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the project root |
| `options.projectKind` | Optional caller-supplied override for the project kind identifier |

### 5.2 Execution Phases

Step 11.5 runs the following phases in order:

#### Phase 1 — Project Kind Gate

Resolve the project kind (§3.1). When it is not `aws_lbs_backend_setup`, return
immediately with a skip result (§5.3). Execution does not proceed past this phase for
non-targeted projects.

#### Phase 2 — File Discovery

Recursively scan `projectRoot`, excluding noise directories (§3.5), to produce a flat
list of relative file paths. This list is shared by all subsequent validation phases.

#### Phase 3 — Shell Script Validation

From the full file list, identify shell scripts (paths ending in `.sh` or `.bash`). For
each identified script:

1. Read the file content.
2. Apply the best-practice checks (§3.2).
3. Collect any violation messages.

Unreadable files are skipped silently and do not produce violations.

#### Phase 4 — Lambda Function Structure Validation

From the full file list, identify Lambda handler paths matching
`src/lambda/<function-name>/index.js`. Group all `src/lambda/<function-name>/` paths by
function name. For each function, verify that both `index.js` and `package.json` are
present (§3.3). Collect paths of any missing files.

#### Phase 5 — AWS Configuration Validation

Search for `aws-config.json` at the two candidate locations (§3.4). Read and parse the
first found file as JSON. Apply the schema check and record whether the configuration is
valid and, when it is not, the reason.

#### Phase 6 — Report Generation and Persistence

Compute the validation summary from the results of Phases 3–5 (§6.1). Format a Markdown
report (§6.4) and write it to the workflow backlog under step `"11_5"`, section
`"AWS_LBS_Validation"`.

#### Phase 7 — AI Architectural Review (Optional)

Runs only when `aiHelper` was injected and initialises successfully:

1. Initialise `aiCache`.
2. Load the AI helpers YAML configuration and extract the `aws_cloud_architect_prompt`
   template.
3. Populate the template with: project name (the `projectRoot` path), shell script count,
   Lambda function count, shell issue count, AWS config validity, and total issue count.
4. Compute a cache key: `step_11_5|<projectRoot>|<totalIssues>`.
5. Execute the prompt via `aiCache.withCache` using the `devops_engineer` persona.
6. When a non-empty response is received, append an `"AWS Architecture Review"` section
   to the report and overwrite the backlog entry with the enriched report.

When the AI phase fails for any reason, the failure is silently swallowed and the
original report is retained.

### 5.3 Skip Conditions

| Reason | Condition |
|---|---|
| Non-targeted project kind | Project kind is anything other than `aws_lbs_backend_setup`, or no kind is resolvable |

The skip result is `{ success: true, skipped: true, reason: "<descriptive message>" }`.

---

## 6. Data Shapes

### 6.1 Validation Summary

| Field | Type | Description |
|---|---|---|
| `shellScriptCount` | Integer | Total shell scripts discovered |
| `shellIssueCount` | Integer | Total shell script best-practice violations |
| `lambdaFunctionCount` | Integer | Total Lambda handler entry points discovered |
| `lambdaStructureValid` | Boolean | `true` when all Lambda directories have both required files |
| `awsConfigValid` | Boolean | `true` when a valid `aws-config.json` was found |
| `awsConfigReason` | String | Reason for invalidity; empty when valid |
| `totalIssues` | Integer | Sum of: shell violations + missing Lambda files + AWS config failures (0 or 1) |
| `passed` | Boolean | `true` when `totalIssues === 0` |

### 6.2 Lambda Structure Result

| Field | Type | Description |
|---|---|---|
| `valid` | Boolean | `true` when all Lambda directories have both required files |
| `missingFiles` | String[] | Relative paths of missing `index.js` or `package.json` files |

### 6.3 StepResult (non-skipped)

| Field | Type | Present when | Description |
|---|---|---|---|
| `success` | Boolean | Always | `true` when `totalIssues === 0` **or** `shellIssues.length === 0` (see §2) |
| `skipped` | `false` | Always | Indicates the step executed normally |
| `shellScripts` | String[] | Always | Relative paths of all discovered shell scripts |
| `shellIssues` | String[] | Always | All shell best-practice violation messages |
| `lambdaFunctions` | String[] | Always | Relative paths of all Lambda `index.js` handler files |
| `lambdaStructureResult` | Object | Always | Lambda structure check result (§6.2) |
| `awsConfigValid` | Boolean | Always | Whether `aws-config.json` passed the schema check |
| `summary` | Object | Always | Validation summary (§6.1) |
| `report` | String | Always | The full Markdown validation report |

### 6.4 StepResult (skipped)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `reason` | String | Description of why the step was skipped |

### 6.5 Backlog Report Format

The Markdown report contains:

**Header:** Overall pass/fail status and total issue count.

**Shell Scripts section:** Count of scripts found, count of issues, and — when issues
exist — a bulleted list of violation messages. When all scripts pass, a success note is
shown.

**Lambda Functions section:** Count of functions found, structure validity, and — when
invalid — a bulleted list of missing file paths.

**AWS Configuration section:** Whether `aws-config.json` is valid and the reason when it
is not.

**Recommendations section:** Present only when `passed` is `false`. Provides numbered
action items for each failing check.

**AWS Architecture Review section:** Present only when the AI phase ran and produced a
non-empty response. Appended as an additional Markdown section.

---

## 7. Constraints and Rules

1. Step 11.5 **must** perform the project kind gate check (Phase 1) before any file I/O.
   No directory scans, file reads, or git queries may occur before the gate result is
   known.

2. The excluded directories (`node_modules`, `.git`, `dist`, `build`, `coverage`) **must**
   never appear in the file list used for validation. Including them would produce false
   positives and degrade performance.

3. Shell script file reads **must** handle unreadable files gracefully by skipping them
   silently. A single unreadable script must not abort the validation of remaining scripts.

4. The `success` field in the step result is `true` when either `totalIssues === 0` or
   `shellIssues.length === 0`. This means Lambda structure or AWS config failures alone do
   not set `success` to `false`. Callers that require a strict all-passing outcome must
   inspect `summary.passed`.

5. The AI architectural review **must** be treated as best-effort. Any exception during
   AI initialisation, prompt building, or response retrieval must be caught and silently
   ignored. The validation outcome must not depend on AI availability.

6. When AI is active, the cache key **must** encode the project root and the total issue
   count. This ensures a fresh review is requested whenever the issue count changes between
   runs.

7. Backlog writes **must not** cause the step to return `success: false`. A failed backlog
   write is a best-effort infrastructure concern, not a validation failure.

8. `aiHelper` and `aiCache` are **opt-in** constructor dependencies. Unlike other
   AI-integrated steps, they are never instantiated by default. Callers that need AI
   enrichment must inject `aiHelper` explicitly.
