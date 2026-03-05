# ai_workflow_fspec — Document Index

This repository contains the **language-independent functional specification** for the
AI-assisted workflow programming model. All specification documents are in `docs/`.

---

## Repository Layout

```
ai_workflow_fspec/
├── INDEX.md                                    ← this file
├── README.md                                   ← project overview
├── LICENSE                                     ← MIT
└── docs/
    ├── step_contract.md                        ← step execution contract (core spec)
    ├── ai_prompt_contract.md                   ← AI prompt hygiene contract
    ├── step_00_pre_analysis.md                 ← step 00: pre-analysis specification
    └── step_01_documentation_analysis.md       ← step 01: documentation analysis specification
```

---

## Core Specifications

These documents define the foundational rules that every step and AI integration must
follow. Read them before any step-level specification.

| Document | Description |
|---|---|
| [`docs/step_contract.md`](docs/step_contract.md) | Step execution contract: step kinds (`PROJECT`, `CONTEXT`, `ANALYSIS`), orchestrator dispatch rules, `StepResult` and `WorkflowContext` data types, and the step implementation rules |
| [`docs/ai_prompt_contract.md`](docs/ai_prompt_contract.md) | AI prompt hygiene contract: file content injection requirements, prohibited prompt techniques (e.g. `@workspace`), anti-hallucination rules, `aiHelper` / `aiCache` dependency contracts |

---

## Step Specifications

One document per workflow step. Each document is language-agnostic and describes
behaviour, data shapes, and constraints — not implementation.

| Step | Name | Kind | Document |
|---|---|---|---|
| `step_00` | Pre-Analysis | PROJECT | [`docs/step_00_pre_analysis.md`](docs/step_00_pre_analysis.md) |
| `step_01` | Documentation Analysis | PROJECT · AI | [`docs/step_01_documentation_analysis.md`](docs/step_01_documentation_analysis.md) |

---

## Reading Order

For a new contributor or implementer, the recommended reading order is:

1. [`docs/step_contract.md`](docs/step_contract.md) — understand step kinds and the result contract
2. [`docs/ai_prompt_contract.md`](docs/ai_prompt_contract.md) — understand AI prompt rules (relevant only for AI-integrated steps)
3. The step specification for the step you are implementing or reviewing

---

## Document Status Legend

| Status | Meaning |
|---|---|
| Draft | Content is complete but not yet formally reviewed |
| Review | Under active review; may change |
| Approved | Reviewed, stable, changes require a spec revision |

All current documents are at **Draft** status.
