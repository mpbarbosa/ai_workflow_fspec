# Step 13: Markdown Lint — Functional Specification

**Step identifier:** `step_13`
**Step name:** Markdown Lint
**Step kind:** `CONTEXT` (see [Step Contract §2.2](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 13 validates Markdown formatting across all Markdown files in the project. It runs
the `mdl` (markdownlint) command-line linter in batches, attempts to auto-fix correctable
violations using `markdownlint-cli`, scans for four common structural anti-patterns, and
consults the AI for targeted, rule-specific remediation advice.

Step 13 operates at the workflow level, not just on changed files: it lints the entire
project's Markdown corpus. As a CONTEXT kind step it receives the full workflow context
and logs through an injected logger rather than the global singleton.

---

## 2. Step Kind Conformance

Step 13 conforms to the **CONTEXT** step kind as defined in `step_contract.md §2.2`.

| Property | Value |
|---|---|
| Kind identifier | `"context"` |
| Execute signature | `execute(context) → Promise<StepResult>` |
| Can be skipped | **Yes** — three conditions produce a graceful exit |
| AI-integrated | **Yes** — one AI call (`technical_writer` persona) |
| Logger source | `this.logger`, injected via the constructor; falls back to `console` |

**Error-handling deviation.** Step 13 re-throws unhandled exceptions from the main
execution path rather than returning `{ success: false, error }`. This deviates from step
contract rule 4 (§5.4 of `step_contract.md`). Callers must be prepared to catch exceptions
propagated from this step.

**AI Prompt Contract conformance.** Step 13 conforms to
[`ai_prompt_contract.md`](./ai_prompt_contract.md) for prompt construction with an
intentional design choice regarding file content. Unlike steps that analyse source code,
Step 13 does not inject the raw content of Markdown files into the AI prompt. Instead, it
injects a structured lint report — violation counts, rule codes, and affected file names —
derived from the linter output. This is deliberate: the AI is asked to interpret linting
statistics and recommend rule-level fixes, not to read or rewrite prose. The fallback
prompt includes an explicit anti-hallucination guard: *"Only reference the files and rules
listed above. Do not invent file paths or rule names not present in this data."*

**AI skip when clean.** When the lint run produces no violations and no anti-patterns are
detected, the AI phase is skipped entirely and the step returns the result immediately.
Calling the AI when there is nothing to analyse would produce fabricated content.

---

## 3. Domain Concepts

### 3.1 Markdown File Enumeration

The step recursively discovers all files with a `.md` extension under `projectRoot`. Files
whose path falls under any of the following directories are excluded:
`node_modules`, `coverage`, `.git`, `dist`, `build`, `.ai_workflow`.

Exclusion is tested against both Unix (`/dir/`) and Windows (`\dir\`) path separators, and
also against prefixes at the path root (e.g. a file directly inside `node_modules/`).

### 3.2 Linter: mdl

The step uses the `mdl` command-line tool (Ruby gem). Before linting, the step checks
availability by running `mdl --version`. If that command fails, the step skips.

The linter is invoked as:

```
mdl --ignore-front-matter [--style <path>] <file1> <file2> …
```

When a `.mdl_style.rb` file exists in `projectRoot`, it is passed via `--style` to allow
the project to align `mdl` rule configuration with `markdownlint-cli` configuration. Files
are passed as explicit arguments rather than relying on `mdl`'s own directory recursion;
this avoids race conditions with git lock files.

`mdl` exits with a non-zero code when it finds violations. This is expected behaviour;
the combined stdout and stderr is still parsed for issues. An empty output with a non-zero
exit code is treated as a real execution failure rather than a violations report.

### 3.3 mdl Output Format

Each `mdl` output line follows the format:

```
<file>:<line>: <ruleCode> <message>
```

For example: `README.md:12: MD022 Headings should be surrounded by blank lines`

Lines that do not match this format are silently ignored.

### 3.4 Batch Linting

The enumerated file list is divided into batches of at most 100 files before being passed
to `mdl`. Batching prevents command-line length overflow and avoids git-lock race
conditions that arise when `mdl` recurses a directory itself. Issues from all batches are
merged into a single flat list before further processing.

### 3.5 Auto-Fix

When the initial lint run produces any violations, the step attempts to auto-fix them
using `markdownlint-cli`:

1. **Preferred path.** Run `npm run lint:md:fix` in `projectRoot`. This uses the project's
   own npm script, which inherits the project's `markdownlint` configuration.

2. **Fallback path.** When the npm script does not exist or fails, run
   `npx markdownlint --fix` with the same exclude directories as ignore arguments and the
   full enumerated file list as targets.

When either path succeeds, auto-fix is considered applied. The step then re-runs mdl on
the same file list to obtain the post-fix violation state. If neither path succeeds,
auto-fix is unavailable and violations require manual review.

### 3.6 Anti-Pattern Detection

Four patterns are checked against each file's content. Detection is bounded to the first
50 enumerated files to avoid excessive processing overhead on large corpora.

| Pattern identifier | What is detected |
|---|---|
| `missing-space-after-hash` | A heading marker (`#` through `######`) immediately followed by a non-space, non-hash, non-shebang character |
| `malformed-bold` | A line containing a single asterisk before a bold sequence |
| `trailing-whitespace` | Any line ending with one or more whitespace characters |
| `multiple-blank-lines` | Three or more consecutive blank lines |

Files that cannot be read are silently skipped. Anti-pattern issues are tagged with their
file path, line number, message, and pattern identifier.

### 3.7 Lint Status Classification

After analysis, a lint status value is derived from the aggregate issue statistics:

| Status | Condition |
|---|---|
| `pass` | Total issues = 0 |
| `warning` | Total issues > 0, and average issues per file < 5 |
| `fail` | Average issues per file ≥ 5 |

The `warning` status is non-blocking: the step still returns `success: true`. Only a
`fail` status signals a documentation quality problem that warrants attention.

### 3.8 Dry Run Mode

When the `dryRun` constructor option is `true`, the step logs a preview of the actions it
would take (enumerate files, check mdl, run linting, detect anti-patterns) without
performing any of them. A dry-run summary is written to the backlog and the step returns
`{ success: true, dryRun: true, message: 'Dry run completed' }`.

---

## 4. Constructor Dependencies

Step 13 receives all external collaborators at construction time via a single options map.

| Dependency key | Role | Behaviour when absent |
|---|---|---|
| `logger` | Emits log output; satisfies the CONTEXT kind logging convention | Falls back to `console` |
| `executor` | Runs shell commands (`mdl`, `npm run lint:md:fix`, `npx markdownlint`, `git`) | Commands fail at runtime when the executor is absent |
| `fileOps` | Recursively enumerates Markdown files and reads file content for anti-pattern detection | Enumeration returns an empty list; anti-pattern detection is skipped |
| `backlogManager` | Writes step issues and summaries to the workflow artifact store | Backlog writes are silently skipped |
| `aiHelper` | Executes AI API requests | Falls back to a new AI helper instance |
| `aiCache` | Caches and retrieves AI responses by key | Falls back to a new AI cache instance |
| `dryRun` | Boolean flag; when `true`, activates dry-run mode (§3.8) | Defaults to `false` |
| `promptsDir` | Directory path for AI prompt templates | Passed to the AI helper; `null` when absent |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `context.projectRoot` | Root directory of the project. Falls back to the current working directory when absent. |
| `context.currentBranch` / `context.branch` | Current git branch name. Used in the AI prompt. When absent from context, the step queries git directly. |
| `context.modifiedFiles` | List of modified file paths. Used to count the number of modified Markdown files for the AI prompt. |
| `context.modifiedCount` | Pre-computed count of modified files. When present, used directly in place of computing from `modifiedFiles`. |

### 5.2 Execution Phases

Step 13 executes the following phases in order:

#### Phase 0 — Dry Run Check

If the `dryRun` constructor flag is set, log a description of planned actions, write a
dry-run summary to the backlog (if `backlogManager` is available), and return
`{ success: true, dryRun: true, message: 'Dry run completed' }`. No further phases execute.

#### Phase 1 — Enumerate Markdown Files

Recursively discover all `.md` files under `projectRoot`, excluding the directories listed
in §3.1. When `fileOps` is absent, the enumerated list is empty. If the resulting list is
empty, write a summary to the backlog and return:

```
{ success: true, noFiles: true, message: 'No markdown files to lint' }
```

#### Phase 2 — Check Linter Installation

Run `mdl --version`. If the command succeeds, capture the version string. If it fails,
write an issue entry and a skip summary to the backlog (both under step `'13'`, section
`'Markdown_Linting'`), and return:

```
{ success: true, skipped: true, reason: 'mdl not installed' }
```

#### Phase 3 — Run mdl Linting

Divide the enumerated file list into batches of at most 100 files (§3.4). Determine
whether a `.mdl_style.rb` style file exists in `projectRoot` and, if so, include the
`--style` flag in the base command. For each batch, run the linter and parse the output
(§3.3). Merge all batch results into a single flat issue list. Log whether violations were
found.

#### Phase 3.5 — Auto-Fix (Conditional)

When the merged issue list is non-empty, attempt auto-fix (§3.5). When auto-fix is applied
successfully, re-run Phase 3 to capture the post-fix violation state and log the number of
remaining violations.

#### Phase 4 — Anti-Pattern Detection

For up to the first 50 enumerated files, read the file content and run all four anti-pattern
checks (§3.6). Collect all detected occurrences tagged with their file path. Unreadable
files are silently skipped.

#### Phase 5 — Report Generation and Backlog Write

Compute aggregate lint statistics (§6.2). Determine the lint status (§3.7). Generate a
structured Markdown report including: linter version, file counts, issues by rule (sorted
by occurrence count descending), issues by file (top 10, sorted descending), anti-pattern
counts, and an overall quality rating.

Write the report as step issues (`'13'`, `'Markdown_Linting'`) to the backlog. Also write
a one-line summary sentence and the status emoji (`✅`, `⚠️`, or `❌`) as the step
summary entry.

#### Phase AI — Lint Analysis (Conditional)

**When the AI is skipped:** If the total lint issue count is zero **and** the anti-pattern
count is zero (a fully clean run), skip the AI phase and return the result assembled in
Phase 5 directly. Generating AI output when there is nothing to analyse risks fabricating
findings.

**When the AI runs** (any issues or anti-patterns are present and the AI helper
initialises successfully):

1. **Initialise the response cache.** Call `aiCache.init()` before any `withCache` call.

2. **Build grounding data.** Produce two structured lists from the lint results:
   - Issues by rule: sorted by occurrence count descending, formatted as
     `<ruleCode>: <count> occurrence(s)`.
   - Issues by file: top 10 files by issue count, formatted as `<file>: <count> issue(s)`.
   These lists are the only data the model is given about violations; they prevent the
   model from referencing rules or files not present in the actual output.

3. **Build the lint report string.** Assemble a structured text block containing:
   file count, total issues, clean files, anti-pattern count, status, the by-rule list,
   and the by-file list. This string is passed as the `lint_report` prompt variable.

4. **Resolve the current branch.** Use `context.currentBranch` or `context.branch` when
   present. When absent, run `git -C "<projectRoot>" branch --show-current`. Leave empty
   if git is unavailable.

5. **Compute the modified Markdown count.** Use `context.modifiedCount` when present.
   Otherwise, count files in `context.modifiedFiles` whose path ends with `.md`. Default
   to `'unknown'` when neither is available.

6. **Build the AI prompt.** Load the `markdown_lint_prompt` template from the AI helpers
   YAML configuration and populate: `project_name`, `lint_report`, `current_branch`,
   `modified_md_count`. When the template is unavailable, fall back to a built-in
   structured prompt that embeds all the same data and includes the explicit
   anti-hallucination instruction.

7. **Execute via cache.** Cache key: `step_13|{fileCount}|{totalIssues}`.
   Persona: `technical_writer`.

8. **Enrich the backlog.** If a response was produced and `backlogManager` is available,
   overwrite the plain summary from Phase 5 with an enriched version that appends the AI
   analysis under `"AI Recommendations"`.

When the AI helper is unavailable, the phase is skipped with a warning logged and the
plain Phase 5 report is retained.

### 5.3 Skip Conditions

| Reason | Condition | Result shape |
|---|---|---|
| Dry run | `dryRun` constructor flag is `true` | `{ success: true, dryRun: true, message: 'Dry run completed' }` |
| No files | No Markdown files found after enumeration and exclusion filtering | `{ success: true, noFiles: true, message: 'No markdown files to lint' }` |
| `mdl not installed` | `mdl --version` fails | `{ success: true, skipped: true, reason: 'mdl not installed' }` |

**Note on the "no files" shape.** The no-files result uses `noFiles: true` rather than
the standard `skipped: true` field. This is a minor deviation from the conventional skip
shape defined in `step_contract.md §4.1`.

---

## 6. Data Shapes

### 6.1 StepResult (normal completion)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed; status may still indicate issues |
| `status` | String | `'pass'`, `'warning'`, or `'fail'` (§3.7) |
| `stats` | LintStats | Aggregate statistics (§6.2) |
| `issues` | LintIssue[] | All lint violations from mdl after auto-fix (§6.3) |
| `antiPatterns` | AntiPattern[] | Detected anti-pattern occurrences (§6.4) |
| `report` | String | Formatted Markdown report text generated in Phase 5 |

### 6.2 LintStats

| Field | Type | Description |
|---|---|---|
| `totalIssues` | Integer | Total number of lint violations |
| `filesWithIssues` | Integer | Number of distinct files with at least one violation |
| `filesChecked` | Integer | Total Markdown files that were linted |
| `cleanFiles` | Integer | Files with no violations (`filesChecked − filesWithIssues`) |
| `uniqueRules` | Integer | Number of distinct rule codes triggered |
| `issuesPerFile` | Number | Average violations per file (`totalIssues / max(filesChecked, 1)`) |

### 6.3 LintIssue

| Field | Type | Description |
|---|---|---|
| `file` | String | Path to the file containing the violation |
| `line` | Integer | Line number of the violation |
| `rule` | String | mdl rule code (e.g. `'MD022'`) |
| `message` | String | Rule violation description as reported by mdl |
| `severity` | `'warning'` | Always `warning`; mdl does not distinguish severity levels |

### 6.4 AntiPattern

| Field | Type | Description |
|---|---|---|
| `file` | String | Path to the file where the pattern was detected |
| `line` | Integer | Line number of the occurrence |
| `message` | String | Human-readable description of the anti-pattern |
| `pattern` | String | Pattern identifier (one of the four values in §3.6) |

### 6.5 StepResult (dry run)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `dryRun` | `true` | — |
| `message` | `'Dry run completed'` | — |

### 6.6 StepResult (no files)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `noFiles` | `true` | Deviation: uses `noFiles` rather than `skipped` |
| `message` | `'No markdown files to lint'` | — |

### 6.7 StepResult (mdl not installed)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `reason` | `'mdl not installed'` | — |

---

## 7. Constraints and Rules

1. The step **must** use `this.logger` for all log output, consistent with the CONTEXT kind
   logging convention (§2.2 of `step_contract.md`). It **must not** obtain or instantiate
   a logger independently outside the constructor.

2. Files under excluded directories (§3.1) **must** be filtered out at enumeration time
   and **must not** be passed to `mdl`. Linting workflow infrastructure files (e.g.
   `.ai_workflow/`) or third-party code (`node_modules/`) produces noise and may cause
   false failures.

3. The linter **must** be invoked with enumerated file arguments, not by pointing it at a
   directory for recursive scanning. Directory-based recursion creates git-lock race
   conditions when the linter's internal directory walking interferes with concurrent git
   operations in the workflow.

4. Batching **must** cap each mdl invocation at 100 files. This prevents operating-system
   command-line length limits from being exceeded on large Markdown corpora.

5. Auto-fix **must** re-lint the file list after a successful fix pass. Skipping the
   re-lint would cause the step to report the pre-fix violation count in its results and
   backlog entry.

6. The AI phase **must** be skipped when `totalIssues === 0` and `antiPatterns.length === 0`.
   Invoking the AI on a clean run provides no useful signal and risks generating fabricated
   findings about non-existent problems.

7. The AI prompt **must** be grounded in the actual lint data (rules violated, files
   affected). The model **must not** be instructed to examine the repository, reference
   files not listed, or apply general Markdown knowledge beyond what the provided data
   supports. (Ref: AI Prompt Contract §4, §5.)

8. The AI response cache **must** be initialised (via `aiCache.init()`) before the first
   `withCache` call. Failure to do so causes a runtime error.

9. Anti-pattern detection **must** be bounded to the first 50 enumerated files. This limit
   exists to prevent excessive I/O and processing time on repositories with many Markdown
   files.

10. The "no files" skip result uses `noFiles: true` rather than `skipped: true`. This is a
    known deviation from the standard skip shape. Callers that check only for `skipped`
    will not recognise this condition as a skip; they should also check for `noFiles`.

11. **Exception propagation deviation.** Callers of `execute` **must** wrap the call in
    error handling. Step 13 does not guarantee that exceptions are converted to
    `{ success: false }` results; unhandled exceptions are re-thrown.
