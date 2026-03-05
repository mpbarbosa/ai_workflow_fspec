# Step 09: Dependency Validation — Functional Specification

**Step identifier:** `step_09`  
**Step name:** Dependency Validation  
**Step kind:** `PROJECT` (see [Step Contract §2.1](./step_contract.md))  
**AI-integrated:** Yes (see [AI Prompt Contract](./ai_prompt_contract.md))  
**Version:** 2.0.0  
**Status:** Draft  
**Related:** [`step_contract.md`](./step_contract.md) · [`ai_prompt_contract.md`](./ai_prompt_contract.md)

---

## 1. Purpose

Step 09 validates the dependency health of the project by detecting the package
ecosystem, running a security audit, identifying outdated packages, and enriching the
findings with AI-powered analysis using the `dependency_analyst` persona.

Step 09 answers three questions:

1. **How many dependencies does the project have?** — production, development, and
   peer dependencies are counted from the language-specific manifest file.
2. **Are any dependencies vulnerable?** — a language-appropriate security audit tool
   is invoked and its output is parsed into a structured vulnerability summary.
3. **Are any dependencies outdated?** — a language-appropriate outdated-packages check
   is invoked and the results are presented as a list of packages with current and
   latest versions.

The AI is then given the combined findings to produce a security risk assessment,
prioritised remediation steps, and maintenance recommendations. For JavaScript and
TypeScript projects, a second AI call reviews the actual `package.json` from the
perspective of a JavaScript developer.

---

## 2. Step Kind Conformance

Step 09 conforms to the **PROJECT** step kind as defined in `step_contract.md §2.1`.

| Property | Value |
|---|---|
| Kind identifier | `"project"` |
| Execute signature | `execute(projectRoot, options?) → Promise<StepResult>` |
| Can be skipped | **Yes** — skips for unsupported languages or when no dependency files are found |
| AI-integrated | **Yes** — makes up to two AI calls via `aiHelper` and `aiCache` |
| Conforms to AI Prompt Contract | **Yes** — for the second AI call (JavaScript developer analysis), the actual `package.json` file content is read and injected into the prompt; no prohibited prompt techniques are used |

**Error handling deviation.** Step 09 does **not** conform to the step contract rule
requiring that `execute` never throw. When an unhandled error occurs in the main
execution body, the `catch` block logs the error and **re-throws** it rather than
returning `{ success: false, error: … }`. AI-phase errors are caught separately and
treated as non-fatal warnings. See §7 for details.

---

## 3. Domain Concepts

### 3.1 Supported Languages

Dependency validation is supported for the following languages. Each has an associated
set of manifest files, a security audit command, an outdated-package check command,
and a set of recommended fix commands.

| Language | Manifest Files | Audit Command | Outdated Command |
|---|---|---|---|
| `javascript` / `typescript` | `package.json`, `package-lock.json`, `yarn.lock` | `npm audit --json` | `npm outdated --json` |
| `python` | `requirements.txt`, `setup.py`, `pyproject.toml`, `Pipfile` | `pip-audit --format json` | `pip list --outdated --format json` |
| `go` | `go.mod`, `go.sum` | `govulncheck -json ./...` | `go list -u -m -json all` |
| `java` | `pom.xml`, `build.gradle` | `mvn org.owasp:dependency-check-maven:check -Dformat=JSON -q` | `mvn versions:display-dependency-updates -q` |
| `ruby` | `Gemfile`, `Gemfile.lock` | `bundle audit --format json` | `bundle outdated --format json` |
| `rust` | `Cargo.toml`, `Cargo.lock` | `cargo audit --json` | `cargo outdated --format json` |

**Unsupported languages:** `bash`, `shell`, and `sh` do not have package managers and
are explicitly excluded. When the detected language is one of these, the step skips
with a formatted report and returns `{ success: true, skipped: true }`.

### 3.2 Dependency Counts

Dependency counts are parsed from the primary manifest file for the detected language:

| Language | Source | Fields |
|---|---|---|
| javascript / typescript | `package.json` | `dependencies` (production), `devDependencies` (development), `peerDependencies` (peer) |
| python | `requirements.txt` | Non-comment, non-option lines (all treated as production) |
| go | `go.mod` | Lines inside `require(…)` blocks, plus single-line `require` statements |
| java | `pom.xml` | Count of `<dependency>` tags |
| ruby | `Gemfile` | Lines beginning with `gem ` |
| rust | `Cargo.toml` | Lines with `=` in `[dependencies]` and `[dev-dependencies]` sections |

### 3.3 Vulnerability Severity Levels

Five severity levels, in descending severity order: `critical`, `high`, `moderate`,
`low`, `info`. The overall severity of the audit result is the highest level for which
the count is greater than zero.

The step reports `hasSecurityIssues: true` when the `critical` or `high` count is
greater than zero.

### 3.4 Dependency Cache

A persistent cache (`DependencyCache`) is used to avoid re-running the security audit
and outdated-package check on every run when the dependency set has not changed.

The cache key is derived from the set of production and development dependency names
and the cache type (`AUDIT` or `OUTDATED`). A hit on the cache returns the stored
result without invoking the external tool. Results are stored after each successful
external invocation.

Cache initialisation is attempted before use; if it fails, the step proceeds without
caching.

### 3.5 AI Call 1 — Dependency Analysis

The primary AI call analyses the full dependency health report. It uses the
`dependency_analyst` persona and the `step8_dependencies_prompt` YAML template,
populated with:

- Project name, description, primary language.
- Package manager and its version (if available).
- Change scope and modified file count (from step options).
- Dependency counts (total, production, development).
- Vulnerability summary (total, critical, high, moderate, low).
- Outdated package list (up to 20 packages: name, current, latest).
- The linter-style report formatted in Phase 6 (first 1,500 characters).
- Lists of production and development package names (up to 30 each).

If the YAML template cannot be loaded, the step falls back to a hardcoded structured
prompt with a security-analyst role, a task describing the vulnerability and outdated
counts, and a three-point approach requesting a risk assessment, remediation steps,
and maintenance recommendations.

The cache key for this call is `step_09|{language}|vuln{total}|outdated{count}`.

### 3.6 AI Call 2 — JavaScript Developer Analysis (Conditional)

A second AI call is made only when the detected language is `javascript` or
`typescript`. It reviews the project's `package.json` from the perspective of a
JavaScript developer, using the `javascript_developer_prompt` YAML template.

The **actual content of `package.json`** is read and injected into this prompt as the
`current_package_json` variable. This satisfies the
[AI Prompt Contract §3](./ai_prompt_contract.md) requirement that file contents be
injected rather than referenced by name.

If `package.json` cannot be read, the placeholder value `"Not found"` is used. The
prompt is still submitted in that case, with a warning implicitly conveyed by the
missing content.

The cache key for this call is `step_09_js|{projectRoot}|vuln{total}|outdated{count}`.

Both AI calls use the `dependency_analyst` persona.

---

## 4. Constructor Dependencies

Step 09 receives all collaborators via a single options map at construction time.

| Dependency key | Role | When absent |
|---|---|---|
| `executor` | Executes audit and outdated-check commands in a subprocess | Falls back to the global executor module |
| `fileOps` | Reads manifest files and the AI helpers YAML | Falls back to a new `FileOperations` instance |
| `backlog` | Writes step summary to the workflow artifact store | Falls back to a new `Backlog` instance |
| `techStack` | Detects the primary language of the project | Falls back to a new `TechStackDetector` instance |
| `aiHelper` | Executes AI API requests | Falls back to a new `AiHelper` instance |
| `aiCache` | Caches and retrieves AI responses by cache key | Falls back to a new `AiCache` instance |
| `depCache` | Caches audit and outdated results to avoid repeated tool invocations | Falls back to a new `DependencyCache` instance rooted at the process working directory |
| `promptsDir` | Directory path for AI prompt templates | Passed to `AiHelper` constructor; defaults to `null` |

---

## 5. Execute Behaviour

### 5.1 Inputs

| Parameter | Description |
|---|---|
| `projectRoot` | Absolute path to the root of the project under analysis |
| `options` | Optional map of additional orchestrator-supplied hints |

Relevant `options` fields:

| Field | Description |
|---|---|
| `projectDescription` | Human-readable project description. Included in the AI prompt. |
| `changeScope` | Change scope string from Step 00. Included in the AI prompt. |
| `modifiedCount` | Count of modified files. Included in the AI prompt. |
| `projectType` / `projectKind` | Project kind identifier. Used in the JavaScript developer prompt. |

### 5.2 Execution Phases

Step 09 runs seven sequential phases:

#### Phase 1 — Detect Language

Run tech-stack detection on `projectRoot`. Use the first detected language. If
detection fails or returns no languages, fall back to `javascript`.

If the detected language is `bash`, `shell`, or `sh`, the step immediately:
1. Formats and saves a skipped dependency report to the backlog.
2. Returns `{ success: true, language, skipped: true }`.

#### Phase 2 — Find Dependency Files

Check which manifest files from §3.1 exist under `projectRoot` for the detected
language. If none exist, save a skipped report to the backlog and return
`{ success: true, language, skipped: true, message: 'No dependency files found' }`.

#### Phase 3 — Parse Dependency Counts

Read the primary manifest file and parse it using the language-appropriate parser
(§3.2). The result includes total, production, development, and peer counts, plus the
raw dependency and devDependency maps for use in AI prompt construction.

#### Phase 3b — Initialise Dependency Cache

Attempt to initialise the dependency cache. If initialisation fails, proceed without
caching. Generate the cache keys for the audit and outdated results using the
dependency maps and the cache type constants.

#### Phase 4 — Run Security Audit (with Cache)

Check the dependency cache for a stored audit result. On a cache hit, use the stored
result. On a cache miss, invoke the language-appropriate audit command as a subprocess
(60-second timeout). Parse the JSON output using the language-appropriate parser.

If the audit command exits with a non-zero code (which many audit tools do when
vulnerabilities are present), attempt to parse `stdout` anyway. If that also fails,
return an empty result `{ summary: null, packages: [] }`.

Store a successful result in the dependency cache.

#### Phase 5 — Check Outdated Packages (with Cache)

Check the dependency cache for a stored outdated result. On a cache hit, use the
stored result. On a cache miss, invoke the language-appropriate outdated-packages
command (60-second timeout). Parse the output using the language-appropriate parser.
If the command fails or produces no parseable output, return an empty list.

Store a successful result in the dependency cache.

#### Phase 6 — Generate Report and Save to Backlog

Assemble a `DependencyReport` record (§6.2) from the collected values. Format a
Markdown report covering dependency counts, vulnerability summary, outdated package
list, and fix recommendations. Save the report to the backlog under step `9`,
section `"Dependency Validation"`.

#### Phase 7 — AI-Powered Analysis (Optional)

This phase runs only when the AI helper can be initialised successfully.

1. Initialise the AI response cache.
2. Build and execute **AI Call 1** (§3.5): the primary dependency analysis. Use
   `aiCache.withCache` with the `step_09|…` cache key. If the YAML template loads
   successfully, use it; otherwise use the hardcoded fallback prompt.
3. For `javascript` and `typescript` projects only, build and execute **AI Call 2**
   (§3.6): the JavaScript developer analysis. Read `package.json` content for injection
   into the `current_package_json` prompt variable. Use `aiCache.withCache` with the
   `step_09_js|…` cache key.
4. If either AI call produces content, save an enriched report to the backlog,
   appending the AI content as additional sections after the base report.

Both AI calls use the `dependency_analyst` persona. All AI errors are caught and
logged as warnings; they do not cause the phase or the step to fail.

### 5.3 Skip Conditions

| Reason | Condition | Result shape |
|---|---|---|
| Unsupported language | Detected language is `bash`, `shell`, or `sh` | `{ success: true, language, skipped: true }` |
| No dependency files | None of the expected manifest files exist | `{ success: true, language, skipped: true, message: 'No dependency files found' }` |

---

## 6. Data Shapes

### 6.1 StepResult (success, full execution)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Step completed without fatal error |
| `hasSecurityIssues` | Boolean | `true` when critical or high vulnerability count > 0 |
| `language` | String | Detected language |
| `dependencyCounts` | Object | Parsed dependency counts (§6.2) |
| `vulnerabilities` | Object | Vulnerability summary and package list (§6.3) |
| `outdatedPackages` | Object[] | Outdated package list (§6.4) |
| `skipped` | `false` | Always `false` on the full execution path |

### 6.2 DependencyCounts

| Field | Type | Description |
|---|---|---|
| `total` | Integer | Total dependencies (production + development + peer) |
| `production` | Integer | Count of production/runtime dependencies |
| `development` | Integer | Count of development-only dependencies |
| `peer` | Integer | Count of peer dependencies |
| `dependencies` | Object | Raw production dependency map (name → version spec). Present for JavaScript/TypeScript only. |
| `devDependencies` | Object | Raw development dependency map (name → version spec). Present for JavaScript/TypeScript only. |

### 6.3 Vulnerabilities

| Field | Type | Description |
|---|---|---|
| `summary` | Object or null | `{ total, critical, high, moderate, low, info }`; null when the audit tool is unavailable for the language |
| `packages` | Object[] | Per-vulnerable-package list: `{ name, severity, via[] }` |

### 6.4 Outdated Package

| Field | Type | Description |
|---|---|---|
| `name` | String | Package name |
| `current` | String | Currently installed version |
| `wanted` | String | Version matching the declared range constraint |
| `latest` | String | Latest available version |
| `type` | String | Dependency type (e.g. `"dependencies"`) |

### 6.5 StepResult (skip)

| Field | Type | Description |
|---|---|---|
| `success` | `true` | — |
| `language` | String | Detected language |
| `skipped` | `true` | — |
| `message` | String | Present when skip reason is `'No dependency files found'` |

---

## 7. Constraints and Rules

1. **Error handling deviation.** Step 09 does **not** comply with the step contract
   rule (§5, rule 4) that forbids propagating exceptions from `execute`. Errors in the
   main execution body are re-thrown after logging. Callers **must** wrap calls to
   `execute` in a try/catch or equivalent. Errors that occur only in Phase 7 (AI
   analysis) are caught and treated as warnings; they do not cause a re-throw.

2. Audit and outdated-check tools frequently exit with a non-zero status code when
   they find issues. The step **must** attempt to parse `stdout` even when the command
   exits with a non-zero code. A non-zero exit code alone **must not** be treated as a
   tool failure.

3. The dependency cache **must** be keyed on the actual dependency names and versions,
   not on the project root path or a timestamp. Using a path-based key would cause
   cache misses whenever the project is moved, and using a timestamp-based key would
   negate the caching benefit entirely.

4. For JavaScript and TypeScript projects, the `package.json` file content **must** be
   read and injected into the JavaScript developer prompt (AI Call 2). The step **must
   not** instruct the model to fetch or infer the file's content. This requirement is
   mandated by the [AI Prompt Contract §3](./ai_prompt_contract.md).

5. Both AI calls use the same `dependency_analyst` persona but produce separate,
   independently cached results. The two results **must not** share a cache key.

6. The dependency cache `init()` call is best-effort. A cache initialisation failure
   **must not** cause the step to skip dependency validation. The step proceeds with
   all tool invocations and simply does not cache the results.

7. AI phase failures **must not** cause the step to return `success: false` or to
   re-throw. The base dependency report (without AI enrichment) is always a valid
   step outcome.

8. The backlog save in Phase 6 may be overwritten by the enriched save in Phase 7.
   This is intentional — the AI-enriched version supersedes the base report. If the AI
   phase is skipped entirely, the Phase 6 save remains as the final artifact.
