# Step 19: TypeScript Review — Functional Specification

**Step identifier:** `step_19`
**Step name:** TypeScript Review
**Step kind:** `ANALYSIS` (see [Step Contract §2.3](./step_contract.md) and §2 below)
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 19 reviews TypeScript source files for type safety, strict-mode compliance, and
common anti-patterns. It uses the `typescript_developer_prompt` AI persona, nicknamed
**Strider**, which is specialised for TypeScript analysis.

Step 19 skips gracefully for projects that contain no TypeScript files. For qualifying
projects it produces a structured analysis report — including a heuristic issue score
computed before the AI is called — and saves the report to the workflow backlog.

Step 19 is the **reference implementation** for AI prompt hygiene in this workflow. Its
file content injection pattern, budget enforcement, and prompt construction are the
canonical examples cited in the [AI Prompt Contract](./ai_prompt_contract.md).

Step 19 depends on Step 18 (Debugging Analysis) as a preceding step.

---

## 2. Step Kind Conformance

Step 19 declares the **ANALYSIS** step kind, as defined in `step_contract.md §2.3`.

| Property | Value |
|---|---|
| Kind identifier | `"analysis"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **Yes** — skips when no TypeScript files are found |
| AI-integrated | **Yes** — calls the Copilot API via `aiHelper` and `aiCache` |
| Estimated duration | Variable; depends on TypeScript file count and AI availability |

**⚠ Deviation from step contract — kind declaration location.** The ANALYSIS kind is
declared in the exported `STEP_DEFINITION` metadata object, not as a static property on
the step class itself. The step class carries no `stepKind` declaration. This is the same
deviation observed in Step 18 and documented in `step_contract.md §2.3`. Until the
contract is updated, orchestrator implementations dispatching ANALYSIS steps must read
the kind from `STEP_DEFINITION.kind` rather than from the class.

Step 19 conforms fully to the step contract error-handling rule: all exceptions are
caught inside `execute` and surfaced as `{ success: false, error: … }`. No exception
propagates to the caller.

Step 19 is the **canonical conforming implementation** of the
[AI Prompt Contract](./ai_prompt_contract.md). It satisfies every requirement in that
contract:

- Reads all TypeScript files within the character budget (§3.6).
- Uses `buildFileContentBlock` for each file (§3.6).
- Passes `file_contents` as a named section in the prompt (§3.6).
- Wraps each file read individually (§3.6).
- Stops adding files when the total budget is reached (§3.6).
- Does not reference `@workspace` or any IDE-specific feature (§4 of the AI Prompt Contract).

---

## 3. Domain Concepts

### 3.1 TypeScript Project Detection

A project qualifies for Step 19 analysis when at least one file in the discovered or
supplied file list has a `.ts` or `.tsx` extension (case-insensitive, including `.mts`,
`.cts`, `.mtsx`, `.ctsx` variants matched by the pattern `/\.[cm]?tsx?$/i`).

If no such file is found, the step skips and saves a minimal skip report to the backlog.

### 3.2 TypeScript Source File Discovery

Step 19 discovers TypeScript source files by scanning the project root for files
matching `**/*.ts` and `**/*.tsx`. The following directories are excluded from
scanning: `node_modules`, `.git`, `dist`, `build`, `coverage`, `test`, `__tests__`,
`docs`.

The `docs` directory is excluded because it contains only documentation files (Markdown,
generated HTML assets, API references, etc.). TypeDoc-generated JavaScript bundles and
search indices found under `docs/api/html/` are not TypeScript source and must not be
injected into a TypeScript review prompt.

The result is de-duplicated. The total list is capped at **100 files**.

When `options.sourceFiles` is provided, discovery is bypassed entirely.

### 3.3 Sample Window

The AI analysis operates on a **sample** of up to 20 files taken from the front of the
discovered (or supplied) TypeScript file list. The issue score (§3.4) is also computed
over this sample. The full file count is reported in the result.

### 3.4 Heuristic Issue Score

Before calling the AI, Step 19 computes a heuristic issue score over the combined text
of the sample file contents. The score measures the density of common TypeScript
anti-patterns:

| Metric | Patterns counted |
|---|---|
| `anyCount` | `: any` (explicit any annotation) and `as any` (unsafe type assertion) |
| `tsIgnoreCount` | `@ts-ignore` (suppressed type error) and `@ts-nocheck` (whole-file suppression) |
| `missingReturnTypeCount` | Exported or async function declarations that appear to lack an explicit return type |
| `totalIssues` | Sum of the above three counts |

The score is purely heuristic: it is derived from pattern matching on the text, not from
a type-checked parse tree. It is included in the backlog report as a quick signal and as
context for the AI prompt.

The score is computed regardless of AI availability.

### 3.5 AI Persona

Step 19 uses the `typescript_developer_prompt` persona from the shared AI helpers YAML
configuration. This persona is nicknamed **Strider** and is specialised for TypeScript
type-safety review, strict-mode compliance, and idiomatic-pattern guidance.

The AI request uses the `typescript_reviewer` persona identifier and, when supported,
the `claude-haiku-4.5` model.

### 3.6 File Content Injection

Step 19 injects actual TypeScript file content into the prompt, conforming to the
[AI Prompt Contract §3](./ai_prompt_contract.md) and serving as its reference
implementation:

- Each file's content is read individually from disk. Unreadable files yield an empty
  string; they are not treated as errors.
- Each non-empty content is wrapped with `buildFileContentBlock`.
- The per-file budget is `MAX_CHARS_PER_FILE` (4,000 characters); content is
  truncated internally by the block builder.
- Files are added until the cumulative total reaches `MAX_CHARS_TOTAL_CONTENTS`
  (30,000 characters); no further files are added once the budget is exhausted.
- The resulting blocks are joined and labelled as a `**File Contents**` section.

### 3.7 Prompt Assembly

The `typescript_developer_prompt` template may use either the standard structure
(with `task_template` placeholders) or the non-standard structure (with `role_prefix` /
`specific_expertise`). Step 19 handles both:

**Primary path (structured template with `task_template`):** Assemble the prompt by
joining the following sections in order:

1. Role declaration (`role_prefix` or `role` field).
2. Behavioural guidelines.
3. Task description — the `task_template` field with the following placeholders
   substituted:

   | Placeholder | Value |
   |---|---|
   | `{project_name}` | `options.projectName` or the project root directory name |
   | `{project_description}` | `'TypeScript project'` (literal) |
   | `{project_kind}` | `options.projectKind` or `'nodejs_api'` |
   | `{primary_language}` | `'TypeScript'` (literal) |
   | `{build_system}` | `'tsc / Vite / Webpack'` (literal) |
   | `{test_framework}` | `'Jest / ts-jest / Vitest'` (literal) |
   | `{test_command}` | `'npm test'` (literal) |
   | `{lint_command}` | `'npm run lint'` (literal) |
   | `{modified_count}` | Total count of discovered TypeScript files |

4. TypeScript files list: total count, sample size, and comma-separated sample file
   paths.
5. File content blocks (§3.6).
6. Approach guidance.

**Fallback path (generic builder):** When the template does not match the primary
structure, the generic YAML step prompt builder is used with `project_name`,
`source_files`, and `file_count` as variables. File content blocks are appended.

### 3.8 Cache Key

The AI response cache key encodes the step identity, reviewer persona, project root,
and total TypeScript file count:

```
"step_19|typescript_reviewer|{projectRoot}|{tsFiles.length}"
```

A change in the project root path or TypeScript file count produces a fresh AI request.

---

## 4. Constructor Dependencies

Step 19 accepts all collaborators at construction time via a single options map. All
have default fallbacks.

| Dependency key | Role | When absent |
|---|---|---|
| `fileOps` | Reads file contents and runs glob patterns for source discovery | A default FileOperations instance is created |
| `backlog` | Writes the review report to the workflow artifact store | A default Backlog instance is created |
| `aiHelper` | Executes AI API requests using the specified persona | A default AiHelper instance is created (with optional `promptsDir`) |
| `aiCache` | Caches and retrieves AI responses by cache key | A default AiCache instance is created |
| `promptsDir` | Path to the prompts directory, forwarded to the default AiHelper | Defaults to `null` |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the project root |
| `options.sourceFiles` | Override list of TypeScript file paths (relative). When present, skips discovery (§3.2). |
| `options.projectName` | Project name for prompt context. Defaults to the project root directory name. |
| `options.projectKind` | Project kind for prompt context. Defaults to `'nodejs_api'` in placeholder substitution. |

### 5.2 Execution Phases

Step 19 executes the following operations in order:

#### Phase 1 — Discover TypeScript Files

When `options.sourceFiles` is provided, use it directly. Otherwise, scan the project
root using the patterns defined in §3.2 to produce a de-duplicated list of up to 100
TypeScript file paths.

#### Phase 2 — TypeScript Project Gate

Apply the TypeScript project detection rule (§3.1) to the file list. If no TypeScript
files are found:

1. Format a skip report using `formatTypeScriptReport` with `skipped: true`.
2. Save the skip report to the backlog under step `19`, section `'TypeScript_Review'`.
3. Return a skip result (§6.4).

No further phases are executed after a skip.

#### Phase 3 — Sample and Read Contents

Take the first 20 files from the TypeScript file list (the sample window, §3.3). Read
each sample file's content from disk. Unreadable files produce an empty string; they
are not treated as errors.

Log the count of TypeScript files being reviewed.

#### Phase 4 — Compute Issue Score

Apply the heuristic issue score (§3.4) to the array of sample file contents. Log the
resulting score breakdown.

#### Phase 5 — Initialise AI

Attempt to initialise the AI helper. If unavailable, log a warning. Phases 6 and 7
are skipped; execution continues from Phase 8.

#### Phase 6 — Load Prompt Configuration and Build File Content Blocks

Load the shared AI helpers YAML configuration file. Build file content blocks from
the sample file contents, respecting the per-file and total character budgets (§3.6).

#### Phase 7 — Assemble Prompt and Request AI

Assemble the prompt using the primary or fallback path (§3.7). When a prompt was
successfully assembled, execute the AI request through the cache using the cache key
(§3.8). Extract the response content string from the AI result.

When the AI request or prompt assembly fails, log a warning and continue with an empty
content string. These failures do not cause the step to return `success: false`.

#### Phase 8 — Format and Save Report

Produce a Markdown report (§6.3) from the sample files analysed, the AI content, and
the issue score. Save the report to the backlog under step `19`, section
`'TypeScript_Review'`.

#### Phase 9 — Return Result

Return the step result (§6.2). Log whether AI content was produced.

### 5.3 Skip Conditions

| Reason | Condition |
|---|---|
| `no TypeScript files` | No file in the discovered or supplied list has a `.ts` / `.tsx` extension |

The skip result carries `success: true`, `skipped: true`, and `filesAnalyzed: []`.

### 5.4 Error Handling

All exceptions raised during any phase are caught at the outer level. On any unhandled
error, Step 19 returns:

```
{ success: false, error: <exception message> }
```

AI-level failures (YAML load error, API error) are caught at the inner level, logged
as warnings, and produce an empty AI content string. These do **not** cause the step
to return failure.

---

## 6. Data Shapes

### 6.1 STEP_DEFINITION (exported metadata)

| Field | Value | Description |
|---|---|---|
| `id` | `'step_19'` | Step identifier |
| `name` | `'TypeScript Review'` | Human-readable name |
| `kind` | `STEP_KIND.ANALYSIS` | Step kind (see §2 deviation note) |
| `description` | `'AI-powered TypeScript review using the "Strider" TypeScript Developer persona'` | — |
| `dependencies` | `['step_18']` | Depends on the Debugging Analysis step |

### 6.2 StepResult (success, not skipped)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `filesAnalyzed` | String[] | The sample file paths analysed (up to 20) |
| `totalTsFiles` | Integer | Total TypeScript file count before sampling |
| `issueScore` | Object | Heuristic issue score (§6.5) |
| `aiContent` | String | AI review text; empty string when AI was unavailable or prompt missing |
| `report` | String | Formatted Markdown report (§6.3) |

### 6.3 Formatted Markdown Report

The report produced and saved to the backlog has the following structure when the step
ran (not skipped):

```
# Step 19: TypeScript Review — Strider

## Files Analyzed
- {file path}
…

## Issue Score (Heuristic)

| Metric | Count |
|--------|-------|
| Explicit `any` / `as any` | {anyCount} |
| `@ts-ignore` / `@ts-nocheck` | {tsIgnoreCount} |
| Functions missing return type | {missingReturnTypeCount} |
| **Total** | **{totalIssues}** |

## AI Analysis

{AI content, or placeholder when unavailable}
```

When the step was skipped (no TypeScript files found), the report contains only the
heading and a note that no TypeScript files were detected.

### 6.4 StepResult (skip)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `filesAnalyzed` | `[]` | Empty list |
| `totalTsFiles` | `0` | — |
| `aiContent` | `''` | Empty string |
| `report` | String | The skip-variant Markdown report |

### 6.5 Issue Score

| Field | Type | Description |
|---|---|---|
| `anyCount` | Integer | Count of `: any` and `as any` occurrences across sample files |
| `tsIgnoreCount` | Integer | Count of `@ts-ignore` and `@ts-nocheck` occurrences |
| `missingReturnTypeCount` | Integer | Heuristic count of exported/async functions without explicit return types |
| `totalIssues` | Integer | Sum of the above three counts |

### 6.6 StepResult (failure)

| Field | Type | Description |
|---|---|---|
| `success` | `false` | Fatal error occurred |
| `error` | String | Exception message |

---

## 7. Constraints and Rules

1. **Step kind declaration deviation.** Step 19 does **not** declare `static stepKind`
   on the class. The ANALYSIS kind is declared only in the exported `STEP_DEFINITION`
   object. This is the same deviation as Step 18 and is documented in
   `step_contract.md §2.3`. Orchestrator implementations must read the kind from
   `STEP_DEFINITION.kind`.

2. Step 19 is the **reference implementation** for the
   [AI Prompt Contract](./ai_prompt_contract.md). Its file content injection pattern
   (§3.6) is the canonical example. Modifications to Step 19 that weaken prompt hygiene
   — such as removing file content injection or introducing `@workspace` references —
   are contract violations.

3. File content injection **must** comply with the [AI Prompt Contract §3](./ai_prompt_contract.md):
   each file read wrapped individually, `buildFileContentBlock` used for each non-empty
   file, per-file budget enforced by the block builder, total budget
   (`MAX_CHARS_TOTAL_CONTENTS`) enforced by the caller with an early stop.

4. The skip gate (Phase 2) **must** save a skip report to the backlog before returning.
   A silent skip that leaves no backlog entry is not permitted.

5. The heuristic issue score (§3.4) **must** be computed before the AI call. It is
   included in the report regardless of AI availability.

6. The sample window **must** be capped at 20 files. The AI must not receive more than
   20 source files regardless of the total discovered count.

7. When the AI helper is unavailable, Step 19 **must** still produce and save a report.
   AI unavailability is not a failure condition.

8. Inner AI errors (YAML load failure, API error) **must** be caught and logged as
   warnings. They **must not** propagate to the outer error handler or cause the step
   to return `success: false`.

9. The `aiCache.init()` call **must** be made before the first `withCache` call, as
   required by the [AI Prompt Contract §6.2](./ai_prompt_contract.md).

10. The total TypeScript file list **must** be de-duplicated and capped at 100 files
    before sampling. The cap is applied in discovery (§3.2); the sample window (§3.3)
    is applied afterward.

11. **The `docs` directory must never be included in TypeScript source file discovery.**
    The `docs` folder contains only documentation files and generated assets (Markdown,
    TypeDoc HTML, search indices, navigation data, etc.). These are not TypeScript
    source and must not be injected into a TypeScript review prompt.
