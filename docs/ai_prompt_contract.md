# AI Prompt Contract — Functional Specification

**Module:** `steps/ai_prompt_builder`  
**Version:** 1.0.0  
**Status:** Draft  
**Related:** [`step_contract.md`](./step_contract.md)

---

## 1. Purpose

This specification defines the **AI prompt contract** — the rules that every workflow step
must satisfy when it calls the Copilot AI API to generate, analyse, or review content.

The rules in this document exist because violations produce **hallucinated responses**: the
model fabricates file contents, suggests edits to the wrong project, or references
non-existent files. The hallucination happens silently — the workflow completes without
errors, but the AI output is entirely wrong.

### Root-Cause Event

During a `olinda_shell_interface.js` workflow run (2026-03-05), `step_01` produced four
completely fabricated documentation edits. The model described the project as a "Bash
shell-based AWS LBS management tool" and suggested replacing real TypeScript API docs with
content from an unrelated project (`ai_workflow.js`). Investigation revealed:

1. The `doc_analysis_prompt` template instructed the model to *"Use `@workspace` to examine
   what was modified"* — a VS Code Chat extension feature that is silently ignored when
   called via the Copilot SDK API.
2. With no actual file content in the prompt, the model fell back to training-data
   assumptions and system-prompt context bleed-through, producing output appropriate for the
   wrong repository.

This specification codifies the rules that prevent this class of failure.

---

## 2. Scope

This contract applies to every workflow step that calls the Copilot AI API — directly or
via `aiHelper` / `aiCache`. The following steps are AI-integrated:

| Step | Primary Persona | Notes |
|------|----------------|-------|
| `step_01` | `documentation_expert` | File content injection required (fixed 2026-03) |
| `step_02` | `documentation_expert` | Per-consistency-rule checks |
| `step_03` | `devops_engineer` | Shell/script documentation review |
| `step_04` | `devops_engineer`, `code_quality_analyst` | Config + test-quality review |
| `step_05` | `devops_engineer` | Dependency analysis |
| `step_09` | various | UX / accessibility |
| `step_0b` | various | Bootstrap documentation |
| `step_10` | `code_quality_analyst` | Code quality (has anti-hallucination guard) |
| `step_11_5` | various | Context management |
| `step_11_6` | various | Context management |
| `step_12` | `git_specialist` | Commit message generation |
| `step_13` | `technical_writer` | Markdown lint review |
| `step_14` | `prompt_engineer` | Prompt engineering |
| `step_15` | various | UX analysis |
| `step_16` | `devops_engineer` | Version bump |
| `step_18` | various | Debugging analysis |
| `step_19` | typescript_developer (`Strider`) | TypeScript review (reference implementation) |

Steps not in this list (e.g., `step_00`, `step_06`, `step_07`, `step_08`, `step_0f`,
`step_11`, `step_17`) do not call the AI API and are exempt from this contract.

---

## 3. File Content Injection

### 3.1 Requirement

Every AI-integrated step that references specific files in its prompt **MUST** inject the
actual contents of those files into the prompt. The model must not be expected to retrieve
file contents through IDE tooling, infer them from filenames, or recall them from training
data.

### 3.2 Standard Pattern

Use the utilities exported from `src/lib/ai_prompt_builder.js`:

```js
import {
  buildFileContentBlock,
  MAX_CHARS_TOTAL_CONTENTS,
} from '../lib/ai_prompt_builder.js';
```

| Constant / Function | Value / Signature | Description |
|---|---|---|
| `MAX_CHARS_PER_FILE` | `4000` | Per-file character budget (applied internally by `buildFileContentBlock`) |
| `MAX_CHARS_TOTAL_CONTENTS` | `30000` | Total character budget for all file contents in one prompt |
| `buildFileContentBlock(filePath, content)` | `→ string` | Returns a fenced block labelled with the file path; truncates `content` to the per-file limit |

### 3.3 Reference Implementation

`src/steps/step_19_typescript_review.js` is the canonical example:

```js
// Collect file contents, respecting total budget
let fileContents = '';
let totalChars = 0;
for (const filePath of filesToReview) {
  try {
    const content = await fs.readFile(filePath, 'utf8');
    fileContents += buildFileContentBlock(filePath, content);
    totalChars += content.length;
    if (totalChars >= MAX_CHARS_TOTAL_CONTENTS) break;
  } catch {
    // Skip unreadable files gracefully
  }
}

// Pass as a prompt context variable
const prompt = buildYamlStepPrompt(config, {
  file_contents: fileContents,
  // ...other variables
});
```

Steps `step_01` (fixed 2026-03) and `step_18` also follow this pattern.

### 3.4 Error Handling

- Each file read **MUST** be wrapped in its own `try/catch`.
- An unreadable file **MUST** be skipped silently; it must not abort the prompt build.
- If no files are readable, the step **MUST** still proceed with an empty `file_contents`
  string and surface a warning in its `StepResult.summary`.

### 3.5 Budget Enforcement

```
totalChars = 0
for each file:
    if totalChars >= MAX_CHARS_TOTAL_CONTENTS → stop
    read file → content
    fileContents += buildFileContentBlock(path, content)
    totalChars += content.length
```

The per-file truncation is applied inside `buildFileContentBlock`; callers do not need to
slice content before passing it. However, callers **MUST** check the cumulative total and
stop adding files once the total budget is reached.

---

## 4. Prohibited Prompt Techniques

The following techniques **MUST NOT** appear in any prompt template shipped in
`.workflow_core/config/ai_helpers.yaml` or constructed at runtime by a step:

| Technique | Reason |
|---|---|
| `@workspace` | VS Code Chat extension feature; silently ignored when called via the Copilot SDK API. The model falls back to training data, causing hallucination. |
| `@file`, `#file`, `#editor`, `#selection` | Same category — VS Code Chat variables; not available via API. |
| "Read the file yourself" / "Examine the repository" | Instructs the model to perform an action it cannot take; leads to fabrication. |
| Referencing a file by name without providing its content | The model will invent plausible-sounding content based on the filename. |

**Enforcement:** YAML prompt templates are reviewed before merging. Any template that
delegates file retrieval to the model rather than injecting content is non-conforming.

---

## 5. Anti-Hallucination Requirements

### 5.1 When Required

Steps that ask the model to analyse or edit **project-specific files** SHOULD include an
explicit anti-hallucination instruction in the prompt. This is especially important when:

- The prompt provides a list of file paths without their contents.
- The prompt references a project name, framework, or technology stack.

### 5.2 Recommended Guard Text

```
Do not reference or assume content for any file not shown above.
Do not reference external projects, repositories, or tools not present in this context.
If a file's content is not provided, state that you cannot analyse it without seeing its contents.
```

### 5.3 Existing Compliant Examples

`step_10`'s prompt template already includes:

> "Do not reference ai_workflow.js, node_modules, or any other project not present in the
> context."

This is the correct approach for steps that are aware of common context-bleed sources.

---

## 6. Dependency Requirements

### 6.1 `aiHelper`

Every AI-integrated step uses `aiHelper` to execute API requests. The dependency is
injected via the constructor.

```js
constructor(deps = {}) {
  this.aiHelper = deps.aiHelper;
}
```

`aiHelper` must expose:
- `executeRequest(prompt, options?) → Promise<string>` — executes a single AI request and
  returns the model response as a string.

### 6.2 `aiCache`

Steps that cache AI responses use `aiCache`. The dependency is injected via the
constructor.

```js
constructor(deps = {}) {
  this.aiCache = deps.aiCache;
}
```

`aiCache` must expose:
- `init() → Promise<void>` — initialises the cache store. **Must be called before the
  first `withCache` call.** Failure to call `init` causes a `TypeError` at runtime.
- `withCache(prompt, cacheKey, fn) → Promise<string>` — returns a cached response for
  `cacheKey` if available; otherwise calls `fn()` to obtain a fresh response and stores it.

### 6.3 Mock Requirements for Tests

When writing unit tests for AI-integrated steps, the `aiCache` mock **must** implement
both `init` and `withCache`:

```js
const mockAiCache = {
  init: () => Promise.resolve(),
  withCache: (_prompt, _key, fn) => fn(),
};
```

Omitting `init` causes `TypeError: this.aiCache.init is not a function` even if the test
never reaches a `withCache` call, because steps call `init` in their setup phase.

---

## 7. Reference Implementation

`src/steps/step_19_typescript_review.js` is the canonical conforming implementation. It:

- Reads all TypeScript files within the budget (§3.3).
- Uses `buildFileContentBlock` for each file (§3.2).
- Passes `file_contents` as a named prompt variable (§3.3).
- Wraps each file read in a `try/catch` (§3.4).
- Stops adding files when the total budget is reached (§3.5).
- Does not reference `@workspace` or any IDE feature (§4).

`src/steps/step_01_documentation.js` (after the 2026-03 fix) and `src/steps/step_18_debugging.js`
also conform to this contract.

---

## 8. Rationale

### Why this happened

The Copilot SDK API does not expose VS Code Chat extension variables. When a prompt
template uses `@workspace`, the API receives a plain-text reference to an undefined
variable. The model interprets this as an instruction to examine the project autonomously —
an action it cannot perform — and falls back to fabricating content based on:

1. Its training-data prior for the apparent project type.
2. System-prompt context that may include instructions or metadata from a different
   project (e.g., a `copilot-instructions.md` file injected by the orchestrator).

The result is silent, confident, and completely wrong output.

### Why file injection works

When `buildFileContentBlock` injects actual file content into the prompt, the model is
grounded in real data. It can only describe, analyse, or edit what is shown to it. The
total budget (30,000 chars) is large enough to include the most relevant files for any
single step without exceeding typical context-window limits.

### Why a separate spec document

The `step_contract.md` governs step structure and dispatch. The rules in this document
govern a specific cross-cutting concern — AI prompt hygiene — that spans many steps and
depends on shared utilities. Keeping the two concerns separate makes each document easier
to maintain and cite independently.
