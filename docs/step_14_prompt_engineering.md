# Step 14: Prompt Engineering Analysis — Functional Specification

**Step identifier:** `step_14`
**Step name:** Prompt Engineer Analysis
**Step kind:** `CONTEXT` (see [Step Contract §2.2](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 14 reviews and scores all AI persona prompts defined in the shared AI helpers
configuration file. It performs static quality analysis on each persona's role, task,
and approach sections, then invokes the AI using the `prompt_engineer` persona to
generate structural improvement recommendations.

Step 14 answers three questions:

1. **How many personas are defined?** — and what are their names?
2. **How good are the prompts?** — scored 0–100 per persona, with per-criterion breakdown.
3. **What should be improved?** — specific, prioritised improvement opportunities
   surfaced both by static analysis and by AI inference.

The step is intentionally narrow in scope: it only runs on workflow automation projects
where the AI helpers configuration is meaningful. On other project types it skips
immediately, writing a brief note to the backlog.

---

## 2. Step Kind Conformance

Step 14 conforms to the **CONTEXT** step kind as defined in `step_contract.md §2.2`.

| Property | Value |
|---|---|
| Kind identifier | `"context"` |
| Execute signature | `execute(context) → Promise<StepResult>` |
| Can be skipped | **Yes** — multiple conditions produce a graceful skip |
| AI-integrated | **Yes** — calls the AI API via `aiHelper` and `aiCache` |
| Conforms to AI Prompt Contract | **Yes** — see §5 Phase 8 for file content and caching details |

**Error handling deviation.** Step 14 re-throws unhandled exceptions from `execute`
rather than returning `{ success: false, error }`. This deviates from the step contract
rule (§5 rule 4) that forbids propagating exceptions. Callers must be prepared to catch.

---

## 3. Domain Concepts

### 3.1 Eligible Project Types

Step 14 runs only on projects whose type is one of the following:

| Identifier | Description |
|---|---|
| `workflow-automation` | A workflow automation project (e.g. this project itself) |
| `bash-automation-framework` | A Bash-based automation framework |
| `configuration_library` | A configuration library used by automation tooling |

When the project type does not match any of these, the step skips immediately. No prompt
analysis is performed.

### 3.2 Persona Discovery

A persona is discovered by scanning the AI helpers configuration file line-by-line for
entries matching the pattern `<identifier>_prompt:` at the start of a line (where
`<identifier>` contains only lowercase letters, digits, and underscores). Each match
contributes one persona name to the analysis list.

### 3.3 Prompt Sections

Each persona prompt is expected to define three sections within its YAML block:

| Section | Key in config | Description |
|---|---|---|
| Role | `role` | Defines the AI's identity for this persona |
| Task | `task_template` | Describes what the AI must do |
| Approach | `approach` | Provides guidelines, tone, and structure |

When a persona block defines none of these sections, the persona is skipped silently
with a warning logged.

### 3.4 Quality Scoring

Each persona is assigned a quality score on a 0–100 scale. The score is the weighted
sum of criterion scores divided by the maximum possible weighted sum, expressed as a
percentage.

| Criterion | Weight | Condition for full marks |
|---|---|---|
| Clarity | 3 | Role section is present and ≥ 20 characters; task section is ≥ 50 characters |
| Specificity | 2 | Task section contains at least one action verb: `analyze`, `identify`, `validate`, `generate`, or `review` |
| Structure | 2 | Both role and task sections are present |
| Examples | 1 | Approach section contains `example`, `e.g.`, or `for instance` |
| Context | 2 | Approach section is ≥ 100 characters |

Partial marks apply to clarity: role presence accounts for 80% of the clarity weight;
task length accounts for the remaining 20%.

### 3.5 Quality Ratings

| Rating | Score threshold |
|---|---|
| `excellent` | ≥ 90 |
| `good` | ≥ 75 |
| `needs-improvement` | ≥ 60 |
| `poor` | < 60 |

### 3.6 Improvement Opportunities

Static analysis produces a list of improvement items for each persona. Each item has:

| Field | Description |
|---|---|
| `category` | One of: `clarity`, `specificity`, `context`, `examples` |
| `severity` | One of: `high`, `medium`, `low` |
| `issue` | What is wrong |
| `suggestion` | Concrete corrective action |

Improvements are generated when:
- Role is absent or < 20 characters → high-severity clarity issue
- Task is absent or < 50 characters → high-severity specificity issue
- Task lacks action verbs → medium-severity specificity issue
- Approach is absent or < 100 characters → medium-severity context issue
- Approach lacks example markers → low-severity examples issue

---

## 4. Constructor Dependencies

Step 14 receives all collaborators at construction time via a single options map. All
dependencies have fallback defaults.

| Dependency key | Role | When absent |
|---|---|---|
| `logger` | Logging — must use `this.logger` per CONTEXT contract | Falls back to `console` |
| `fileOps` | Reads the AI helpers configuration YAML file | Step will produce config-not-found skip |
| `backlogManager` | Writes step issues and summary to the workflow artifact store | Backlog writes are silently skipped |
| `aiHelper` | Executes AI API requests (see AI Prompt Contract §6.1) | Falls back to a new `AiHelper` instance |
| `aiCache` | Caches AI responses by key (see AI Prompt Contract §6.2) | Falls back to a new `AiCache` instance |
| `configPath` | Path to the AI helpers configuration YAML | Defaults to `.workflow_core/config/ai_helpers.yaml` |
| `dryRun` | Boolean; when `true`, skips all I/O and returns immediately | Defaults to `false` |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Source | Description |
|---|---|---|
| `context` | Orchestrator | Full workflow context object |
| `context.projectType` | Preferably Step 00 `contextUpdate` | Project kind identifier used to gate the step |
| `context.config.projectType` | Fallback | Alternative location for the project kind identifier |

### 5.2 Execution Phases

Step 14 runs the following phases in order:

#### Phase 1 — Dry-Run Check

When the `dryRun` constructor flag is `true`, the step logs what it would do, writes
a dry-run note to the backlog (if available), and returns
`{ success: true, dryRun: true, message: 'Dry run completed' }`. No further phases
execute.

#### Phase 2 — Project Type Eligibility Check

Resolve `projectType` from `context.projectType` or `context.config.projectType`.

If the resolved type does not match any eligible project type (§3.1), skip immediately
with reason `'project type not eligible'` and write a skip note to the backlog.

#### Phase 3 — Load AI Helpers Configuration

Attempt to read the configuration file at `configPath` using `fileOps`.

- If `fileOps` is absent, or the file cannot be read: skip with reason
  `'configuration not found'` and write a warning note to the backlog.

#### Phase 4 — Extract Persona Names

Parse the configuration file content to discover persona names (§3.2).

- If no persona names are found: return
  `{ success: true, noPrompts: true, message: 'No prompts to analyze' }` and write
  a note to the backlog. This is not a standard skip (the `skipped` field is absent);
  it is a no-work completion.

#### Phase 5 — Analyse Each Persona

For each discovered persona name, in order:

1. Extract the prompt sections (role, task, approach) from the configuration content.
   If extraction yields no content (all three sections are empty), log a warning and
   skip this persona.
2. Calculate the quality score (§3.4).
3. Determine the quality rating (§3.5).
4. Identify improvement opportunities (§3.6).
5. Log the persona name, score, and rating.

The result is a list of per-persona analysis records.

#### Phase 6 — Calculate Aggregate Statistics

Compute aggregate statistics across all analysed personas:

- Total number of personas
- Average score (rounded to nearest integer)
- Count per rating (excellent, good, needs-improvement, poor)
- Total number of improvement opportunities across all personas

#### Phase 7 — Generate Static Report and Persist

Produce a structured Markdown report (§6.3) from the aggregate statistics and
per-persona analyses. Write to the backlog:

- **Step issues entry** (`step "14"`, `"Prompt_Engineer_Analysis"`): the full
  Markdown report.
- **Step summary entry** (`step "14"`, `"Prompt_Engineer_Analysis"`): a one-line
  sentence stating total personas, average score, and total improvements.

The status emoji is `✅` when the average score is ≥ 75; otherwise `⚠️`.

#### Phase 8 — AI-Powered Improvement Suggestions

Attempt to initialise `aiHelper`. If the AI is unavailable, log a warning and skip this
phase.

When AI is available:

1. Initialise `aiCache` (call `init()` before any `withCache` call).
2. Attempt to load the `step13_prompt_engineer_prompt` template from the YAML
   configuration. Populate the template with:
   - `total_prompts` — total persona count
   - `average_score` — average quality score
   - `total_improvements` — total improvement count
   - `below_threshold` — count of personas below the good threshold (defaults to 0)
3. If the template cannot be loaded, fall back to a built-in structured prompt with the
   same data inlined.
4. Invoke the AI with cache key `step_14|{totalPrompts}|{averageScore}`, calling
   `aiCache.withCache(prompt, cacheKey, fn)`. The persona is `prompt_engineer`.
5. If the AI returns non-empty content and a backlog manager is available, append the
   AI output as an `## AI Recommendations` section to the static report and overwrite
   the step summary entry with this enriched version.

**Note on file content injection:** Step 14 does not inject file contents into the AI
prompt. The prompt contains only aggregate numeric metrics (total prompts, average
score, total improvements). Because no specific file contents are referenced, the file
content injection requirement of the AI Prompt Contract §3 does not apply here. The AI
is asked to reason about structural patterns, not analyse specific files.

### 5.3 Skip Conditions

| Reason | Condition | `skipped` field |
|---|---|---|
| `'project type not eligible'` | Resolved project type not in eligible list (§3.1) | `true` |
| `'configuration not found'` | Config file missing or unreadable | `true` |
| *(no-work completion)* | No persona names found in config | `noPrompts: true` (not `skipped`) |

Dry-run mode produces `{ success: true, dryRun: true }` — neither `skipped` nor
`noPrompts`.

---

## 6. Data Shapes

### 6.1 Per-Persona Analysis Record

| Field | Type | Description |
|---|---|---|
| `personaName` | String | Persona identifier (e.g. `documentation_expert`) |
| `prompt` | Object | Extracted sections: `{ role, task, approach }` |
| `score` | Integer | Quality score 0–100 |
| `rating` | String | Quality rating (§3.5) |
| `improvements` | Object[] | List of improvement opportunities (§3.6) |

### 6.2 Aggregate Statistics

| Field | Type | Description |
|---|---|---|
| `totalPrompts` | Integer | Number of personas analysed |
| `averageScore` | Integer | Mean quality score, rounded |
| `excellentCount` | Integer | Personas rated `excellent` |
| `goodCount` | Integer | Personas rated `good` |
| `needsImprovementCount` | Integer | Personas rated `needs-improvement` |
| `poorCount` | Integer | Personas rated `poor` |
| `totalImprovements` | Integer | Total improvement opportunities across all personas |

### 6.3 Markdown Report Structure

The static report (written to the backlog issues entry) contains:

- A summary section: total prompts, average score, total improvements.
- A quality distribution section: counts per rating with status emoji.
- A "Prompts Needing Attention" section (present only when improvements > 0): the top
  10 personas by improvement count, sorted descending, each showing name, score, and
  improvement count. A trailing note is appended when more than 10 personas need attention.

When AI recommendations are appended (Phase 8), a horizontal rule and `## AI
Recommendations` section follow the static report in the backlog summary entry.

### 6.4 StepResult (success, not skipped)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `stats` | Object | Aggregate statistics (§6.2) |
| `analyses` | Object[] | Per-persona analysis records (§6.1) |
| `report` | String | The static Markdown report content |

### 6.5 StepResult (skip)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `reason` | String | Skip reason string |

### 6.6 StepResult (no-work completion)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `noPrompts` | `true` | Signals no personas were found |
| `message` | String | `'No prompts to analyze'` |

---

## 7. Constraints and Rules

1. Step 14 **must** check project type before loading or parsing any configuration.
   Unnecessary file I/O on ineligible project types is a contract violation.

2. Step 14 **must** skip gracefully (not fail) when the configuration file cannot be
   read. A missing config is an expected condition on projects that do not yet have AI
   helpers defined.

3. The quality score calculation **must** be deterministic. Given the same prompt
   content, the same score must always be produced. The calculation must not depend on
   any external state, random values, or time.

4. `aiCache.init()` **must** be called before any `withCache` call, as required by the
   AI Prompt Contract §6.2. Omitting this call causes a runtime type error even in tests.

5. Backlog writes (issues and summary) **must** be best-effort. A failure to write to
   the backlog must not cause the step to return `success: false` or throw.

6. **Error handling deviation.** Step 14 re-throws unhandled exceptions rather than
   returning `{ success: false, error }`. This violates step contract §5 rule 4.
   Callers must wrap `execute` in their own error handler.

7. The AI cache key encodes aggregate statistics (`totalPrompts`, `averageScore`), not
   individual persona content. This means the cache entry is invalidated when the prompt
   population changes, but remains valid across runs that produce the same aggregate
   metrics.
