# Step 15: UX Analysis — Functional Specification

**Step identifier:** `step_15`
**Step name:** UX Analysis
**Step kind:** `CONTEXT` (see [Step Contract §2.2](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 15 discovers UI source files in the project, reads representative samples for AI
grounding, and submits them to the `ux_analyst` AI persona for a structured
accessibility, usability, visual design, and component architecture review.

Step 15 answers two questions:

1. **Are there UI components to review?** — by probing the filesystem for recognised
   UI file extensions.
2. **What UX issues exist?** — by injecting sampled file content into the AI and
   parsing the structured Markdown response for critical issues, warnings, and
   improvement suggestions.

The step is conditional: it runs only on projects with recognised UI project types or,
as a fallback, on any project where UI files are actually found on disk regardless of
the declared project type.

---

## 2. Step Kind Conformance

Step 15 conforms to the **CONTEXT** step kind as defined in `step_contract.md §2.2`.

| Property | Value |
|---|---|
| Kind identifier | `"context"` |
| Execute signature | `execute(context) → Promise<StepResult>` |
| Can be skipped | **Yes** — multiple conditions produce a graceful skip |
| AI-integrated | **Yes** — calls the AI API via `aiHelper`; does **not** use `aiCache` |
| Conforms to AI Prompt Contract | **Yes** — file content injection is applied (§5 Phase 6) |

**No caching.** Unlike most AI-integrated steps, Step 15 does not use `aiCache`.
Each run always performs a live AI request. There is no cache key computation and no
`withCache` call.

**Error handling compliance.** Step 15 catches all unhandled exceptions and returns
`{ success: false, error: <message> }`. This is fully compliant with the step contract.

---

## 3. Domain Concepts

### 3.1 UI Project Types

Step 15 runs unconditionally on projects whose declared project kind matches one of the
following eligible types:

| Identifier | Description |
|---|---|
| `react_spa` | React single-page application |
| `vue_spa` | Vue single-page application |
| `client_spa` | Generic client-side SPA |
| `static_website` | Static HTML/CSS website |
| `web_application` | General web application |
| `documentation_site` | Documentation website |

Hyphens in the incoming project type are normalised to underscores before matching.

### 3.2 UI File Detection Fallback

When the project type is not in the eligible list, the step does not skip immediately.
Instead, it performs a filesystem probe: it scans the project directory for files
matching any recognised UI extension (§3.3). If UI files are found, the step proceeds
as if the project type were eligible. This allows non-UI project types that nonetheless
contain a Vue or React sub-project to receive UX analysis.

If the probe also finds no UI files, the step skips with reason
`'project type not eligible'`.

### 3.3 UI File Extensions

Files are classified as UI files based on their extension. Excluded directories
(§3.4) are never included in the file list.

| Category | Extensions |
|---|---|
| React components | `.jsx`, `.tsx` |
| Vue components | `.vue` |
| HTML documents | `.html` |
| Stylesheets | `.css`, `.scss`, `.sass`, `.less` |
| Svelte components | `.svelte` |

### 3.4 Excluded Directories

The following directory names are excluded from all file scans:

`node_modules`, `.git`, `dist`, `build`, `coverage`, `.next`, `out`, `venv`, `.venv`,
`__pycache__`, `vendor`, `.cache`, `tmp`

A file is excluded if its path contains any excluded directory name as a path segment,
or if the path begins with that segment followed by a separator.

### 3.5 Key File Selection

To stay within the AI prompt budget, only a subset of discovered UI files is read for
content injection. The selection algorithm prioritises files with the highest
accessibility and usability signal:

- **HTML files** fill approximately 70% of the available slots (rounded down).
- **CSS files** fill the remaining slots, receiving any unused HTML slots as a bonus.
- A maximum of 10 files is selected in total.

The selection is done from the full discovered UI file list, in discovery order, within
each category's slot budget.

### 3.6 UX Issue Categories and Severities

The AI produces a structured Markdown response. Issues are classified by:

**Categories:** `accessibility`, `usability`, `visual`, `performance`,
`component-architecture`

**Severities:** `critical`, `warning`, `suggestion`

Issue counts are extracted from the AI response by pattern matching:
- `critical` — occurrences of `**Severity**: Critical` in the response
- `warning` — occurrences of `**Severity**: Warning` in the response
- `suggestion` — sub-headings (`###`) under the `## Improvement Suggestions` section

---

## 4. Constructor Dependencies

Step 15 receives all collaborators at construction time via a single options map.

| Dependency key | Role | When absent |
|---|---|---|
| `logger` | Logging — must use `this.logger` per CONTEXT contract | Falls back to a new `Logger` instance |
| `fileOps` | File system access: glob patterns, file reads | Falls back to a new `FileOperations` instance |
| `backlog` | Writes step summary to the workflow artifact store | Falls back to a new `Backlog` instance |
| `aiHelper` | Executes AI API requests (see AI Prompt Contract §6.1) | Falls back to a new `AiHelper` instance |
| `dryRun` | Boolean; when `true`, skips all I/O and returns immediately | Defaults to `false` |
| `projectRoot` | Default project root path for file discovery | Defaults to the process working directory |

**Note:** `aiCache` is **not** a constructor dependency of Step 15. The step makes
live AI requests on every run without caching.

The `projectRoot` set at construction time may be overridden by the value in the
`context` object at execution time (§5.1).

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Source | Description |
|---|---|---|
| `context` | Orchestrator | Full workflow context object |
| `context.projectRoot` | Preferably Step 00 `contextUpdate` | Overrides the constructor `projectRoot` when present |
| `context.projectType` | Preferably Step 00 `contextUpdate` | Project kind identifier; defaults to `'generic'` when absent |

### 5.2 Execution Phases

Step 15 runs the following phases in order:

#### Phase 1 — Dry-Run Check

When the `dryRun` constructor flag is `true`, the step logs what it would do and
returns `{ success: true, dryRun: true, message: 'UX analysis dry run completed' }`.
No further phases execute.

#### Phase 2 — Project Type Eligibility Check

Resolve `projectType` from the context (default `'generic'`).

If the project type is not eligible (§3.1):

- Perform the UI file probe fallback (§3.2): scan the project root for any UI files.
- If the probe finds at least one UI file, log the count and continue to Phase 3
  treating the project as eligible.
- If the probe finds no UI files, write a skip note to the backlog and return
  `{ success: true, skipped: true, reason: 'project type not eligible' }`.

#### Phase 3 — Discover UI Files

Scan the project root for all files matching any recognised UI extension (§3.3),
excluding all excluded directories (§3.4).

- If no UI files are found, write a skip note to the backlog and return
  `{ success: true, skipped: true, reason: 'no UI files found' }`.

#### Phase 4 — Group, Select, and Read Key File Contents

1. Group discovered UI files by type (§3.3).
2. Take a sample of up to 20 files for the prompt file list (used for display only).
3. Select key files for content injection using the prioritisation algorithm (§3.5).
4. Read each key file's content. Each file read is wrapped in its own error handler;
   unreadable files are silently skipped. Content is truncated to 3 072 bytes per file.
   Reading stops when the cumulative total reaches 20 480 bytes.

The collected file contents ground the AI in actual source code, preventing fabrication.

#### Phase 5 — Build Analysis Prompt

Assemble the prompt from two layers:

1. **Base prompt** (built-in): includes project type, file count, type breakdown, file
   list sample, and the injected file content blocks from Phase 4. The prompt instructs
   the AI to cover accessibility (WCAG 2.1), usability, visual design, component
   architecture, and performance; and to produce a structured Markdown response with
   `## Critical Issues`, `## Warnings`, and `## Improvement Suggestions` sections.

2. **YAML template enrichment** (optional): attempt to load the `ui_ux_designer_prompt`
   template from the AI helpers configuration. If successful, build a personalised
   prompt using `project_name`, `file_count`, and `project_type`, then prepend it to
   the base prompt, separated by a horizontal rule. A project-kind `ux_designer` role
   overlay is prepended further if one is defined in the project kinds configuration.

If the YAML template cannot be loaded, the base prompt is used unchanged.

#### Phase 6 — AI Analysis

Initialise `aiHelper`. If initialisation fails, write a warning note to the backlog
and return `{ success: true, skipped: true, reason: 'AI helper not available',
fileCount: <count> }`.

When AI is available, submit the assembled prompt with persona `ux_analyst`. The
response is expected to be a Markdown string. No caching is applied.

#### Phase 7 — Parse Issue Counts

Extract issue counts from the AI response (§3.6):
- `criticalCount` — count of critical severity markers
- `warningCount` — count of warning severity markers
- `suggestionCount` — count of suggestion sub-headings
- `totalIssues` — sum of the three counts

#### Phase 8 — Format Report and Save to Backlog

Produce a structured Markdown report including the ISO timestamp, project type, file
count, issue summary table, and the full AI analysis body. Write this report to the
backlog summary entry for step `'15'`, section `'UX_Analysis'`, with status `✅`.

Log the issue counts.

### 5.3 Skip Conditions

| Reason | Condition |
|---|---|
| `'project type not eligible'` | Project type not in eligible list (§3.1) and UI file probe finds no files |
| `'no UI files found'` | Eligible project type but filesystem scan returns an empty UI file list |
| `'AI helper not available'` | `aiHelper.initialize()` returns falsy |

All skip results return `{ success: true, skipped: true, reason: <value> }`.

### 5.4 Error Handling

All unhandled exceptions are caught. When an exception occurs:

- The error message is written to the backlog issues entry for step `'15'`.
- The step returns `{ success: false, error: <message>, duration: <ms> }`.

This is fully compliant with the step contract.

---

## 6. Data Shapes

### 6.1 StepResult (success, not skipped)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `fileCount` | Integer | Total number of UI files discovered |
| `fileGroups` | Object | UI file paths grouped by type (§3.3 keys) |
| `issueCounts` | Object | Parsed issue counts from AI response (§6.2) |
| `duration` | Integer | Wall-clock execution time in milliseconds |

### 6.2 Issue Counts

| Field | Type | Description |
|---|---|---|
| `criticalCount` | Integer | Count of critical severity issues in AI response |
| `warningCount` | Integer | Count of warning severity issues in AI response |
| `suggestionCount` | Integer | Count of improvement suggestions in AI response |
| `totalIssues` | Integer | Sum of all three counts |

### 6.3 StepResult (skip)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `reason` | String | Skip reason identifier |
| `fileCount` | Integer | Present only when reason is `'AI helper not available'` |

### 6.4 StepResult (failure)

| Field | Type | Description |
|---|---|---|
| `success` | `false` | — |
| `error` | String | Exception message |
| `duration` | Integer | Wall-clock time in milliseconds |

---

## 7. Constraints and Rules

1. Step 15 **must** attempt the UI file probe before giving up when the project type
   is ineligible. A project declared as `nodejs_api` that contains a `.vue` front-end
   sub-project must still receive UX analysis.

2. File content injection **must** comply with the AI Prompt Contract §3: each file
   read is wrapped in its own error handler, unreadable files are silently skipped, and
   the cumulative character budget (20 480 bytes) is respected by stopping reads early.

3. Step 15 **must not** use `aiCache`. The step makes live AI requests on every run.
   Adding `aiCache.withCache` to this step would require implementing `aiCache.init()`
   too, as per the AI Prompt Contract §6.2.

4. The key file selection algorithm (§3.5) **must** preserve the HTML-priority and
   CSS-reservation logic. Reversing the priority or removing the CSS reservation
   reduces the accessibility signal available to the AI.

5. The project-kind role overlay and YAML template enrichment are optional. Failure to
   load either must not prevent the base prompt from being submitted to the AI.

6. `projectRoot` at execution time overrides the constructor default. The step must
   update its internal `projectRoot` from `context.projectRoot` before beginning file
   discovery.

7. The step **must** write a backlog entry for every outcome: skip, failure, and
   success. Silent outcomes that do not update the backlog are not permitted.
