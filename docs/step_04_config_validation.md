# Step 04: Configuration Validation — Functional Specification

**Step identifier:** `step_04`
**Step name:** Configuration Validation
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))
**Version:** 2.0.0
**Status:** Active
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 04 validates configuration files within the project for syntax correctness, exposed
secrets, and adherence to authoring best practices. It combines static analysis (syntax
parsing, secret pattern matching, best-practice rules) with AI-powered review to produce
actionable remediation guidance.

The step discovers configuration files either from the git working tree (modified files
only, preferred) or from a filesystem glob scan (fallback when the tree is clean or git is
unavailable). It then applies four sequential checks to each discovered file and, when the
AI is available, invites two AI personas to produce targeted recommendations.

---

## 2. Step Kind Conformance

Step 04 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **Yes** — when no configuration files are discovered |
| AI-integrated | **Yes** — partition-aware primary review, optional whole-scope synthesis, and supplementary quality review |

**Error-handling deviation.** Step 04 re-throws unhandled exceptions from `execute` rather
than returning `{ success: false, error }`. This deviates from step contract rule 4
(§5.4 of `step_contract.md`). Callers must be prepared to catch exceptions propagated
from this step.

**AI Prompt Contract conformance.** Step 04 conforms to
[`ai_prompt_contract.md`](./ai_prompt_contract.md). Actual file content is injected into
the AI prompts using content-block builders that render each file as a fenced code block
with its relative path as a header. The primary and supplementary reviews partition
oversized scope into prompt-safe batches instead of silently dropping files, while the
whole-scope synthesis pass uses bounded per-file excerpts to reconcile findings across
partitions. Prompts explicitly instruct the AI to treat truncation markers as partial
evidence rather than as a basis for full-file success claims. File reads are individually
wrapped in error handlers so that an unreadable file is silently skipped rather than
aborting prompt construction.

---

## 3. Domain Concepts

### 3.1 Configuration File Types

Files are categorised into named types based on their filename or extension. Types are
evaluated in priority order (first match wins):

| Type | Matching Rule |
|---|---|
| `ci` | GitHub Actions workflow YAML, CircleCI config, GitLab CI, Jenkinsfile |
| `docker` | `Dockerfile`, `.dockerignore`, `docker-compose.*.yaml` |
| `editor` | `.editorconfig`, `.nvmrc`, `.node-version`, `.mdlrc` |
| `git` | `.gitignore` |
| `make` | `Makefile` |
| `env` | `.env`, `.env.<suffix>` |
| `toml` | `.toml` extension |
| `ini` | `.ini` extension |
| `yaml` | `.yaml` or `.yml` extension |
| `json` | `.json` or `.jsonc` extension |
| `unknown` | Anything else recognised as a configuration file |

### 3.2 Configuration File Discovery

Files are discovered using a two-strategy approach:

**Strategy 1 — Git-modified (preferred).** Query git for the list of currently modified
files. Retain only those that match a recognised configuration type and are not located
under an excluded directory. If this yields at least one file, discovery ends here.

**Strategy 2 — Glob fallback.** Activated when git is unavailable or returns no
matching files. Glob for `**/*.json`, `**/*.yaml`, `**/*.yml`, `**/.env*`, and
`**/Dockerfile` under `projectRoot`. Filter results to those matching a recognised
configuration type. De-duplicate the combined list before returning.

Excluded directories (applied to both strategies):
`node_modules`, `.git`, `dist`, `build`, `coverage`, `.ai_cache`, `venv`, `.venv`, `env`.

### 3.3 Syntax Validation

Each file is validated according to its type:

| Types | Validation |
|---|---|
| `json` | Parse after stripping JSONC-style comments (`//` and `/* … */`). Syntax errors include message and, when available, line and column numbers. |
| `yaml` | Line-by-line check for tab characters (prohibited in YAML) and indentation that is not a multiple of two. Block scalars (`\|`, `>`) are detected and skipped to prevent false positives. |
| `toml`, `ini`, `env`, `docker`, `ci` | Basic validation only; no structural parsing. |

A read failure for any individual file is silently skipped and does not produce a syntax
error.

### 3.4 Secret Pattern Detection

Each file is scanned line-by-line against six regular-expression patterns:

| Pattern name | What it matches |
|---|---|
| AWS Key | AWS access key identifiers (`AKIA…`) |
| API Key | Key-value pairs whose key includes `api_key` or `apikey` |
| Private Key | PEM private key headers |
| OAuth Token | Key-value pairs whose key includes `oauth` or `token` |
| Password | Key-value pairs whose key includes `password`, `passwd`, or `pwd` |
| Secret Key | Key-value pairs whose key includes `secret_key` or `secretkey` |

Files whose path contains `.env.example` or `.env.template` are entirely excluded from
secret scanning; these files are designed to hold placeholder values.

### 3.5 Best Practice Checks

Two format-specific checks are applied:

| Type | Issue | Description |
|---|---|---|
| `json` | Comments present | Strict JSON does not support comments; presence of `//` or `/* … */` is flagged |
| `json` | Trailing commas | Trailing commas before `}` or `]` are invalid in strict JSON |
| `yaml` | `yes`/`no` booleans | `yes` and `no` are deprecated boolean literals; `true`/`false` should be used instead |

### 3.6 AI Response Quality Gate

After the primary AI call, the response text is checked to ensure it adequately covers the
files it was asked to review. A response is considered adequate when at least 30% of the
analysed file names (matched by basename or any trailing path segment) appear in the
response text. Responses below this threshold are logged as low quality but do not cause
the step to fail or the backlog write to be suppressed.

### 3.7 File-Content-Hash Guard

Step 04 uses a **file-content-hash guard** instead of TTL-based caching for its AI calls.
Before each AI API request, the guard computes a SHA256 hash of the current file contents
(sorted for order-independence, one `"${relativePath}:${content}"` entry per file). The
hash is compared against a value stored in `.ai_workflow/.ai_cache/step_hashes.json` from
the previous run:

- **Hash unchanged** → the stored AI response is returned immediately; no API call is made.
- **Hash changed (or no prior entry)** → the AI function is invoked, and the new hash +
  response are persisted for the next run.

Unlike TTL-based caching (`withCache`), the guard response persists indefinitely — the API
is called only when the input files actually change. Both AI phases (AI-1 and AI-2) share
the same `fileHashEntries` but use different step identifiers (`step_04` and
`step_04_quality`) so their responses are stored and invalidated independently.

---

## 4. Constructor Dependencies

Step 04 receives all external collaborators at construction time via a single options map.
Each dependency falls back to a freshly instantiated default when absent.

| Dependency key | Role | Behaviour when absent |
|---|---|---|
| `fileOps` | Reads file contents and executes glob patterns | Falls back to a new file-operations instance |
| `backlog` | Writes step summaries to the workflow artifact store | Falls back to a new backlog instance |
| `gitOps` | Queries git for the list of modified files | Falls back to a new git-automation instance |
| `aiHelper` | Executes AI API requests | Falls back to a new AI helper instance |
| `aiCache` | Caches and retrieves AI responses by key | Falls back to a new AI cache instance |
| `techStack` | Detects the project's primary language and active frameworks | Falls back to a new tech-stack detector instance |
| `promptsDir` | Directory path for AI prompt templates | Passed to the AI helper; `null` when absent |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the root of the project under analysis |
| `options.projectKind` | Optional project-kind hint. When present, overrides tech-stack detection for AI prompt context. |

### 5.2 Execution Phases

Step 04 executes the following phases in order:

#### Phase 1 — Discover Configuration Files

Apply the two-strategy discovery process (§3.2) to `projectRoot`. If no configuration
files are found, log the absence and skip immediately:

```
{ success: true, skipped: true, reason: 'no_config_files' }
```

No further phases execute.

#### Phase 2 — Syntax Validation

For each discovered file, determine its configuration type (§3.1) and apply the
appropriate syntax check (§3.3). Collect all syntax errors across all files. Log the total
error count, and log each individual error with its file and line number.

#### Phase 3 — Secret Scanning

For each discovered file, apply the secret pattern scan (§3.4). Collect all security
findings across all files. Log the total finding count.

#### Phase 4 — Best Practice Checks

For each discovered file, apply the best-practice rules (§3.5). Collect all issues across
all files. Log the total issue count.

#### Phase 5 — Report Generation

Assemble the counts from Phases 2–4 into a results record (see §6.1). Generate a
formatted Markdown report and write it to the workflow backlog under step `4`, section
`"Configuration Validation"`. The report includes: summary counts, syntax error details
(up to 10, with file and line), security finding details (up to 10, with type and a line
preview), and best-practice issues (up to 5).

#### Phase AI-1 — Primary AI Analysis (Conditional)

Runs when the AI helper initialises successfully.

1. **Initialise the response cache.** Call `aiCache.init()` before any cache call.

2. **Detect tech stack.** Query the tech-stack detector for the primary language and active
   frameworks. Build a comma-separated tech stack summary string. If `options.projectKind`
   is provided, use it in place of the detected language for the `project_kind` variable.

3. **Build prompt-safe file entries and partitions.** For each discovered file, read its
   content and produce both a labelled fenced block (for prompts) and a
   `"${relativePath}:${content}"` hash-entry string (for the file-change guard, §3.7).
   Generated lockfiles may be summarized before partitioning, oversized files may be split
   into `(part X/Y)` entries, and unreadable files are silently skipped. Generated workflow
   helper bundles such as `.workflow_core/config/ai_helpers.yaml` are not analysed directly
   in prompt slices; instead, Step 04 substitutes `.workflow_core/.workflow-config.yaml`
   and `.workflow-config.yaml` as the authoritative validation/reporting context for that
   artifact. (See §3.2 of the AI Prompt Contract.)

4. **Build and execute partition prompts.** Load the `configuration_specialist_prompt`
   template from the AI helpers YAML configuration and populate the per-partition
   variables (`project_name`, `partition_header`, `config_files_list`,
   `config_files_content`, `config_count`, `project_kind`, `tech_stack`). When the
   template is unavailable, fall back to a built-in structured prompt embedding the issue
   counts and the current partition content block directly. Each partition runs through
   `aiCache.withFileChangeGuard('step_04_p{n}', fileHashEntries, fn)` with persona
   `devops_engineer`.

5. **Validate each partition response.** Apply the evidence-handling guard and quality gate
   (§3.6). Log warnings when a partition response overclaims against partial evidence or
   mentions too few files.

6. **Run whole-scope synthesis when partitioned.** When the primary review required more
   than one partition, execute an additional `devops_engineer` prompt that sees the full
   file list, bounded whole-scope excerpts, and all per-partition findings. This synthesis
   pass is responsible for detecting contradictions and reconciliations that only become
   visible across partitions.

7. **Write enriched backlog entry.** If AI output was produced, append the partition
   analysis and any whole-scope synthesis under an `"AI Recommendations"` heading.

#### Phase AI-2 — Quality Review (Conditional, Supplementary)

Runs when the AI helper initialised successfully and the `quality_prompt` YAML template is
available.

1. **Build prompt-safe supplementary partitions.** Populate `quality_prompt` for the full
   discovered configuration scope rather than only the first 10 files. When necessary,
   split the supplementary review into multiple prompt-safe partitions and preserve any
   `(part X/Y)` labels in the rendered scope.

2. **Execute via file-change guard.** Call
   `aiCache.withFileChangeGuard('step_04_quality_p{n}', fileHashEntries, fn)` for each
   supplementary partition. When the hash is unchanged, the cached quality response is
   returned without an API call (§3.7). Persona: `code_quality_analyst`.

3. **Append quality review.** If responses were produced, append the partitioned
   supplementary review under a `"Quality Review"` heading.

When the AI helper is unavailable, both AI phases are skipped with a warning logged.

### 5.3 Skip Conditions

| Reason | Condition |
|---|---|
| `no_config_files` | No configuration files discovered after applying both discovery strategies |

The skip result is `{ success: true, skipped: true, reason: 'no_config_files' }`.

---

## 6. Data Shapes

### 6.1 StepResult (success, not skipped)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without a fatal error |
| `filesChecked` | Integer | Count of configuration files discovered and processed |
| `syntaxErrors` | SyntaxError[] | Syntax errors from Phase 2 (§6.2) |
| `securityFindings` | SecurityFinding[] | Secret detections from Phase 3 (§6.3) |
| `bestPracticeIssues` | BestPracticeIssue[] | Best-practice violations from Phase 4 (§6.4) |

### 6.2 SyntaxError

| Field | Type | Description |
|---|---|---|
| `file` | String | Absolute path to the file |
| `type` | String | Configuration type identifier (e.g. `'json'`, `'yaml'`) |
| `error` | String | Parse error message (JSON-type errors) |
| `line` | Integer or absent | Line number of the error, when available |
| `column` | Integer or absent | Column number of the error, when available |
| `issues` | Object[] or absent | YAML-style issue list (used for YAML errors instead of `error`) |

### 6.3 SecurityFinding

| Field | Type | Description |
|---|---|---|
| `type` | `'security_risk'` | Always this value |
| `secretType` | String | Human-readable pattern label (e.g. `'AWS Key'`, `'Password'`) |
| `line` | Integer | Line number where the pattern was matched |
| `file` | String | Absolute path to the file |
| `preview` | String | First 50 characters of the matching line (truncated with `…` when longer) |

### 6.4 BestPracticeIssue

| Field | Type | Description |
|---|---|---|
| `type` | String | Issue type identifier: `'syntax_error'` or `'invalid_value'` |
| `message` | String | Human-readable description of the issue |

### 6.5 StepResult (skip)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `skipped` | `true` | — |
| `reason` | `'no_config_files'` | No configuration files were discovered |

---

## 7. Constraints and Rules

1. The step **must** prefer git-modified files over a glob scan when git is available and
   returns at least one matching configuration file. Performing a glob scan when git has
   already identified the relevant files wastes I/O and may include irrelevant unchanged
   files.

2. Files located under any excluded directory (§3.2) **must** be filtered out regardless
   of which discovery strategy is used.

3. `.env.example` and `.env.template` files **must** be excluded from secret scanning
   (§3.4). These files exist specifically to document the expected shape of secrets without
   holding real values.

4. Each file read during Phase AI-1 prompt construction **must** be wrapped individually
   in an error handler. A single unreadable file must not prevent the prompt from being
   built for the remaining files. (Ref: AI Prompt Contract §3.4.)

5. The AI response cache **must** be initialised (via `aiCache.init()`) before the first
   `withFileChangeGuard` call. Failure to do so causes a runtime error even when no prior
   hash entry has been written.

6. Every AI phase (partitioned primary review, whole-scope synthesis, and partitioned
   supplementary quality review) **must** use `withFileChangeGuard`. Cache keys must remain
   distinct per phase/partition so that a change to any relevant discovered file invalidates
   the right cached response without conflating primary, synthesis, and quality outputs.
   Hashes are persisted in `.ai_workflow/.ai_cache/step_hashes.json`; entries survive
   process restarts and have no TTL — they are replaced only when the hash changes.

7. The primary review and whole-scope synthesis use the `devops_engineer` persona. This
   persona covers the full
   scope of the `configuration_specialist_prompt` template (JSON/YAML/TOML, CI/CD, Docker,
   environment configuration). Using a narrower persona (e.g. `security_expert`) would
   create a mismatch with the broad-scope prompt content.

8. The supplementary quality review call uses the `code_quality_analyst` persona, which
   aligns with the `quality_prompt` template's focus on anti-patterns, best practices, and
   maintainability.

9. The quality gate (§3.6) **must not** cause the step to fail or suppress a backlog write.
   Low response quality is informational only.

10. **Exception propagation deviation.** Callers of `execute` **must** wrap the call in
    error handling. Step 04 does not guarantee that exceptions are converted to
    `{ success: false }` results; unhandled exceptions are re-thrown.
