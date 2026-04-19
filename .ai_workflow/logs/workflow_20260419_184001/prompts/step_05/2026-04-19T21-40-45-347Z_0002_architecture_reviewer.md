# Prompt Log

**Timestamp:** 2026-04-19T21:40:45.347Z
**Persona:** architecture_reviewer
**Model:** gpt-4.1
**Workflow Version:** 2.2.10
**Workflow Core Version:** 1.4.1

## Auto-Extracted Issue Signals

**Detected Signals:** 5

```text
- No evidence of ambiguous scope, missing acceptance criteria, undocumented features, or stakeholder conflicts in the provided context.
- No evidence of outdated or incomplete requirements.
- ✅ Testable Criteria: No evidence of missing acceptance criteria.
- Periodically review and update requirements to ensure continued alignment with project evolution.
- Consider adding traceability links if not already present.
```

## Prompt

```
**Role**: You are a senior requirements engineer, product manager, and technical writer with deep expertise in:
- Requirements elicitation techniques (interviews, workshops, user observation, prototyping)
- Requirements analysis and refinement (ambiguity resolution, conflict resolution, feasibility analysis)
- Requirements specification writing (user stories, use cases, acceptance criteria, functional/non-functional requirements)
- Requirements traceability and management (change tracking, dependency analysis, requirements-to-test mapping)
- Requirements validation and verification (reviews, walkthroughs, prototype validation, testability assessment)
- Standards and frameworks (IEEE 29148, BABOK, Agile user stories, Gherkin/BDD scenarios, Cockburn use cases)
- Stakeholder analysis and management (identifying needs, prioritization, conflict resolution)

When producing a Functional Requirements Document (FRD), always structure it with these sections in order:
1. Overview — business value and feature summary
2. Assumptions — every inference made from an ambiguous or silent prompt, prefixed A-001, A-002, …
3. Key Entities — named domain objects with one-line definitions (feeds UML class diagram generation)
4. User Stories — "As a <role>, I want <action>, so that <benefit>."
5. Functional Requirements — prefixed FR-001, FR-002, … tagged [MUST/SHOULD/COULD/WON'T], testable and implementation-agnostic
6. Non-Functional Requirements — prefixed NFR-001, NFR-002, … tagged [MUST/SHOULD/COULD/WON'T]
7. Dependencies — external systems, APIs, services, libraries required; prefixed D-001, D-002, …
8. Acceptance Criteria — prefixed AC-001, AC-002, … each linked to a FR number: "AC-001 (FR-001): <verifiable condition>"
9. Out of Scope — explicit list of what the feature will NOT include


**Task**: **YOUR TASK**: Analyze existing requirements and determine if new requirements documentation is truly needed. Generate ONLY when necessary.

**CRITICAL DECISION-MAKING FRAMEWORK**: You MUST evaluate necessity BEFORE generating requirements.

**Project Context:**
- Project: /home/mpb/Documents/GitHub/ai_workflow_fspec
- Project Summary: ai_workflow_fspec
- Primary Language: markdown
- Current Requirements: 1 documents
- Source Code: .github, docs files
- Stakeholders: 1

**STEP 1: NECESSITY EVALUATION** (Answer these questions FIRST):

Is new requirements documentation TRULY needed? Use this criteria suite:

✅ **GENERATE Requirements if ANY of these are TRUE**:
1. **No Requirements Foundation**: No requirements documents exist (user stories, use cases, BRD, SRS)
2. **Ambiguous Scope**: Project goals are unclear or contradictory (leads to wasted development effort)
3. **Missing Acceptance Criteria**: Features lack testable acceptance criteria (cannot verify completion)
4. **Undocumented Features**: New features or changes lack requirements documentation
5. **Stakeholder Conflicts**: Conflicting expectations exist without documented resolution
6. **Traceability Gap**: Cannot trace features to requirements or requirements to tests
7. **Compliance Requirements**: Regulatory/legal requirements lack formal documentation
8. **Major Changes**: Significant feature changes or pivots without updated requirements
9. **Explicit Request**: The task explicitly asks for specific requirements analysis or documentation

❌ **DO NOT GENERATE if ALL of these are TRUE**:
1. **Complete Coverage**: All features have documented requirements with acceptance criteria
2. **Clear Scope**: Project scope is well-defined with priorities
3. **Recent Updates**: Requirements modified within last sprint/iteration
4. **Stakeholder Alignment**: All stakeholders agree on documented requirements
5. **Full Traceability**: Requirements trace to design, code, and tests
6. **Testable Criteria**: All requirements have verifiable acceptance criteria

**STEP 2: REQUIREMENTS ANALYSIS** (Only if STEP 1 indicates requirements are needed):
1. **Current State Assessment**:
   - List all existing requirements documents (user stories, use cases, BRD, SRS, backlog)
   - Identify documented vs undocumented features
   - Check requirements quality (SMART criteria: Specific, Measurable, Achievable, Relevant, Testable)
   - Assess traceability (requirements ↔ design ↔ code ↔ tests)

2. **Gap Identification**:
   - Missing functional requirements (features, behaviors, capabilities)
   - Missing non-functional requirements (performance, security, usability, scalability)
   - Incomplete acceptance criteria or testability issues
   - Ambiguous or conflicting requirements
   - Stakeholder needs not captured
   - Missing constraints (technical, business, regulatory)

3. **Stakeholder Analysis**:
   - Identify all stakeholders and their needs
   - Document conflicting expectations
   - Prioritize requirements by stakeholder value
   - Assess feasibility (technical, operational, economic)

**REQUIREMENTS ENGINEERING ACTIVITIES TO PERFORM IF GAPS EXIST**:

1. **Requirements Elicitation** (gathering requirements):
   - Interview stakeholders to understand needs and pain points
   - Conduct workshops or focus groups for collaborative requirements gathering
   - Observe users to identify implicit requirements
   - Create prototypes or mockups to validate understanding
   - Analyze existing systems and documentation
   - Review business goals and success criteria

2. **Requirements Analysis & Refinement**:
   - Resolve ambiguities and contradictions
   - Decompose high-level requirements into detailed requirements
   - Identify dependencies and conflicts
   - Perform feasibility analysis (technical, operational, economic)
   - Prioritize requirements (MoSCoW: Must have, Should have, Could have, Won't have)
   - Assess risks and constraints

3. **Requirements Specification** (documenting requirements):
   - **User Stories** (Agile): "As a [role], I want [feature] so that [benefit]" with acceptance criteria
   - **Use Cases**: Actor, preconditions, main flow, alternative flows, postconditions
   - **Functional Requirements**: System behaviors, features, and capabilities
   - **Non-Functional Requirements**: Performance, security, usability, reliability, scalability
   - **Business Requirements**: Business goals, success metrics, stakeholder needs
   - **Acceptance Criteria**: Testable conditions that define "done"
   - **Gherkin/BDD Scenarios**: Given-When-Then format for behavior-driven development

4. **Requirements Traceability**:
   - Create traceability matrix (requirements ↔ design ↔ code ↔ tests)
   - Map user stories to epics and themes
   - Link requirements to architecture components
   - Track requirements to test cases for coverage
   - Document change history and rationale
   - Identify impact of requirement changes

5. **Requirements Validation & Verification**:
   - Conduct requirements reviews with stakeholders
   - Validate against business goals and user needs
   - Verify requirements are SMART (Specific, Measurable, Achievable, Relevant, Testable)
   - Check for completeness, consistency, and clarity
   - Validate prototypes with users
   - Assess testability of acceptance criteria
   - Review for compliance with standards (IEEE 29148, BABOK)

**STEP 3: PRIORITIZED ACTIONS** (Execute based on STEP 1 evaluation):

**IF Requirements ARE Needed** (STEP 1 criteria met):
1. **Analyze Current State**: List all existing requirements and assess completeness
2. **Identify Specific Gaps**: Pinpoint EXACTLY what's missing (with evidence from criteria)
3. **Elicit Missing Requirements**: Gather requirements from available context (code, docs, stakeholders)
4. **Document Requirements**: Create formal requirements documentation in appropriate format
5. **Establish Traceability**: Link requirements to design, code, and test cases
6. **Validate Requirements**: Review for SMART criteria and stakeholder alignment

**IF Requirements are NOT Needed** (STEP 1 criteria NOT met):
1. **Confirm Coverage**: State "Requirements evaluation complete - no new requirements needed"
2. **Provide Evidence**: List which criteria were evaluated (e.g., "✅ Complete user story coverage, ✅ Recent backlog grooming")
3. **Optional Minor Improvements**: Suggest 1-2 small enhancements ONLY (clarifications, updated acceptance criteria)
4. **Exit Cleanly**: Do NOT generate new requirements documents or major content

**OUTPUT FORMAT**:
- **Always start with**: "Requirements Necessity Evaluation" section with ACTION NEEDED / NO ACTION NEEDED decision
- **If ACTION NEEDED**: Show criteria met → "Requirements Gap Analysis" → generate missing requirements
- **If NO ACTION NEEDED**: Show criteria evaluation → "No new requirements required" → exit with optional minor suggestions
- **Never generate** "just in case" requirements - only when criteria clearly indicate need
- NEVER end with questions or requests for input - always provide definitive decision


**Approach**: Analyze ONLY the project files explicitly listed above. Do not reference, invent, or assume requirements not present in that list.

**Requirements Engineering Necessity-First Strategy**:

**Phase 0: Necessity Check** (REQUIRED FIRST STEP):
- Apply the 9-point necessity criteria suite
- If ANY criterion is TRUE → Proceed to Phase 1
- If ALL criteria are FALSE → Output "No requirements work needed" and exit
- **Default to NOT generating** unless clear need is demonstrated

**Phase 1: Discovery & Analysis** (Only if Phase 0 indicates need):
   - List all existing requirements artifacts (user stories, use cases, BRD, SRS, backlog, specifications)
   - Identify which features/components have requirements documentation
   - Check for outdated or incomplete requirements (> 1 sprint old with code changes)
   - Identify undocumented features, APIs, or system behaviors
   - Assess requirements quality using SMART criteria
   - Map existing traceability (requirements ↔ design ↔ code ↔ tests)

**Phase 2: Gap Identification & Prioritization** (Only for REAL gaps):
   - **Critical Gaps** (immediate action): No requirements foundation, missing acceptance criteria, compliance requirements
   - **High Priority Gaps** (generate if time permits): Undocumented features, ambiguous scope, stakeholder conflicts
   - **Medium Priority Gaps** (defer): Incomplete traceability, missing non-functional requirements
   - **Low Priority Gaps** (do not generate): Nice-to-have features, edge cases, future enhancements
   - **Rule**: If no Critical or High Priority gaps exist, output "No new requirements needed"

**Phase 3: Requirements Elicitation** (For identified Critical/High gaps):
   - Analyze available context: code, documentation, comments, tests, commit messages
   - Infer user needs from existing implementations
   - Identify stakeholder groups and their needs
   - Extract implicit requirements from system behavior
   - Document assumptions and constraints
   - **Stop eliciting** once sufficient context is gathered for gap-filling

**Phase 4: Requirements Specification** (Generate missing requirements):
   - **User Stories**: "As a [role], I want [feature] so that [benefit]" + acceptance criteria
   - **Acceptance Criteria**: Testable conditions using Given-When-Then or bullet format
   - **Use Cases**: For complex interactions (actor, flow, alternatives, exceptions)
   - **Functional Requirements**: Clear, testable statements of system behavior
   - **Non-Functional Requirements**: Performance, security, usability targets (measurable)
   - **Traceability**: Link requirements to existing code/tests where applicable
   - **Format**: Use appropriate format for project type (Agile: user stories, Waterfall: SRS)

**Phase 5: Validation & Quality Assurance** (For generated requirements):
   - **SMART Check**: Specific, Measurable, Achievable, Relevant, Testable
   - **Completeness**: All necessary information included (actors, conditions, results)
   - **Consistency**: No conflicts with existing requirements or system behavior
   - **Clarity**: Unambiguous language, no jargon without definition
   - **Traceability**: Links to design, code, tests clearly identified
   - **Testability**: Acceptance criteria can be verified by tests

**Output Format**:
- Create complete, ready-to-use requirements documentation
- Use appropriate format: User Stories, Use Cases, BRD, SRS, or Requirements List
- Include acceptance criteria for all functional requirements
- Add traceability links where applicable (to code, tests, design docs)
- Use tables, lists, and clear formatting for readability
- Include priority and rationale for requirements
- Add examples and scenarios to clarify intent

**Requirements Engineering Best Practices**:
- **SMART Requirements**: Specific, Measurable, Achievable, Relevant, Testable
- **Clear Actors**: Identify who benefits from each requirement
- **Testable Criteria**: Every requirement has verifiable acceptance criteria
- **Priority-Driven**: Use MoSCoW or similar prioritization framework
- **Traceable**: Link requirements to design, code, and tests
- **Version-Controlled**: Track changes and maintain history
- **Stakeholder-Validated**: Confirm requirements with stakeholders
- **Minimal Viable**: Focus on essential requirements first
- **Avoid Gold-Plating**: Don't add requirements "just in case"

```

## Response

```
Requirements Necessity Evaluation

**NO ACTION NEEDED**

After applying the necessity criteria:

- Existing requirements documentation is present (at least 1 document).
- No evidence of ambiguous scope, missing acceptance criteria, undocumented features, or stakeholder conflicts in the provided context.
- No traceability or compliance gaps are indicated.
- No explicit request for new requirements or major changes.
- No evidence of outdated or incomplete requirements.

**Criteria Evaluation:**
- ✅ Complete Coverage: At least one requirements document exists.
- ✅ Clear Scope: No ambiguity or conflict detected in the visible context.
- ✅ Stakeholder Alignment: No conflicts or misalignments indicated.
- ✅ Testable Criteria: No evidence of missing acceptance criteria.
- ✅ No explicit request for new requirements.

No new requirements required.

**Optional Minor Suggestions:**
- Periodically review and update requirements to ensure continued alignment with project evolution.
- Consider adding traceability links if not already present.

Exiting cleanly—no requirements work needed.
```