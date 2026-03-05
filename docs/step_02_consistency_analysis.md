# Step 02: Consistency Analysis — Functional Specification

**Step identifier:** `step_02`
**Step name:** Consistency Analysis
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Draft
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 02 analyses a project's documentation files for internal consistency defects: broken
file-reference links and version string mismatches. It then uses an AI
`documentation_expert` persona to provide deeper consistency review and remediation
recommendations, partitioning large documentation sets into prompt-safe chunks to avoid
exceeding model context limits.

Step 02 answers two questions:

1. **Are the project's documentation files internally consistent?** — do cross-file links
   resolve to real files, and do version strings match the canonical project version?
2. **What should be done to fix any inconsistencies?** — AI-generated recommendations
   covering broken references, stale version strings, and other consistency concerns.

---

## 2. Step Kind Conformance

Step 02 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **Yes** — when no documentation files are found |
| AI-integrated | **Yes** — calls the AI API via `aiHelper` and `aiCache` |

**Logging convention.** Step 02 uses the **global logger singleton** directly, which is
consistent with the PROJECT step convention defined in `step_contract.md §2.1`.

**Error handling deviation.** Step 02's top-level catch block logs the error and then
**re-throws** it rather than returning `{ success: false, error: … }`. This violates
`step_contract.md §5` rule 4, which requires that `execute` never propagate an unhandled
exception. Callers must be prepared to catch exceptions from this step.

---

## 3. Domain Concepts

### 3.1 Documentation File Discovery

Step 02 scans the project for documentation files using the following glob patterns:
`**/*.md`, `**/README*`, `**/CHANGELOG*`, `**/CONTRIBUTING*`.

The following directories are excluded from discovery:
`node_modules`, `.git`, `dist`, `build`, `coverage`, `venv`, `.venv`, `env`.

Duplicate paths (matched by multiple patterns) are deduplicated.

### 3.2 Expected Version Resolution

The project's canonical version is resolved in priority order:

1. **`package.json`** — the `version` field is used when the file is present and
   parseable.
2. **`.workflow-config.yaml`** — the `project.version` field is used as a fallback.

When neither source yields a version, version checking is skipped entirely.

### 3.3 Version Consistency Check

For each documentation file that contains at least one version string, Step 02 extracts
all strings matching the semantic version pattern (with or without a `v` prefix and
optional pre-release label). Each extracted version is compared to the canonical version.
If they differ (after normalising away the `v` prefix), an issue is recorded.

When no canonical version could be resolved (§3.2), this check produces no issues.

### 3.4 Link Extraction and Validation

Step 02 extracts two categories of links from each documentation file:

| Link type | Pattern |
|---|---|
| Markdown link | `[text](url)` |
| Autolink | `<http://…>` or `<https://…>` |

For each extracted link, Step 02 checks whether the URL is a file reference (not an
external `http://` or `https://` URL and not a bare anchor starting with `#`). File
references are resolved relative to the source file's directory and validated against the
full file index.

The **file index** is built from the project directory in two passes: regular files and
dotfile directories (e.g. `.github/`). It includes both file paths and all ancestor
directory paths down to the project root, so that directory link targets (e.g.
`.github/scripts/`) resolve correctly.

A link that resolves to a path not present in the index is recorded as a broken link
issue.

### 3.5 Prompt Partitioning

To prevent individual AI prompts from exceeding model context limits, Step 02 divides the
discovered documentation files into partitions of at most **50 files** each. Each
partition is sent to the AI as a separate request.

For each partition, Step 02 builds a prompt context that includes:
- The list of documentation files in that partition.
- The subset of broken-link issues whose **source file** belongs to the partition,
  formatted as `source:line → target` pairs, capped at **30 pairs** per partition.
- A partition header (`[Partition N of M — analyse ONLY the files listed below]`) when
  more than one partition exists.

All prompts are hard-truncated at **60,000 characters** before submission if they exceed
that limit.

### 3.6 AI Response Quality Validation

After each AI response is received, Step 02 evaluates its quality:

- An empty or blank response is classified as `empty_response`.
- A response shorter than 200 characters (when broken references were sent) is classified
  as `too_short`.
- Otherwise, Step 02 checks what fraction of the broken-reference targets (the part after
  `→` in each `source:line → target` pair) appear in the response text. If fewer than
  **50%** of the flagged targets are mentioned, the response is classified as
  `low_coverage`.

A low-quality response causes a warning to be logged. It does not cause the step to fail.

### 3.7 Project-Kind Role Overlay

When a project-kind configuration file is available, Step 02 loads the role definition
for the `code_reviewer` role within the detected project kind and prepends it to each
partition prompt as a role declaration. This overlays project-specific expertise onto the
base `documentation_expert` persona.

---

## 4. Constructor Dependencies

Step 02 receives all external collaborators at construction time via a single options map.
All dependencies fall back to new instances when not provided.

| Dependency key | Role | When absent |
|---|---|---|
| `fileOps` | Reads file contents, performs glob patterns for discovery and file indexing | Falls back to a new FileOperations instance |
| `backlog` | Writes consistency reports and AI recommendations to the workflow artifact store | Falls back to a new Backlog instance |
| `aiHelper` | Executes AI API requests | Falls back to a new AiHelper instance |
| `aiCache` | Caches and retrieves AI responses by cache key | Falls back to a new AiCache instance |
| `techStack` | Detects the project's primary programming language | Falls back to a new TechStackDetector instance |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the root of the project under analysis |
| `options.language` | Override for primary language detection; when absent, language is auto-detected |
| `options.projectDescription` | Project description string injected into the YAML prompt template |
| `options.scope` | Change scope string (e.g. `"full-stack"`) injected into the YAML prompt template |
| `options.projectKind` | Project kind identifier used to load the role overlay (§3.7); defaults to `"default"` |
| `options.modifiedFiles` | Reserved; not currently used by this step |

### 5.2 Execution Phases

Step 02 runs the following phases in order:

#### Phase 1 — Discover Documentation Files

Apply the discovery patterns and exclusions (§3.1) to the project root. Deduplicate
results.

If the resulting list is empty, skip immediately:
```
{ success: true, skipped: true, reason: 'no_docs' }
```

Log the count of discovered documentation files.

#### Phase 2 — Resolve Expected Version

Attempt to load the canonical version from `package.json`, then `.workflow-config.yaml`
(§3.2). Log the resolved version, or `"not found"` when absent.

#### Phase 3 — Check Version Consistency

For each documentation file, read its content and extract all version strings. Compare
each against the canonical version (§3.3). Collect all mismatch issues.

Log the count of version issues found. When no canonical version was resolved, this phase
produces zero issues.

#### Phase 4 — Build File Index and Check Links

Build the file index for the project root (§3.4) in two parallel passes (regular and
dotfile entries). For each documentation file, extract all links and validate file
references against the index. Collect all broken-link issues.

Log the count of broken links found.

#### Phase 5 — Generate Structural Report

Compute total issue count (`versionIssues + brokenLinks`). Format a Markdown consistency
report summarising:
- Files checked
- Total issues
- Broken links (up to 10 shown; remainder counted)
- Version issues (up to 10 shown; remainder counted)

Write the report to the backlog under step `2`, section `"Consistency Analysis"`. This
write happens before the AI phase; the AI phase may overwrite the backlog entry with an
enriched version.

#### Phase 6 — AI Consistency Analysis (Partitioned)

Initialise the AI helper. When unavailable, log a warning and skip this phase (Phase 7
still executes with the structural report only).

When available:

**6a — Initialise cache.** Call `aiCache.init()`.

**6b — Detect language.** Use the `language` option or auto-detect via the tech-stack
service. Defaults to `"javascript"` on detection failure.

**6c — Load configuration.** Attempt to load the AI helpers YAML configuration from the
package's `.workflow_core/config/ai_helpers.yaml`. Attempt to load the project-kind
configuration from the package's `.workflow_core/config/project_kinds.yaml` (§3.7). Both
loads are optional; failure is silently ignored.

**6d — Partition and iterate.** Divide the relative paths of all discovered documentation
files into partitions of at most 50 files (§3.5). For each partition:

1. Build the prompt context (documentation file list, broken-references list, partition
   header).
2. Construct the prompt:
   - When the YAML configuration is available, use the `step2_consistency_prompt` template
     with substitution of `project_name`, `project_description`, `primary_language`,
     `change_scope`, `doc_count`, `modified_count`, `broken_refs_content`, and `doc_files`.
     Prepend the role overlay (§3.7) and partition header when present.
   - When the YAML configuration is unavailable, use the built-in consistency prompt
     builder and prepend the partition header.
3. Hard-truncate the prompt at 60,000 characters (§3.5), logging a warning when
   truncation occurs.
4. Submit to the AI using the cache key
   `step_02|{projectRoot}|part{i}of{total}|{docFileCount}|{totalIssues}`, with the
   `documentation_expert` persona.
5. Validate the AI response quality (§3.6). Log a warning when quality is inadequate.
6. Accumulate the response in the partition results list.

**6e — Merge and persist.** Concatenate all non-empty partition results, separated by
`---` dividers when more than one partition was processed. Append the merged AI
recommendations to the structural report and write the enriched version to the backlog.

#### Phase 7 — Return Result

Return:

```
{
  success: true,
  filesChecked: <integer>,
  totalIssues: <integer>,
  brokenLinks: <issue array>,
  versionIssues: <issue array>
}
```

### 5.3 Skip Conditions

| Reason | Condition |
|---|---|
| `no_docs` | No documentation files discovered in Phase 1 |

Skip results return `{ success: true, skipped: true, reason: 'no_docs' }`.

---

## 6. Data Shapes

### 6.1 Broken Link Issue

Represents a documentation link that could not be resolved to an existing file.

| Field | Type | Description |
|---|---|---|
| `file` | String | Absolute path to the source documentation file |
| `link` | String | The raw URL or path from the link |
| `text` | String | The display text of the link |
| `line` | Integer | Line number within the source file where the link appears |
| `type` | String | Always `"broken_link"` |

### 6.2 Version Issue

Represents a version string in a documentation file that does not match the canonical
version.

| Field | Type | Description |
|---|---|---|
| `file` | String | Absolute path to the documentation file |
| `found` | String | The version string found in the file |
| `expected` | String | The canonical version |
| `type` | String | Always `"invalid_version"` |

### 6.3 StepResult (success, not skipped)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `filesChecked` | Integer | Number of documentation files analysed |
| `totalIssues` | Integer | Sum of broken links and version issues |
| `brokenLinks` | BrokenLinkIssue[] | All broken link issues found (§6.1) |
| `versionIssues` | VersionIssue[] | All version mismatch issues found (§6.2) |

### 6.4 StepResult (skip)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `reason` | `"no_docs"` | Human-readable explanation |

---

## 7. Constraints and Rules

1. **Error handling deviation.** Step 02 re-throws unhandled exceptions from `execute`
   rather than returning `{ success: false, error: … }`. This violates the step contract.
   Callers **must** wrap invocations of Step 02 in their own try/catch.

2. The structural consistency report (broken links, version issues) **must** be written to
   the backlog before the AI phase begins. The structural report is always available, even
   when the AI is unavailable.

3. The AI cache **must** be initialised (`aiCache.init()`) before any `withCache` call.
   The cache key must encode the project root, partition index, total partition count,
   document file count, and total issue count. This ensures that a cache hit is only served
   when the same documentation set and issue context was previously analysed.

4. The AI must be invoked with the **`documentation_expert` persona** for all consistency
   analysis requests. Step 02 performs documentation cross-reference and terminology
   review, not code quality review. The prompt template and the persona declaration must
   agree.

5. Prompt hard-truncation (at 60,000 characters) is a safety measure and **must** be
   applied after all other prompt construction steps. A warning must be logged whenever
   truncation occurs. Silently sending an over-limit prompt is not permitted.

6. AI response quality validation (§3.6) **must** be applied to every partition response.
   A low-quality result **must not** cause the step to fail, but it **must** be logged as a
   warning with the reason code and coverage percentage.

7. Broken-references in partition prompts are presented as `source:line → target` pairs so
   the AI knows **what is broken and where**, not merely which files contain broken links.
   Formatting broken links as bare filenames is incorrect and must not be used.

8. The file index used for link validation **must** be built with two passes to include
   dotfile directories. A single-pass glob that omits dotfiles causes links into `.github/`
   and similar directories to be incorrectly classified as broken.

9. Step 02 does not inject file contents into AI prompts. It provides only file lists and
   broken-reference summaries. Implementors adding content injection in future must do so in
   accordance with the [AI Prompt Contract §3](./ai_prompt_contract.md).
