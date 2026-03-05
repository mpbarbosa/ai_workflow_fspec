# Step 18: Debugging Analysis — Functional Specification

**Step identifier:** `step_18`  
**Step name:** Debugging Analysis  
**Step kind:** `ANALYSIS` (see [Step Contract §2.3](./step_contract.md) and §2 below)  
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))  
**Version:** 2.0.0  
**Status:** Draft  
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 18 performs an AI-powered debugging analysis of source code in the project. It
discovers source files across multiple supported languages, selects the most relevant
AI debugging persona based on heuristic code-pattern detection, and produces a targeted
analysis report covering observer patterns, async flow issues, or data-structure problems
depending on which patterns dominate the code.

Unlike steps that respond to recent changes, Step 18 scans the full source tree. It is
designed to surface latent structural issues rather than review incremental modifications.

---

## 2. Step Kind Conformance

Step 18 declares the **ANALYSIS** step kind, as defined in `step_contract.md §2.3`.

| Property | Value |
|---|---|
| Kind identifier | `"analysis"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **No** — runs on all projects regardless of kind or language |
| AI-integrated | **Yes** — calls the Copilot API via `aiHelper` and `aiCache` |
| Estimated duration | Variable; depends on file count and AI availability |

**⚠ Deviation from step contract — kind declaration location.** The ANALYSIS kind is
declared in the exported `STEP_DEFINITION` metadata object, not as a static property on
the step class itself. The step class carries no `stepKind` declaration. This differs
from the PROJECT and CONTEXT conventions, which require a static class-level property.
The `ANALYSIS` constant is not yet defined in `step_contract.js` — it is a known
specification gap (see `step_contract.md §2.3`). Until the contract is updated,
orchestrator implementations that dispatch ANALYSIS steps must read the kind from
`STEP_DEFINITION.kind` rather than from the class.

Step 18 conforms fully to the step contract error-handling rule: all exceptions are
caught inside `execute` and surfaced as `{ success: false, error: … }`. No exception
propagates to the caller.

Step 18 conforms to the [AI Prompt Contract](./ai_prompt_contract.md): file contents
are injected using `buildFileContentBlock`, per-file and total character budgets are
enforced, each file read is wrapped individually, and no IDE-specific prompt techniques
are used.

---

## 3. Domain Concepts

### 3.1 Source File Discovery

Step 18 discovers source files by scanning the project root for files with recognised
extensions. The supported extensions are:

| Language family | Extensions |
|---|---|
| JavaScript / TypeScript | `.js`, `.ts` |
| Python | `.py` |
| Java | `.java` |
| Go | `.go` |

The following directories are excluded from scanning: `node_modules`, `.git`, `dist`,
`build`, `coverage`, `test`, `__tests__`.

The result is de-duplicated. The total list is capped at **100 files**. When
`options.sourceFiles` is provided, discovery is bypassed entirely.

### 3.2 Sample Window

Persona detection and AI analysis operate on a **sample** of up to 20 files taken from
the front of the discovered (or supplied) source file list. All three personas receive
only this sample. The full file count is reported in the result but the AI never
receives more than 20 files.

### 3.3 Debug Personas

Three debugging personas are available, each specialised for a different class of code
problem. Personas are identified by the YAML key used to load their configuration from
the shared AI helpers file:

| Persona key | Focus area |
|---|---|
| `observer_pattern_debugger_prompt` | Observer and event-listener patterns; pub/sub wiring |
| `async_flow_debugger_prompt` | Async/await, Promise chains, callback flow |
| `data_structure_debugger_prompt` | Map, Set, linked lists, trees, heaps, graphs |

### 3.4 Persona Selection Heuristic

Exactly one persona is selected per execution. The selection is based on counting
pattern occurrences in the combined text of the sample file contents (case-insensitive):

| Score | Patterns counted |
|---|---|
| Observer score | `EventEmitter`, `addEventListener`, `.subscribe(`, `.on(` |
| Async score | `async`, `await`, `new Promise`, `.then(`, `callback` |
| Data-structure score | `new Map`, `new Set`, `LinkedList`, `BinaryTree`, `heap`, `graph` |

Selection rules (evaluated in order):

1. If the observer score is greater than or equal to both the async score and the
   data-structure score, select `observer_pattern_debugger_prompt`.
2. If the data-structure score is greater than the async score, select
   `data_structure_debugger_prompt`.
3. Otherwise (including when all scores are zero), select `async_flow_debugger_prompt`
   as the default.

A caller may override this heuristic by supplying `options.forcedPersona`.

### 3.5 Prompt Assembly

The debugging persona templates in the shared AI helpers configuration follow a
non-standard structure (`role_prefix`, `specific_expertise`, `approach`,
`output_format`) rather than the standard `role` / `task_template` structure used by
most other steps. Step 18 handles both:

**Primary path (structured template):** When the persona configuration is a valid
object with the expected fields, the step assembles the prompt manually by joining
the following sections:

1. Optional project-kind role override, prefixed as `[Project-Kind Role: …]`.
2. Role description (`role_prefix` or `role` field).
3. Specific expertise statement.
4. Source file list with total count.
5. File content blocks (injected per §3.6).
6. Approach guidance.
7. Output format specification.

**Fallback path (generic builder):** When the configuration does not match the
expected structure, the generic YAML step prompt builder is used with placeholders
for project name, source files, and file count. File content blocks are appended
after the built prompt.

### 3.6 File Content Injection

Step 18 injects actual source file content into the prompt, conforming to the
[AI Prompt Contract §3](./ai_prompt_contract.md):

- Each file's content is read individually; unreadable files yield an empty string
  (no error is raised for individual failures).
- Each non-empty content is wrapped with `buildFileContentBlock`.
- The per-file budget is `MAX_CHARS_PER_FILE` (4,000 characters); content is
  truncated internally by the block builder.
- Files are added until the cumulative total reaches `MAX_CHARS_TOTAL_CONTENTS`
  (30,000 characters); no further files are added once the budget is exhausted.

### 3.7 Project-Kind Role Overlay

Step 18 optionally loads a `debugging_specialist` role from the project-kind
configuration. When present, the role is prepended to the prompt as a context
declaration. This overlay is purely additive; its absence does not affect the rest of
the prompt.

### 3.8 Cache Key

The AI response cache key encodes the step identity, persona, project root, and total
source file count:

```
"step_18|{personaKey}|{projectRoot}|{sourceFiles.length}"
```

A change in the project root path, source file count, or selected persona produces a
fresh AI request.

---

## 4. Constructor Dependencies

Step 18 accepts all collaborators at construction time via a single options map. All
have default fallbacks.

| Dependency key | Role | When absent |
|---|---|---|
| `fileOps` | Reads file contents and runs glob patterns for source discovery | A default FileOperations instance is created |
| `backlog` | Writes the debugging report to the workflow artifact store | A default Backlog instance is created |
| `aiHelper` | Executes AI API requests using the specified persona | A default AiHelper instance is created (with optional `promptsDir`) |
| `aiCache` | Caches and retrieves AI responses by cache key | A default AiCache instance is created |
| `promptsDir` | Path to the prompts directory, forwarded to the default AiHelper | Defaults to `null` |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the project root |
| `options.sourceFiles` | Override list of source file paths (relative). When present, skips discovery (§3.1). |
| `options.forcedPersona` | Override persona key. When present, skips the heuristic (§3.4). |
| `options.projectKind` | Project kind identifier, used for the optional role overlay (§3.7). |

### 5.2 Execution Phases

Step 18 executes the following operations in order:

#### Phase 1 — Discover Source Files

When `options.sourceFiles` is provided, use it directly. Otherwise, scan the project
root using the patterns defined in §3.1 to produce a de-duplicated list of up to 100
source file paths. Log the count of files discovered.

#### Phase 2 — Sample and Read Contents

Take the first 20 files from the source list (the sample window, §3.2). Read each
sample file's content from disk. Unreadable files produce an empty string; they are
not treated as errors.

#### Phase 3 — Select Debug Persona

When `options.forcedPersona` is provided, use it as the persona key. Otherwise, apply
the heuristic (§3.4) to the array of sample file contents to select a persona key.
Log the selected persona.

#### Phase 4 — Initialise AI

Attempt to initialise the AI helper. If unavailable, log a warning. Phases 5 and 6
are skipped when AI is not available; execution continues from Phase 7.

#### Phase 5 — Load Prompt Configuration

Load the shared AI helpers YAML configuration file. Attempt to load the project-kind
YAML to obtain a `debugging_specialist` role overlay (§3.7); silently ignore failures.

#### Phase 6 — Build File Content Blocks

Iterate over the sample files, respecting the per-file and total character budgets
(§3.6). Build a labelled content block for each non-empty file. Accumulate into a
single file contents section.

#### Phase 7 — Assemble Prompt and Request AI

Assemble the prompt using the primary or fallback path (§3.5). When a prompt was
successfully assembled, execute the AI request through the cache using the cache key
(§3.8). The AI request uses the `code_quality_analyst` persona and, when supported,
the `claude-haiku-4.5` model.

Extract the response content string from the AI result. When the AI request or prompt
assembly fails, log a warning and continue with an empty content string.

#### Phase 8 — Format Report

Produce a Markdown report (§6.3) from the selected persona key, the list of sample
files analysed, and the AI content. The report is produced regardless of whether AI
content was obtained.

#### Phase 9 — Save to Backlog

Save the report to the workflow backlog under step `18`, section
`'Debugging_Analysis'`.

#### Phase 10 — Return Result

Return the step result (§6.2). Log whether AI content was produced.

### 5.3 Skip Conditions

Step 18 has **no skip conditions**. It runs for all projects and all change sets. When
no source files are found, the step runs with an empty sample and produces a minimal
report.

### 5.4 Error Handling

All exceptions raised during any phase are caught at the outer level. On any unhandled
error, Step 18 returns:

```
{ success: false, error: <exception message> }
```

AI-level failures (prompt load errors, API errors) are caught at the inner level,
logged as warnings, and produce an empty AI content string. These do **not** cause
the step to return failure.

---

## 6. Data Shapes

### 6.1 STEP_DEFINITION (exported metadata)

| Field | Value | Description |
|---|---|---|
| `id` | `'step_18'` | Step identifier |
| `name` | `'Debugging Analysis'` | Human-readable name |
| `kind` | `STEP_KIND.ANALYSIS` | Step kind (see §2 deviation note) |
| `description` | `'AI-powered debugging analysis using observer, async, and data-structure personas'` | — |
| `dependencies` | `[]` | No preceding steps required |

### 6.2 StepResult (success)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `personaKey` | String | The persona key that was used |
| `filesAnalyzed` | String[] | The sample file paths passed to the AI |
| `totalSourceFiles` | Integer | Total discovered source file count (before sampling) |
| `aiContent` | String | AI analysis text; empty string when AI was unavailable or prompt missing |
| `report` | String | Formatted Markdown report (§6.3) |

### 6.3 Debugging Report (Markdown)

The report produced and saved to the backlog has the following structure:

```
# Step 18: Debugging Analysis — {Persona Label}

## Files Analyzed
- {file path}
…

## AI Analysis

{AI content, or placeholder when unavailable}
```

Persona labels are:

| Persona key | Label |
|---|---|
| `observer_pattern_debugger_prompt` | `Observer Pattern Debugger` |
| `async_flow_debugger_prompt` | `Async Flow Debugger` |
| `data_structure_debugger_prompt` | `Data Structure Debugger` |
| Any other value | `Debugging Analysis` |

### 6.4 StepResult (failure)

| Field | Type | Description |
|---|---|---|
| `success` | `false` | Fatal error occurred |
| `error` | String | Exception message |

---

## 7. Constraints and Rules

1. **Step kind declaration deviation.** Step 18 does **not** declare `static stepKind`
   on the class. The ANALYSIS kind is declared only in the exported `STEP_DEFINITION`
   object. Orchestrator implementations dispatching Step 18 must read the kind from
   `STEP_DEFINITION.kind`. This is a known deviation from the PROJECT and CONTEXT
   conventions and from `step_contract.md §5` rule 1. Resolution requires either
   updating the step to add a static class property, or updating the contract to
   formally define where ANALYSIS steps declare their kind.

2. The sample window **must** be capped at 20 files. The AI must not receive more
   than 20 source files regardless of the total discovered count.

3. The source file discovery result **must** be de-duplicated and capped at 100
   files. The cap is applied before sampling.

4. File content injection **must** comply with the [AI Prompt Contract §3](./ai_prompt_contract.md):
   each file read wrapped individually, `buildFileContentBlock` used for each non-empty
   file, per-file budget enforced by the block builder, total budget (`MAX_CHARS_TOTAL_CONTENTS`)
   enforced by the caller.

5. When `options.forcedPersona` is supplied, the heuristic **must not** run. The forced
   value is used as-is.

6. When the AI helper is unavailable, Step 18 **must** still produce and save a report.
   AI unavailability is not a failure condition.

7. Inner AI errors (prompt load failure, API error) **must** be caught and logged as
   warnings. They **must not** propagate to the outer error handler or cause the step to
   return `success: false`.

8. The `aiCache.init()` call **must** be made before the first `withCache` call, as
   required by the [AI Prompt Contract §6.2](./ai_prompt_contract.md).
