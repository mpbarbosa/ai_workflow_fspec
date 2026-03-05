# Step 03: Script References — Functional Specification

**Step identifier:** `step_03`  
**Step name:** Script Reference Validation  
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))  
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))  
**Version:** 2.0.0  
**Status:** Draft  
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 03 validates that a project's documentation accurately describes its scripts. It
answers three questions:

1. **Are documented scripts present on disk?** — detecting references to non-existent
   scripts.
2. **Do scripts have correct execution permissions?** — flagging scripts that cannot be
   run directly.
3. **Are all discovered scripts documented?** — identifying scripts that have no
   documentation coverage.

The step reports findings as a structured Markdown summary and saves it to the workflow
backlog. When an AI helper is available, the `devops_engineer` persona is invoked to
recommend corrective actions based on the coverage gaps and issue counts.

---

## 2. Step Kind Conformance

Step 03 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **Yes** — skips when no script files are found |
| AI-integrated | **Yes** — calls the AI API via `aiHelper` and `aiCache` when the helper initialises successfully |
| AI persona | `devops_engineer` |

**Error handling deviation.** Step 03 re-throws unhandled exceptions from `execute`
rather than returning `{ success: false, error }`. This deviates from the step contract
rule that forbids propagating exceptions (`step_contract.md §5`, rule 4). Callers must
be prepared to catch.

---

## 3. Domain Concepts

### 3.1 Script Language Detection

The step begins by detecting the project's primary language via the tech-stack detector.
This determines which file extensions and directories are searched. When detection fails,
the step defaults to `bash`. Supported languages and their file extensions:

| Language | Extensions |
|---|---|
| bash | `.sh` |
| python | `.py` |
| javascript | `.js`, `.mjs` |
| typescript | `.ts`, `.mts` |
| go | `.go` |
| java | `.java` |
| ruby | `.rb` |
| rust | `.rs` |
| cpp | `.cpp`, `.cc`, `.h`, `.hpp` |

### 3.2 Search Directories by Language

Script discovery is scoped to a set of directories that vary by language:

| Language | Directories searched |
|---|---|
| bash | `.` (project root), `scripts/`, `src/scripts/`, `src/workflow/` |
| python | `scripts/`, `src/` |
| javascript | `scripts/`, `src/` |
| typescript | `scripts/`, `src/` |
| All others (default) | `scripts/` |

The following directories are always excluded from all discovery searches:
`node_modules`, `.git`, `dist`, `build`, `coverage`.

### 3.3 Script Reference Extraction

Script references are extracted from documentation (README and additional doc files)
using two patterns:

- **Inline code spans:** backtick-wrapped paths ending in a recognised script extension
  (e.g. `` `./scripts/deploy.sh` ``).
- **Fenced code blocks:** blocks with a shell or script language marker (`bash`, `sh`,
  `python`, `javascript`, `typescript`); script-like tokens within the block body are
  extracted, excluding shell substitution expressions.

Duplicate references are removed. Only references whose extension matches the detected
language's file patterns are retained for validation, to prevent cross-language false
positives.

### 3.4 Missing Reference Validation

A reference is considered missing when its normalised form (leading `./` stripped) does
not match any path in the set of scripts discovered on disk.

### 3.5 Executable Permission Check

For each script discovered on disk, the file system mode is inspected. A script is
non-executable when none of the user, group, or owner execute bits are set. This check
is meaningful only on Unix-like operating systems; on other platforms, the check silently
returns an empty result.

### 3.6 Documentation Coverage

A script is considered documented when its filename (basename) or its full path appears
in the README or in any of the additionally loaded documentation files. Coverage is
checked across all loaded documentation sources — not only the README.

### 3.7 Prompt Language Disambiguation

When the detected language is `bash` (the fallback), the step performs a secondary
tech-stack detection pass to identify the project's true primary language. This value is
used in the AI prompt to give the model accurate project context, even when only shell
scripts are being validated.

### 3.8 AI Prompt Construction

Two prompt-building strategies are attempted in order:

1. **YAML template (preferred):** Load the `step3_script_refs_prompt` template from the
   shared AI helpers YAML configuration. Populate it with: project name, project
   description, primary language (from §3.7), script directories, script count, change
   scope, issue count, a script issue summary, the list of all discovered script paths,
   a per-script documentation coverage map, and an excerpt of each loaded documentation
   file (§3.10).

2. **Built-in fallback:** When the YAML template cannot be loaded, construct a structured
   prompt with role, task, and approach sections, populated with: script count, script
   names, missing reference count, non-executable count, undocumented count, and total
   issue count.

Both strategies use the `devops_engineer` persona.

### 3.9 Cache Key

The AI response is cached with the key:

```
step_03|v3|<scriptsFound>|<totalIssues>|<docHash>
```

`<docHash>` is derived from the first 100 characters of each documentation file's
content, combined. This ensures the cache is invalidated when documentation content
changes between runs, even when issue counts remain the same.

### 3.10 Documentation Context Budget

When building the YAML-template prompt, up to 80 lines are included per documentation
file. The combined documentation context across all files is capped at 2,000 characters;
content beyond this limit is truncated with a `"... [truncated]"` marker.

---

## 4. Constructor Dependencies

Step 03 receives all collaborators via a single options map at construction time. All
dependencies are always instantiated — there are no opt-in dependencies.

| Dependency key | Role | Behaviour when absent |
|---|---|---|
| `fileOps` | File discovery, reading, permission checks, and glob matching | Defaults to a new `FileOperations` instance |
| `backlog` | Writes the validation report to the workflow artifact store | Defaults to a new `Backlog` instance |
| `techStack` | Detects the project's primary language and tech stack | Defaults to a new `TechStackDetector` instance |
| `aiHelper` | Executes AI API requests | Defaults to a new `AiHelper` instance |
| `aiCache` | Caches and retrieves AI responses | Defaults to a new `AiCache` instance |
| `promptsDir` | Directory from which `aiHelper` loads custom prompt files | `null`; `aiHelper` uses its own default |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the project root |
| `options.language` | Override for the detected primary language |
| `options.projectDescription` | Optional plain-text description injected into the AI prompt |
| `options.scope` | Change scope identifier forwarded to the AI prompt template |

### 5.2 Execution Phases

Step 03 runs seven sequential validation phases followed by a conditional AI phase:

#### Phase 1 — Language and Pattern Detection

Detect the primary language from `options.language` or via the tech-stack detector (§3.1).
Derive file extension patterns and search directories (§3.2). When the detected language
is `bash`, perform the prompt-language disambiguation pass (§3.7) via a secondary
tech-stack detection query.

#### Phase 2 — Script Discovery

Search the resolved directories using the language's file extension patterns. Produce a
de-duplicated list of relative script paths. When no scripts are found, skip immediately:

```
{ success: true, skipped: true, reason: "no_scripts" }
```

#### Phase 3 — Documentation Loading

Load the README file, trying common capitalisation variants
(`README.md`, `README.MD`, `readme.md`, `Readme.md`). Load up to four additional
documentation files from a fixed priority list: `docs/API.md`,
`docs/ARCHITECTURE.md`, `docs/GETTING_STARTED.md`, `docs/CONTRIBUTING.md`,
`CONTRIBUTING.md`. Files are included only when they are readable and non-empty. The
README and all additional files are combined into a unified documentation source list
for subsequent phases.

#### Phase 4 — Reference Extraction and Validation

Extract script references from the README content (§3.3). Filter the extracted
references to retain only those whose extension matches the detected language's patterns
(§3.3). Compare normalised references against the discovered script set (§3.4). Produce
a list of missing-reference issue records.

#### Phase 5 — Executable Permission Checks

For each discovered script, inspect the file system mode (§3.5). Produce a list of paths
for scripts that lack execute permission.

#### Phase 6 — Documentation Coverage Check

For each discovered script, check whether it is mentioned (by basename or path) in the
README or in any additionally loaded documentation file (§3.6). Produce a list of paths
for undocumented scripts.

#### Phase 7 — Report Generation and Persistence

Aggregate counts: `scriptsFound`, `referencesChecked`, `totalIssues`,
`missingReferences`, `nonExecutable`, `undocumented`. Format a Markdown report (§6.3)
and write it to the workflow backlog under step `3`, section
`"Script Reference Validation"`.

#### Phase AI — AI-Powered Recommendations (Conditional)

Runs when the AI helper successfully initialises. The AI phase does not affect any field
in the step result.

1. Initialise `aiCache`.
2. Build the per-script documentation coverage map across all loaded documentation files
   (§3.6).
3. Attempt to build the prompt from the YAML template (§3.8); fall back to the built-in
   prompt when the template is unavailable.
4. Compute the cache key (§3.9).
5. Execute the prompt via `aiCache.withCache` using the `devops_engineer` persona.
6. When a non-empty response is received, append an `"AI Recommendations"` section to
   the report and overwrite the backlog entry with the enriched report.

When the AI helper is unavailable (initialisation fails), the AI phase is bypassed with
a warning. The step result is returned with the non-AI report.

### 5.3 Skip Conditions

| Reason | Condition |
|---|---|
| `no_scripts` | No script files match the language's patterns in any search directory |

The skip result is `{ success: true, skipped: true, reason: "no_scripts" }`.

---

## 6. Data Shapes

### 6.1 Missing Reference Issue

| Field | Type | Description |
|---|---|---|
| `reference` | String | The original reference string as found in the documentation |
| `normalized` | String | The reference with any leading `./` removed |
| `type` | String | Always `"missing_reference"` |

### 6.2 StepResult (success, not skipped)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without error |
| `scriptsFound` | Integer | Total number of script files discovered on disk |
| `referencesChecked` | Integer | Number of language-filtered documentation references validated |
| `totalIssues` | Integer | Sum of `missingReferences.length + nonExecutable.length + undocumented.length` |
| `missingReferences` | Object[] | Missing reference issue records (§6.1) |
| `nonExecutable` | String[] | Paths of scripts that lack execute permission |
| `undocumented` | String[] | Paths of scripts not mentioned in any documentation file |

### 6.3 Backlog Report Format

The Markdown report contains:

**Summary section:** Scripts found, references checked, total issues, missing reference
count, non-executable count, undocumented count. A status line shows either
`✅ All script references valid` or `⚠️ Issues found — review required`.

**Missing References section:** Present when `missingReferences.length > 0`. Lists up to
10 entries with original and normalised paths. A count of additional entries is shown
when there are more than 10.

**Non-Executable Scripts section:** Present when `nonExecutable.length > 0`. Lists up to
10 paths. A count of additional entries is shown when there are more than 10.

**Undocumented Scripts section:** Present when `undocumented.length > 0`. Lists up to 10
paths. A count of additional entries is shown when there are more than 10.

**AI Recommendations section:** Appended when the AI phase ran and produced a non-empty
response.

### 6.4 StepResult (skipped)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `reason` | String | `"no_scripts"` |

---

## 7. Constraints and Rules

1. Language detection **must** occur before file discovery. File extension patterns and
   search directories are language-dependent; using incorrect patterns yields wrong
   results.

2. References extracted from documentation **must** be filtered to match the detected
   language's extensions before validation (§3.3). Validating cross-language references
   produces false missing-reference issues.

3. Excluded directories (`node_modules`, `.git`, `dist`, `build`, `coverage`) **must**
   never appear in the discovered script list.

4. Documentation loading **must** handle unreadable files gracefully. A missing README
   or missing additional doc file must not abort the step; execution proceeds with
   whatever content is available.

5. The AI prompt **must** include documentation file excerpts (§3.10) to ground the
   model's recommendations in actual content. Documentation references without injected
   content allow the model to fabricate issues — a violation of the
   [AI Prompt Contract §3](./ai_prompt_contract.md).

6. The cache key **must** include a content-derived hash component (§3.9). A cache key
   based solely on issue counts allows stale recommendations to be served when file
   contents change between runs.

7. The AI phase is advisory. AI output **must not** change the values of `success`,
   `totalIssues`, `missingReferences`, `nonExecutable`, or `undocumented` in the step
   result. AI output affects only the backlog report.

8. **Error propagation deviation:** Step 03 re-throws unhandled exceptions from `execute`
   rather than returning `{ success: false, error }`. This deviates from the step
   contract (§5, rule 4). Callers must wrap the call in error handling.

9. The executable permission check (Phase 5) is meaningful only on Unix-like file
   systems. On platforms that do not expose execute permission bits via file mode, the
   check silently returns an empty result and must not be treated as a failure.

10. The `devops_engineer` persona is used for all AI requests in this step. Changing the
    persona requires bumping the cache key version identifier to avoid serving cached
    responses built for a different persona.
