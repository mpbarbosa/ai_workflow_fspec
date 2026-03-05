# Step 11.6: AWS Serverless AI Review — Functional Specification

**Step identifier:** `step_11_6`
**Step name:** AWS Serverless AI Review
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 11.6 performs an AI-powered deployment-readiness review of projects of the
`aws_lbs_backend_setup` kind. It uses the `aws_serverless_engineer` AI persona to
evaluate shell scripts, Lambda handler packages, and AWS configuration hygiene.

Step 11.6 is an **opt-in step**: it skips automatically and silently for every project
kind other than `aws_lbs_backend_setup`. For qualifying projects it consumes the output
of Step 11.5 (AWS LBS validation) and produces a structured Markdown review report that
is saved to the workflow backlog.

Step 11.6 slots between Step 11.5 and Step 13 in the workflow sequence.

---

## 2. Step Kind Conformance

Step 11.6 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **Yes** — skips for all non-`aws_lbs_backend_setup` project kinds |
| AI-integrated | **Yes** — calls the Copilot API via `aiHelper` and `aiCache` |
| Estimated duration | 30 seconds (metadata) |

Step 11.6 conforms fully to the step contract error-handling rule: all exceptions are
caught inside `execute` and surfaced as `{ success: false, error: … }`. No exception
propagates to the caller.

Step 11.6 conforms to the [AI Prompt Contract](./ai_prompt_contract.md): it initialises
`aiCache` before the first `withCache` call, uses a structured cache key, and routes AI
requests through `aiHelper.executeRequest`.

---

## 3. Domain Concepts

### 3.1 Target Project Kind

Step 11.6 targets exactly one project kind: `aws_lbs_backend_setup`. This identifier
represents a backend setup project built with AWS serverless technologies (Lambda,
IAM, shell-based deployment automation). Projects of any other kind are outside the
scope of this step.

### 3.2 Review Context

The **review context** is a normalised summary of the AWS-specific artefacts that the
AI should evaluate. It is derived from Step 11.5's output and the project root:

| Field | Source | Description |
|---|---|---|
| `shellScripts` | Step 11.5 result | Paths to shell scripts found in the project |
| `lambdaFunctions` | Step 11.5 result | Paths or identifiers of Lambda handler packages |
| `awsConfigKeys` | Step 11.5 result | AWS configuration keys when the config is valid |
| `projectRoot` | Caller | Absolute path to the project root |

When Step 11.5 output is absent, unavailable, or its fields are not arrays, the review
context uses empty arrays as safe defaults. This allows Step 11.6 to run and produce a
minimal AI review even without Step 11.5 data.

### 3.3 AI Persona

The `aws_serverless_engineer` persona is used for all AI requests. This persona is
defined in the shared AI helpers configuration and is specialised for AWS serverless
architecture, IAM hygiene, Lambda packaging, and shell script safety analysis.

### 3.4 AWS Serverless Prompt

The AI prompt is constructed by the `buildAwsServerlessPrompt` utility from the shared
prompt builder, populated with the review context and project kind. The prompt directs
the AI to evaluate:

- Deployment readiness of shell scripts and Lambda packages.
- IAM permission hygiene.
- Risks in shell script safety and Lambda handler structure.

### 3.5 Cache Key

The cache key for AI responses encodes the step identity and the dimensions of the
review context:

```
"step_11_6|{shellScripts.length}|{lambdaFunctions.length}|{awsConfigKeys joined by comma}"
```

The same shell-script count, Lambda count, and config-key set will reuse a cached
response. A change to any of these values causes a fresh AI request.

---

## 4. Constructor Dependencies

Step 11.6 accepts all collaborators at construction time via a single dependency map.
All dependencies have default fallbacks.

| Dependency key | Role | When absent |
|---|---|---|
| `backlog` | Writes the step review report to the workflow artifact store | A default Backlog instance is created |
| `aiHelper` | Executes AI API requests using the specified persona | A default AiHelper instance is created |
| `aiCache` | Caches and retrieves AI responses by cache key | A default AiCache instance is created |
| `projectKindConfig` | Provides the configured project kind when not passed at execute time | Defaults to `null`; project kind must then be supplied via `options.projectKind` |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the project root |
| `options.projectKind` | Project kind override. When present, takes precedence over the `projectKindConfig` service. |
| `options.step11_5Result` | Direct output from Step 11.5. Takes precedence over `options.results`. |
| `options.results` | Accumulated step result array from the orchestrator. Used to locate Step 11.5 output when `options.step11_5Result` is absent. |

### 5.2 Execution Phases

Step 11.6 executes the following operations in order:

#### Phase 1 — Resolve Project Kind

Determine the project kind using the following priority order:

1. `options.projectKind` — caller-supplied override (highest priority).
2. The `projectKindConfig` service — queried when present.
3. Empty string — when neither source is available.

#### Phase 2 — Project Kind Gate

Evaluate whether the resolved project kind is `aws_lbs_backend_setup`. If the project
kind is any other value (including empty string), Step 11.6 immediately returns a skip
result:

```
{ success: true, skipped: true, reason: <descriptive message> }
```

The skip reason message distinguishes between "wrong kind" and "no kind provided".
No further phases are executed after a skip.

#### Phase 3 — Build Review Context

Locate the Step 11.5 result using the following priority order:

1. `options.step11_5Result` — direct injection (highest priority).
2. The first element in `options.results` whose `stepId` is `'step_11_5'`, using its
   `output` field.
3. An empty object — when neither source is available.

Apply the review-context normalisation rules (§3.2) to produce the review context.
Log the count of shell scripts and Lambda handlers that will be reviewed.

#### Phase 4 — Initialise AI and Prepare Cache

Attempt to initialise the AI helper. If the AI helper is available, initialise the AI
cache.

#### Phase 5 — Build Prompt and Execute AI Review

Executed only when the AI helper is available.

1. Build the AWS serverless prompt using `buildAwsServerlessPrompt`, supplying the
   review context and the resolved project kind.
2. Construct the cache key (§3.5).
3. Execute the AI request through the cache. If a cached response exists for the key,
   return it. Otherwise, submit the prompt to the AI API using the
   `aws_serverless_engineer` persona and cache the response.

When the AI helper is not available, this phase is skipped and `aiResponse` is set to
`null`. A warning is logged.

#### Phase 6 — Format and Save Report

Format a Markdown review report (§6.3) from the AI response and the review context.
Save the report to the workflow backlog under step `'11_6'`, section
`'AWS_Serverless_Review'`. The backlog write is not guarded against failure by this step;
any backlog error will be caught by the outer error handler.

#### Phase 7 — Return Result

Return the step result (§6.2).

### 5.3 Skip Conditions

| Reason | Condition |
|---|---|
| `"Step 11.6 skipped — project kind '{kind}' is not aws_lbs_backend_setup"` | Project kind was resolved but is not the target kind |
| `"Step 11.6 skipped — no project kind provided"` | Project kind resolved to empty string |

All skip results carry `success: true` and `skipped: true`.

### 5.4 Error Handling

All exceptions raised during any phase are caught. On any unhandled error, Step 11.6
returns:

```
{ success: false, error: <exception message> }
```

---

## 6. Data Shapes

### 6.1 Review Context (internal)

| Field | Type | Description |
|---|---|---|
| `shellScripts` | String[] | Shell script paths from Step 11.5; empty array when unavailable |
| `lambdaFunctions` | String[] | Lambda handler paths or identifiers; empty array when unavailable |
| `awsConfigKeys` | String[] | AWS configuration key names; empty array when config invalid or absent |
| `projectRoot` | String | Absolute path to the project root |

### 6.2 StepResult

| Field | Type | Present when | Description |
|---|---|---|---|
| `success` | Boolean | Always | `true` when Step 11.6 completed without error |
| `skipped` | Boolean | Skip | `true` on a skip result |
| `reason` | String | Skip | Human-readable skip message |
| `aiAvailable` | Boolean | Non-skip success | Whether the AI helper was available |
| `aiResponse` | Object or null | Non-skip success | Raw AI response object; `null` when AI was unavailable |
| `reviewContext` | Object | Non-skip success | The normalised review context (§6.1) used for the AI request |
| `report` | String | Non-skip success | Formatted Markdown review report (§6.3) |
| `error` | String | Failure | Exception message when `success` is `false` |

### 6.3 Formatted Markdown Report

The report produced by the format utility and saved to the backlog includes the
following sections:

- **Header:** `# AWS Serverless AI Review`
- **Metadata block:** persona identifier, project root path, count of shell scripts
  reviewed, count of Lambda handlers reviewed.
- **Horizontal rule separator.**
- **AI review content:** the raw text returned by the AI helper. When the AI was
  unavailable, a placeholder note is included instead.

---

## 7. Step Metadata

Step 11.6 exposes a metadata record used by the orchestrator for scheduling and reporting.

| Field | Value | Description |
|---|---|---|
| `id` | `'11_6'` | Step sequence identifier |
| `name` | `'AWS Serverless AI Review'` | Human-readable step name |
| `category` | `'ai_review'` | Broad step category |
| `estimatedDuration` | `30` | Expected execution time in seconds |
| `canSkip` | `true` | Step skips for non-target project kinds |
| `dependencies` | `['step_11_5']` | Step 11.5 output is consumed (optional; falls back to empty context) |

---

## 8. Constraints and Rules

1. Step 11.6 **must** skip immediately when the resolved project kind is not
   `aws_lbs_backend_setup`. It **must not** call the AI API for any other project kind.

2. The project kind resolution **must** follow the priority order defined in §5.2
   Phase 1: caller override first, then the config service, then empty string.

3. The Step 11.5 result lookup **must** follow the priority order in §5.2 Phase 3:
   direct injection first, then the orchestrator results array. When neither is
   available, the review context **must** use empty arrays rather than failing.

4. The `aiCache.init()` call **must** be made before the first `withCache` call, as
   required by the [AI Prompt Contract §6.2](./ai_prompt_contract.md). Failure to
   initialise the cache causes a runtime error on the first cache access.

5. The cache key **must** encode the step identity and the dimensions of the review
   context (§3.5). A key that omits context dimensions risks serving a cached AI
   response generated for a different set of artefacts.

6. When the AI helper is unavailable, Step 11.6 **must** still produce a report and
   save it to the backlog. The report uses a placeholder where AI content would appear.
   Unavailability of the AI is **not** a failure condition.

7. Step 11.6 **must** conform to the [AI Prompt Contract](./ai_prompt_contract.md)
   for all AI request execution. In particular, `aiHelper` and `aiCache` must be
   injected as constructor dependencies.
