## INDEX

# ai_workflow_fspec — Document Index

This repository contains the **language-independent functional specification** for the
AI-assisted workflow programming model. All specification documents are in `docs/`.

---

## Repository Layout

```
ai_workflow_fspec/
├── INDEX.md                                          ← this file
├── README.md                                         ← project overview
├── ROADMAP.md                                        ← quality-evaluation prompt roadmap
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
| `step_02` | Doc/Code Consistency Analysis | PROJ

---

## GENERATION_SUMMARY

# Functional Spec Generation Summary

**Date:** 2026-03-05
**Scope:** All 25 workflow steps in `ai_workflow.js`

---

## Overview

Language-agnostic functional requirement specifications were generated for all 25 workflow
steps. Each document follows the format defined in `step_contract.md` and is free of
implementation-language references.

---

## Spec Files Generated

| Step | Name | Kind | AI | File |
|---|---|---|---|---|
| `step_00` | Pre-Analysis | PROJECT | — | `step_00_pre_analysis.md` |
| `step_01` | Documentation Analysis | PROJECT | `documentation_expert` | `step_01_documentation_analysis.md` |
| `step_02` | Doc/Code Consistency Analysis | PROJECT | `documentation_expert` | `step_02_consistency_analysis.md` |
| `step_02_5` | Documentation Optimization | PROJECT | opt-in (`aiAnalyzer`) | `step_02_5_documentation_optimization.md` |
| `step_03` | Script Reference Validation | PROJECT | `devops_engineer` | `step_03_script_references.md` |
| `step_04` | Configuration Validation | PROJECT | `devops_engineer` + `code_quality_analyst` | `step_04_config_validation.md` |
| `step_05` | Directory Structure Analysis | PROJECT | `architecture_reviewer` × 2 | `step_05_directory_analysis.md` |
| `step_06` | Test Review | CONTEXT | `test_engineer` (partition-rotate) | `step_06_test_review.md` |
| `step_07` | Test Generation | PROJECT | `test_engineer` | `step_07_test_generation.md` |
| `step_08` | Test Execution | PROJECT | `test_engineer` × 2 | `step_08_test_execution.md` |
| `step_09` | Dependency Validation | PROJECT | `dependency_analyst` × 2 | `step_09_dependency_validation.md` |
| `step_0b` | Bootstrap Documentation | CONTEXT | `technical_writer` | `step_0b_bootstrap_documentation.md` |
| `step_0f` | Commit Artifacts | PROJECT | — | `step_0f_commit_artifacts.md` |
| `step_10` | Code Quality Analysis | PROJECT | `code_quality_analyst` (partition-rotate) | `step_10_code_quality.md` |
| `step_11` | Context Health Analysis | CONTEXT | `health_check` | `step_11_context_analysis.md` |
| `step_11_5` | AWS LBS Validation | PROJECT | `devops_engineer` (opt-in) | `step_11_5_aws_lbs_validation.md` |
| `step_11_6` | AWS Serverless Review | PROJECT | `aws_serverless_engineer` (opt-in) | `step_11_6_aws_serverless_review.md` |
| `step_12` | Git Finalization | CONTEXT | `git_specialist` | `step_12_git_finalization.md` |
| `step_13` | Markdown Lint | CONTEXT | `technical_writer` (skipped when clean) | `step_13_markdown_lint.md` |
| `step_14` | Prompt Engineering Review | CONTEXT | `prompt_engineer` | `step_14_prompt_engineering.md` |
| `step_15` | UX Analysis | CONTEXT | `ux_analyst` (no `aiCache`) | `step_15_ux_analysis.md` |
| `step_16` | Version Update | CONTEXT | `devops_engineer` | `step_16_version_update.md` |
| `step_17` | Workflow Summary | CONTEXT | — | `step_17_workflow_summary.md` |
| `step_18` | Debugging Analysis | ANALYSIS | `debug_analyst` (STEP_DEFINITION-only kind) | `step_18_debugging.md` |
| `step_19` | TypeScript Review | ANALYSIS | `typescript_reviewer` (reference prompt impl) | `step_19_typescript_review.md` |

---

## Cross-Cutting Findings

### Deviations from `step_contract.md`

The following deviations from the step execution contract were found across the codebase
and are documented in the §2 and §7 sections of the relevant spec files.

| Deviation | Steps affected |
|---|---|
| Re-throws exceptions instead of returning `{ success: false, error }` | `step_01`, `step_02`, `step_03`, `step_05`, `step_06`, `step_07`, `step_08`, `step_09`, `step_10`, `step_11`, `step_12`, `step_13`, `step_14` |
| `ANALYSIS` kind declared only in `STEP_DEFINITION`, not as a class property | `step_18`, `step_19` |
| Uses global logger singleton instead of injected `this.logger` | `step_06`, `step_11`, `step_17` |
| No `aiCache` dependency (AI-integrated CONTEXT step) | `step_15` |
| Uses `aiAnalyzer` instead of `aiHelper`/`aiCache` — exempt from AI Prompt Contract | `step_02_5` |
| Skip represented as `{ noFiles: true }` instead of `{ ski
