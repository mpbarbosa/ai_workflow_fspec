# ai_workflow_fspec — Document Index

This repository contains the **language-independent functional specification** for the
AI-assisted workflow programming model. All specification documents are in `docs/`.

---

## Repository Layout

```
ai_workflow_fspec/
├── INDEX.md                                          ← this file
├── README.md                                         ← project overview
├── LICENSE                                           ← MIT
└── docs/
    ├── step_contract.md                              ← step execution contract (core spec)
    ├── ai_prompt_contract.md                         ← AI prompt hygiene contract
    ├── step_00_pre_analysis.md                       ← step 00: pre-analysis
    ├── step_01_documentation_analysis.md             ← step 01: documentation analysis
    ├── step_02_consistency_analysis.md               ← step 02: doc/code consistency analysis
    ├── step_02_5_documentation_optimization.md       ← step 02_5: documentation optimization
    ├── step_03_script_references.md                  ← step 03: script reference validation
    ├── step_04_config_validation.md                  ← step 04: configuration validation
    ├── step_05_directory_analysis.md                 ← step 05: directory structure analysis
    ├── step_06_test_review.md                        ← step 06: test review
    ├── step_07_test_generation.md                    ← step 07: test generation
    ├── step_08_test_execution.md                     ← step 08: test execution
    ├── step_09_dependency_validation.md              ← step 09: dependency validation
    ├── step_0b_bootstrap_documentation.md            ← step 0b: bootstrap documentation
    ├── step_0f_commit_artifacts.md                   ← step 0f: commit artifacts
    ├── step_10_code_quality.md                       ← step 10: code quality analysis
    ├── step_11_context_analysis.md                   ← step 11: context health analysis
    ├── step_11_5_aws_lbs_validation.md               ← step 11_5: AWS LBS validation
    ├── step_11_6_aws_serverless_review.md            ← step 11_6: AWS serverless review
    ├── step_12_git_finalization.md                   ← step 12: git finalization
    ├── step_13_markdown_lint.md                      ← step 13: markdown lint
    ├── step_14_prompt_engineering.md                 ← step 14: prompt engineering review
    ├── step_15_ux_analysis.md                        ← step 15: UX analysis
    ├── step_16_version_update.md                     ← step 16: version update
    ├── step_17_workflow_summary.md                   ← step 17: workflow summary
    ├── step_18_debugging.md                          ← step 18: debugging analysis
    └── step_19_typescript_review.md                  ← step 19: TypeScript review
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
| `step_02` | Doc/Code Consistency Analysis | PROJECT · AI | [`docs/step_02_consistency_analysis.md`](docs/step_02_consistency_analysis.md) |
| `step_02_5` | Documentation Optimization | PROJECT | [`docs/step_02_5_documentation_optimization.md`](docs/step_02_5_documentation_optimization.md) |
| `step_03` | Script Reference Validation | PROJECT · AI | [`docs/step_03_script_references.md`](docs/step_03_script_references.md) |
| `step_04` | Configuration Validation | PROJECT · AI | [`docs/step_04_config_validation.md`](docs/step_04_config_validation.md) |
| `step_05` | Directory Structure Analysis | PROJECT · AI | [`docs/step_05_directory_analysis.md`](docs/step_05_directory_analysis.md) |
| `step_06` | Test Review | CONTEXT · AI | [`docs/step_06_test_review.md`](docs/step_06_test_review.md) |
| `step_07` | Test Generation | PROJECT · AI | [`docs/step_07_test_generation.md`](docs/step_07_test_generation.md) |
| `step_08` | Test Execution | PROJECT · AI | [`docs/step_08_test_execution.md`](docs/step_08_test_execution.md) |
| `step_09` | Dependency Validation | PROJECT · AI | [`docs/step_09_dependency_validation.md`](docs/step_09_dependency_validation.md) |
| `step_0b` | Bootstrap Documentation | CONTEXT · AI | [`docs/step_0b_bootstrap_documentation.md`](docs/step_0b_bootstrap_documentation.md) |
| `step_0f` | Commit Artifacts | PROJECT | [`docs/step_0f_commit_artifacts.md`](docs/step_0f_commit_artifacts.md) |
| `step_10` | Code Quality Analysis | PROJECT · AI | [`docs/step_10_code_quality.md`](docs/step_10_code_quality.md) |
| `step_11` | Context Health Analysis | CONTEXT · AI | [`docs/step_11_context_analysis.md`](docs/step_11_context_analysis.md) |
| `step_11_5` | AWS LBS Validation | PROJECT · AI | [`docs/step_11_5_aws_lbs_validation.md`](docs/step_11_5_aws_lbs_validation.md) |
| `step_11_6` | AWS Serverless Review | PROJECT · AI | [`docs/step_11_6_aws_serverless_review.md`](docs/step_11_6_aws_serverless_review.md) |
| `step_12` | Git Finalization | CONTEXT · AI | [`docs/step_12_git_finalization.md`](docs/step_12_git_finalization.md) |
| `step_13` | Markdown Lint | CONTEXT · AI | [`docs/step_13_markdown_lint.md`](docs/step_13_markdown_lint.md) |
| `step_14` | Prompt Engineering Review | CONTEXT · AI | [`docs/step_14_prompt_engineering.md`](docs/step_14_prompt_engineering.md) |
| `step_15` | UX Analysis | CONTEXT · AI | [`docs/step_15_ux_analysis.md`](docs/step_15_ux_analysis.md) |
| `step_16` | Version Update | CONTEXT · AI | [`docs/step_16_version_update.md`](docs/step_16_version_update.md) |
| `step_17` | Workflow Summary | CONTEXT | [`docs/step_17_workflow_summary.md`](docs/step_17_workflow_summary.md) |
| `step_18` | Debugging Analysis | ANALYSIS · AI | [`docs/step_18_debugging.md`](docs/step_18_debugging.md) |
| `step_19` | TypeScript Review | ANALYSIS · AI | [`docs/step_19_typescript_review.md`](docs/step_19_typescript_review.md) |

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
