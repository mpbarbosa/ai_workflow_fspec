# Roadmap: Progress Quality-Evaluation Prompts

## Problem Statement

The workflow contains nine AI-integrated steps that evaluate project quality. All are at
**Draft** status. Several have documented deviations from the step and AI prompt contracts,
and the current quality-evaluation prompts vary significantly in rigour and consistency.

This roadmap drives those steps from their current Draft state to well-specified,
contract-conformant, and progressively richer quality evaluators.

## Steps in Scope (Quality-Evaluation)

| Step | Name | Kind | Key Issues |
|------|------|------|-----------|
| `step_01` | Documentation Analysis | PROJECT·AI | Error handling deviation |
| `step_02` | Doc/Code Consistency | PROJECT·AI | Partial prompt contract (paths only, no file contents) |
| `step_06` | Test Review | CONTEXT·AI | Error handling deviation; global logger deviation |
| `step_08` | Test Execution | PROJECT·AI | Error handling deviation |
| `step_10` | Code Quality Analysis | PROJECT·AI | Error handling deviation |
| `step_14` | Prompt Engineering Review | CONTEXT·AI | Error handling deviation; shallow scoring model |
| `step_18` | Debugging Analysis | ANALYSIS·AI | ANALYSIS kind not formalised in `step_contract.md` |
| `step_19` | TypeScript Review | ANALYSIS·AI | Reference implementation — maintain as canonical |

---

## Phase 1 — Foundation: Fix the Contracts

Fix the upstream contract documents before touching any step spec. All Phase 2+ work
depends on clean contracts.

### 1.1 Formalise the ANALYSIS step kind in `step_contract.md`

- Add §2.3 (ANALYSIS) that mirrors the PROJECT/CONTEXT pattern.
- Define the class-level `stepKind` requirement for ANALYSIS steps.
- Clarify that orchestrator implementations must read kind from `STEP_DEFINITION.kind`
  until implementations migrate.
- Remove the "specification gap" note from `step_18.md` and `step_19.md` §2.

### 1.2 Add error-handling compliance table to `step_contract.md`

- Enumerate all steps that re-throw and mark them as non-conforming.
- Add a §7 "Known Deviations" section so the contract itself is the authoritative tracker.
- Define the remediation target: each step spec must either resolve the deviation or
  formally record it with a tracking reference.

### 1.3 Update `ai_prompt_contract.md` scope table

- Correct the step list in §2 (several entries map to wrong steps, e.g. `step_05` /
  `step_09` labels appear inconsistent with the INDEX).
- Add `step_02`'s partial-compliance status explicitly.

---

## Phase 2 — Prompt Contract Compliance

Bring every quality-evaluation step to full AI Prompt Contract conformance.

### 2.1 `step_02` — Inject file contents (not just paths)

Current state: `step_02` injects file paths but not file contents into the AI prompt.
This is the same root-cause class as the `step_01` hallucination event documented in
`ai_prompt_contract.md §1`.

Actions in the spec:

- Update §5 (Execute Behaviour) to require `buildFileContentMap` / `buildFileContentBlock`
  injection, matching the `step_19` reference pattern.
- Update §2 conformance table to "Yes" once resolved.
- Add a constraint in §7 mirroring `step_19` §7 rule 3.

### 2.2 Verify and document anti-hallucination guards for all quality steps

For each quality-evaluation step, confirm whether the AI prompt template includes the
recommended guard text from `ai_prompt_contract.md §5.2`. Document findings in §7 of
each spec. Add a constraint requiring the guard where absent.

Priority order: `step_02` (highest risk, partial compliance), `step_06`, `step_08`,
`step_01`.

### 2.3 Resolve or formally accept `aiCache` deviation in `step_15`

`step_15` has no `aiCache` (every AI call is live). For quality-evaluation steps this
is low risk — `step_15` is UX analysis, not in the primary quality-eval set — but the
deviation should be:

- Documented as a deliberate design choice in `step_15` §2, or
- Resolved by adding `aiCache` with an appropriate cache key.

---

## Phase 3 — Scoring Model Maturity

Improve the quality metrics used by steps that score or rate project artefacts.

### 3.1 `step_14` — Deepen the prompt quality scoring criteria

Current scoring is purely heuristic: character counts and action-verb presence.
This misses meaningful quality signals.

Extend the scoring model (§3.4) to include:

| Criterion | Weight | Condition for full marks |
|---|---|---|
| **Grounding** | 2 | Prompt includes an explicit instruction to work only from injected context (anti-hallucination guard) |
| **Output format** | 1 | Approach section specifies a structured output format (JSON, Markdown table, numbered list, etc.) |
| **Failure modes** | 1 | Prompt tells the model what to say when information is insufficient |
| **Scope constraint** | 2 | Prompt bounds the analysis to specific files/artefacts rather than asking for general advice |

Update §3.4 weights table and §3.5 rating thresholds if the maximum weighted sum changes.

### 3.2 `step_10` — Add severity weighting to quality rating bands

The four-band system (excellent / good / moderate / poor) in §3.4 is based on
issue-per-file rate only. Add a secondary signal: per-file **error severity weighting**
(errors count more than warnings). Document the weighting formula in §3.4 and update
§6.3 report structure to include the weighted rating alongside the raw rating.

### 3.3 `step_06` — Formalise a test quality score

`step_06` currently has no numeric scoring model analogous to `step_10`'s issue rate or
`step_14`'s quality score. Add a test quality score to §3 covering:

- **Coverage signal**: presence and value of a coverage report.
- **Test-to-source ratio**: test file count vs source file count.
- **AI-rated quality**: map the AI review output to a 0–100 score bucket
  (excellent / good / needs-improvement / poor) for inclusion in the `StepResult`.

---

## Phase 4 — Coverage Extension

Add quality dimensions not currently addressed by any step.

### 4.1 New spec: `step_20` — Security Review

Gap: no step currently evaluates security anti-patterns (hardcoded secrets, injection
vectors, insecure dependencies).

Define a new ANALYSIS step spec following the `step_18` / `step_19` pattern:

- Persona: `security_analyst`
- Heuristic pre-scan: flag obvious patterns (hardcoded tokens, `eval`, `exec`,
  SQL string interpolation) before the AI call.
- AI prompt: file content injection following the `step_19` reference pattern;
  anti-hallucination guard required.
- Dependency: `step_09` (dependency validation) as a preceding step.

### 4.2 New spec: `step_21` — API Contract Validation

Gap: no step validates that the API surface matches its documentation (OpenAPI / schema
drift).

Define a PROJECT step spec:

- Persona: `api_contract_reviewer`
- Detect OpenAPI / JSON Schema / GraphQL schema files in the project.
- Inject schema content and relevant route handler source files.
- Report drift between the declared and implemented surface.

### 4.3 Extend `step_18` persona selection

`step_18` currently selects a debugging persona based on code-pattern heuristics
(observer / async / data-structure). Add a fourth persona path:

- **Memory-leak analysis** persona when `setInterval`, `addEventListener`, or
  `WeakMap` / `WeakRef` patterns dominate the sampled source.

---

## Phase 5 — Spec Status Progression

Move documents from Draft → Review → Approved.

### 5.1 Review gate criteria

A spec may move to **Review** when:

- All documented deviations are either resolved or formally accepted with a tracking
  reference.
- The AI Prompt Contract conformance row in §2 is accurate.
- All §7 constraints are testable propositions (not implementation notes).

A spec may move to **Approved** when:

- It has been reviewed by a second contributor.
- No open tracking references remain in §7.
- The spec version is bumped to reflect the review cycle.

### 5.2 Recommended review order

1. `step_contract.md` and `ai_prompt_contract.md` (blocking for all others)
2. `step_19` (reference implementation — establish the gold standard)
3. `step_10`, `step_14` (most widely referenced quality-eval steps)
4. `step_01`, `step_02`, `step_06` (high deviation count)
5. `step_08`, `step_18` (lower complexity)

---

## Work Items

| ID | Title | Phase | Depends on |
|----|-------|-------|-----------|
| P1-analysis-kind | Formalise ANALYSIS kind in `step_contract.md` | 1 | — |
| P1-error-table | Add error-handling deviation table to `step_contract.md` | 1 | — |
| P1-prompt-contract-scope | Fix `ai_prompt_contract.md` §2 scope table | 1 | — |
| P2-step02-injection | Update `step_02` spec for full file content injection | 2 | P1-prompt-contract-scope |
| P2-hallucination-audit | Audit anti-hallucination guards in steps 01, 06, 08 | 2 | P1-prompt-contract-scope |
| P2-step15-cache | Resolve `aiCache` deviation in `step_15` | 2 | P1-prompt-contract-scope |
| P3-step14-scoring | Deepen `step_14` prompt quality scoring criteria | 3 | P1-error-table |
| P3-step10-severity | Add severity weighting to `step_10` quality rating | 3 | — |
| P3-step06-score | Add test quality score model to `step_06` | 3 | — |
| P4-step20-security | New spec: `step_20` Security Review | 4 | P1-analysis-kind |
| P4-step21-api | New spec: `step_21` API Contract Validation | 4 | P1-analysis-kind |
| P4-step18-persona | Extend `step_18` persona selection (memory-leak path) | 4 | — |
| P5-review-contracts | Review gate: `step_contract` + `ai_prompt_contract` | 5 | P1-\*, P2-\* |
| P5-review-step19 | Review gate: `step_19` (gold standard) | 5 | P5-review-contracts |
| P5-review-quality-steps | Review gate: `step_10`, `step_14` | 5 | P5-review-step19 |
| P5-review-remaining | Review gate: `step_01`, `step_02`, `step_06`, `step_08`, `step_18` | 5 | P5-review-quality-steps |
