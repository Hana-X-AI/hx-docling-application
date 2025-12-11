# Project Charter Template

**Document Type:** Project/Service Charter  
**Template Version:** 1.0  
**Last Updated:** 2025-11-16

---

## Purpose

The project charter is the **foundational document** that defines the vision, justification, and scope of any project or service deployment. It answers the fundamental questions:

- **WHY** does this project exist? (Vision & Justification)
- **WHAT** will it deliver? (Scope & Deliverables)
- **WHO** is involved? (Stakeholders & Ownership)
- **HOW** will success be measured? (Success Criteria)
- **WHEN** will it be delivered? (Timeline & Milestones)

**The charter is the FIRST document created** in the HX-Infrastructure workflow. Nothing proceeds without an approved charter. All subsequent documents (specification, plan, tasks) derive from and must align with the charter.

**Charter vs. Other Documents:**
- **Charter:** Defines the "WHY" and "WHAT" - vision, purpose, scope (this document)
- **Specification:** Defines the "HOW" in detail - requirements, implementation details
- **Plan:** Defines the "HOW" for deployment - architecture, strategy, steps

Think of the charter as the **executive summary and business case** - concise, strategic, and decision-focused. Detailed technical information belongs in the specification and plan documents.

---

## Embedded Workflow

**This charter template includes an embedded workflow for project initiation:**

### Charter Creation Workflow

```
Step 1: Vision & Justification
├─ Define project purpose and vision
├─ Document business/technical justification
├─ Identify key stakeholders
└─ OUTPUT: Vision statement, justification, stakeholder list

Step 2: Scope Definition
├─ Define what IS in scope
├─ Define what IS NOT in scope
├─ Identify constraints and assumptions
└─ OUTPUT: Clear scope boundaries

Step 3: Success Criteria
├─ Define measurable success criteria
├─ Establish acceptance criteria
├─ Define quality metrics
└─ OUTPUT: Success metrics and acceptance criteria

Step 4: Resource Planning
├─ Identify required agents/teams
├─ Estimate timeline and milestones
├─ Identify dependencies
└─ OUTPUT: Resource allocation, timeline, dependencies

Step 5: Risk Assessment
├─ Identify top 3-5 critical risks
├─ Document top 3-5 key assumptions
├─ Note: Detailed tracking happens in existing RAIDD log
└─ OUTPUT: Critical risks/assumptions for stakeholder awareness

Step 6: Approval & Kickoff
├─ Charter review by stakeholders
├─ Constitution alignment validation
├─ Approval signatures
└─ OUTPUT: Approved charter, ready to proceed

Step 7: Post-Approval Actions (After charter approved)
├─ Review and update existing RAIDD log with charter items
├─ Review and update existing Backlog with deferred/out-of-scope items
├─ Review and update defect log (if applicable)
├─ Kick-off meeting with team
├─ Knowledge assignment review (team reviews relevant repos)
└─ Begin specification work
```

**Workflow Position:** FIRST document in any project/service lifecycle  
**Next Step After Charter:** Review existing logs, kick-off meeting with knowledge review, then specification

**Important Notes:**
- RAIDD log, Backlog, and Defect log ALREADY EXIST at project level
- Charter process REVIEWS and UPDATES these existing logs, does NOT create new ones
- Knowledge review is CRITICAL before specification work begins

---

## Project/Service Identification

### Basic Information
**Project/Service Name:** [descriptive-name]  
**Project/Service ID:** [unique-identifier]  
**Project Type:** [Infrastructure | Service | Tool | Integration | Platform]  
**Charter Version:** 1.0  
**Charter Date:** [YYYY-MM-DD]  
**Charter Status:** [Draft | Approved | Active | Completed]

### Ownership
**Project Owner:** [Name/Role]  
**Technical Lead:** [Name/Role - Agent/Human]  
**Stakeholders:** [List key stakeholders]

---

## Vision and Purpose

### Vision Statement
[2-3 sentences describing the aspirational goal of this project/service]

### Business/Technical Justification
**Why does this project exist?**

[Detailed explanation of the business or technical need this addresses]

**What problem does it solve?**

[Specific problem statement]

**What value does it deliver?**

[Expected value, benefits, improvements]

### Strategic Alignment
**Constitution Alignment:**
- [Principle 1]: [How this aligns]
- [Principle 2]: [How this aligns]

**Infrastructure Strategy Alignment:**
[How this fits into broader infrastructure goals]

---

## Scope

### In Scope
**This project INCLUDES:**
1. [Specific deliverable 1]
2. [Specific deliverable 2]
3. [Specific deliverable 3]
4. [Additional items]

**Capabilities Delivered:**
- [Capability 1]
- [Capability 2]
- [Capability 3]

### Out of Scope
**This project EXPLICITLY EXCLUDES:**
1. [Item 1 - with rationale why excluded]
2. [Item 2 - with rationale why excluded]
3. [Item 3 - with rationale why excluded]

**Future Considerations:**
[Items that may be considered for future phases but are out of current scope]

### Boundaries and Constraints
**Technical Constraints:**
- [Constraint 1 - e.g., Must use existing infrastructure]
- [Constraint 2 - e.g., Limited to specific technology stack]

**Resource Constraints:**
- [Constraint 1 - e.g., Available agents/personnel]
- [Constraint 2 - e.g., Budget/time limitations]

**Operational Constraints:**
- [Constraint 1 - e.g., Must maintain 99.9% uptime of existing services]
- [Constraint 2 - e.g., Zero-downtime deployment required]

---

## Success Criteria

### Measurable Success Criteria
**The project is successful when:**

1. **[Success Criterion 1]**
   - Metric: [How measured]
   - Target: [Specific target value]
   - Validation: [How validated]

2. **[Success Criterion 2]**
   - Metric: [How measured]
   - Target: [Specific target value]
   - Validation: [How validated]

3. **[Success Criterion 3]**
   - Metric: [How measured]
   - Target: [Specific target value]
   - Validation: [How validated]

### Acceptance Criteria
**Project can be accepted when:**
- [ ] All success criteria met
- [ ] All in-scope deliverables completed
- [ ] 100% test coverage achieved
- [ ] Documentation complete
- [ ] Operational readiness validated
- [ ] Stakeholder sign-off obtained

### Quality Metrics
**Quality will be measured by:**
- [Quality metric 1]: Target [value]
- [Quality metric 2]: Target [value]
- [Quality metric 3]: Target [value]

---

## Stakeholders and Roles

### Primary Stakeholders
| Stakeholder | Role | Responsibility | Decision Authority |
|-------------|------|----------------|-------------------|
| [Name] | Project Owner | Overall accountability | Final approval |
| [Name] | Technical Lead | Technical decisions | Architecture decisions |
| [Name] | Agent/Team | Implementation | Development decisions |
| [Name] | Operations | Service operation | Operational decisions |

### Communication Plan
**Status Updates:** [Frequency - e.g., Weekly]  
**Status Report Format:** [Reference to status-report-template.md]  
**Escalation Path:** [Who to contact for issues]  
**Decision-Making Process:** [How decisions are made and documented]

---

## High-Level Approach

### Architecture Overview
[High-level architectural approach - details in architecture documents]

**Key Components:**
1. [Component 1]
2. [Component 2]
3. [Component 3]

**Integration Points:**
- [Integration 1]
- [Integration 2]

### Technology Stack
**Primary Technologies:**
- [Technology 1]: [Purpose]
- [Technology 2]: [Purpose]
- [Technology 3]: [Purpose]

### Deployment Strategy
[High-level deployment approach]

---

## Timeline and Milestones

### Key Milestones
| Milestone | Target Date | Deliverables | Dependencies |
|-----------|-------------|--------------|--------------|
| Charter Approval | [DATE] | Approved charter | Stakeholder review |
| Specification Complete | [DATE] | spec.md | Charter approval |
| Design Complete | [DATE] | plan.md, architecture.md | Specification |
| Testing Complete | [DATE] | All tests passing | Implementation |
| Deployment Complete | [DATE] | Service operational | Testing |
| Project Closure | [DATE] | Lessons learned | Deployment |

### Estimated Duration
**Total Project Duration:** [X weeks/months]  
**Phases:**
- Specification & Design: [Duration]
- Implementation: [Duration]
- Testing: [Duration]
- Deployment & Stabilization: [Duration]

---

## Resources

### Agent/Team Allocation
**Assigned Agents:**
- [Agent 1]: [Role] - [Time allocation]
- [Agent 2]: [Role] - [Time allocation]
- [Agent 3]: [Role] - [Time allocation]

**Human Resources:**
- [Human 1]: [Role] - [Time allocation]

### Infrastructure Resources
**Required Infrastructure:**
- [Resource 1]: [Specification]
- [Resource 2]: [Specification]

**Node Allocation:**
- [Node name]: [Purpose]

### Knowledge Resources
**Required Knowledge Repositories:**
- [Repo 1]: [Purpose]
- [Repo 2]: [Purpose]

---

## Dependencies and Prerequisites

### Internal Dependencies
**Depends On:**
1. [Dependency 1] - Status: [Status] - Impact if delayed: [Impact]
2. [Dependency 2] - Status: [Status] - Impact if delayed: [Impact]

### External Dependencies
**External Factors:**
1. [External dependency 1] - Provider: [Who] - Risk: [Level]
2. [External dependency 2] - Provider: [Who] - Risk: [Level]

### Prerequisites
**Must be complete before project starts:**
- [ ] [Prerequisite 1]
- [ ] [Prerequisite 2]
- [ ] [Prerequisite 3]

---

## Risks and Assumptions

**⚠️ CRITICAL: LIMIT TO TOP 3-5 ITEMS ONLY**  
*The charter is NOT the place for comprehensive risk/assumption tracking. That's what the RAIDD log is for. If you find yourself listing more than 5 risks or 5 assumptions, you're doing it wrong. Stop and move the detailed tracking to the RAIDD log.*

### Initial Risk Assessment
**LIMIT: Top 3-5 Most Critical Risks Only**  
*Include only the highest-impact, most-likely risks that stakeholders MUST be aware of before approving the project. All other risks go in the RAIDD log.*

**Selection Criteria:** Include a risk in the charter ONLY if:
- It has HIGH likelihood AND HIGH impact, OR
- It could fundamentally change or cancel the project, OR
- Stakeholders need to know it for approval decision

| Risk ID | Risk Description | Likelihood | Impact | Mitigation Strategy |
|---------|-----------------|------------|--------|---------------------|
| R-001 | [Critical Risk 1] | [H/M/L] | [H/M/L] | [High-level strategy] |
| R-002 | [Critical Risk 2] | [H/M/L] | [H/M/L] | [High-level strategy] |
| R-003 | [Critical Risk 3] | [H/M/L] | [H/M/L] | [High-level strategy] |

### Key Assumptions
**LIMIT: Top 3-5 Most Critical Assumptions Only**  
*Include only assumptions that, if proven wrong, would fundamentally change the project approach. All other assumptions go in the RAIDD log.*

**Selection Criteria:** Include an assumption in the charter ONLY if:
- The entire project approach depends on it being true, OR
- It needs stakeholder validation/approval, OR
- If wrong, would require major project replanning

1. [Critical Assumption 1] - Validation method: [How to validate]
2. [Critical Assumption 2] - Validation method: [How to validate]
3. [Critical Assumption 3] - Validation method: [How to validate]

**⚠️ For Claude Code (CC):** If you're tempted to add more than 5 risks or 5 assumptions to this charter, that's a signal that you need to:
1. Create the RAIDD log NOW (don't wait)
2. Put the detailed items in the RAIDD log where they belong
3. Keep ONLY the top 3-5 most critical items in the charter
4. Remember: Charter = executive summary, RAIDD log = comprehensive tracking

**RAIDD Log Reference:** `/home/agent0/HX-Infrastructure/docs/raidd-log.md` (single centralized file)
**All risks and assumptions MUST be documented in the centralized RAIDD log.**

---

## Governance and Approval

### Charter Review
**Reviewed By:**
- [ ] Project Owner - [Name] - Date: [DATE]
- [ ] Technical Lead - [Name] - Date: [DATE]
- [ ] Infrastructure Lead - [Name] - Date: [DATE]

### Charter Approval
**Approved By:**

**Project Owner:** [Name]  
**Signature:** ________________  
**Date:** [YYYY-MM-DD]

**Technical Authority:** [Name]  
**Signature:** ________________  
**Date:** [YYYY-MM-DD]

**Infrastructure Authority:** [Name]  
**Signature:** ________________  
**Date:** [YYYY-MM-DD]

### Constitution Validation
**Constitution Alignment Verified:** [YES | NO]  
**Verified By:** [Name]  
**Date:** [YYYY-MM-DD]  
**Notes:** [Any alignment notes or required adjustments]

---

## Next Steps After Charter Approval

### Immediate Next Actions

1. **Review and Update RAIDD Log**
   - **Action:** REVIEW existing `raidd-log.md` (centralized project RAIDD log)
   - **Update:** Add any new risks, assumptions, dependencies from this charter
   - **Note:** RAIDD log already exists - do NOT create a new one
   - Transfer top 3-5 charter items to comprehensive RAIDD log entries
   - Add project-specific context and details

2. **Review and Update Backlog**
   - **Action:** REVIEW existing `backlog.md` (centralized project backlog)
   - **Update:** Add items from this charter that are:
     - Out-of-scope items agreed to delay to future phases
     - Known tasks/features/capabilities for this phase
     - Any scope items deferred from charter discussions
   - **Note:** Backlog already exists - do NOT create a new one
   - **Backlog Purpose:** Track delayed/deferred work, NOT all work (detailed tasks come later from spec/plan)

3. **Review and Update Defect Log** (if applicable)
   - **Action:** REVIEW existing centralized defect log
   - **Update:** Note any known issues or technical debt related to this project
   - **Note:** Defect log already exists - do NOT create a new one

4. **Setup Project Structure**
   - Create project/service directory structure
   - Initialize project-specific documentation
   - Setup version control branch (if applicable)

5. **Kick-off Meeting with Team**
   - Review charter with team
   - **Review Knowledge Assignments** (BEFORE specification work):
     - Each team member reviews relevant knowledge vault repositories
     - Example: For docling-mcp project:
       - ALL team members: Deep analysis of `docling-mcp` core repository
       - Each member: Review their expertise-specific repos from knowledge vault
         - Agent responsible for API: Review API integration patterns
         - Agent responsible for deployment: Review deployment repos
         - Agent responsible for testing: Review testing frameworks
     - Ensure team has comprehensive understanding before beginning specification
   - Assign initial responsibilities
   - Begin specification work

### Documents to Create Next
- [ ] `services/<service>/charter.md` - THIS document (if service-specific)
- [ ] `services/<service>/spec.md` - Detailed specification
- [ ] `services/<service>/status-report.md` - Initial status report

### Documents to Review and Update
- [ ] `raidd-log.md` - Review and add charter items
- [ ] `backlog.md` - Review and add deferred/out-of-scope items
- [ ] `defect-log.md` - Review for known issues (if applicable)

---

## Document References

**Related Templates:**
- `service-spec-template.md` - Specification
- `status-report-template.md` - Status reporting

**Existing Project Documents to Review/Update:**
- `raidd-log.md` - RAIDD tracking (already exists - review and update)
- `backlog.md` - Backlog management (already exists - review and update)
- `defect-log.md` - Defect tracking (already exists - review if applicable)

**Governance Documents:**
- `constitution.md` - Project principles
- `architecture-standards.md` - Architecture governance
- `deployment-requirements.md` - Deployment standards
- `testing-requirements.md` - Testing standards

**Knowledge Resources:**
- `hx-knowledge-vault-catalog.md` - Knowledge vault repository catalog
- Relevant repositories based on project scope (review during kick-off)

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | [DATE] | [Name] | Initial charter creation |

---

## Charter Checklist

**Use this checklist to ensure charter completeness:**

### Content Completeness
- [ ] Vision and purpose clearly articulated
- [ ] Scope (in/out) explicitly defined
- [ ] Success criteria measurable and specific
- [ ] Stakeholders identified with roles
- [ ] Timeline with milestones
- [ ] Resources identified and allocated
- [ ] Dependencies documented
- [ ] Top 3-5 critical risks listed (detailed tracking in RAIDD log)
- [ ] Top 3-5 key assumptions listed (detailed tracking in RAIDD log)
- [ ] Approval signatures obtained

### Quality Checks
- [ ] Aligns with constitution principles
- [ ] Success criteria are SMART (Specific, Measurable, Achievable, Relevant, Time-bound)
- [ ] Scope is clear and realistic
- [ ] Dependencies are actionable
- [ ] Risks have high-level mitigation strategies
- [ ] Timeline is achievable
- [ ] Charter is concise (avoid bloat - details go in spec/plan)

### Process Checks
- [ ] RAIDD log will be created immediately after approval
- [ ] Backlog will be initialized
- [ ] Specification phase planned
- [ ] Review process completed
- [ ] All stakeholders engaged

### Anti-Bloat Checks
- [ ] Charter is 3-5 pages maximum (not a detailed specification)
- [ ] Only top 3-5 risks included (rest in RAIDD log)
- [ ] Only top 3-5 assumptions included (rest in RAIDD log)
- [ ] No detailed technical design (belongs in plan.md)
- [ ] No detailed budget breakdown (if needed, create separate doc)

---

**Template Version:** 1.0  
**Last Updated:** 2025-11-16  
**Repository:** https://github.com/Hana-X-AI/HX-Infrastructure.git

---

## Template Usage Notes

**When to Create a Charter:**
- At the start of any new project
- At the start of any major service deployment
- When initiating significant infrastructure changes

**Charter Length & Scope:**
- **Target Length:** 3-5 pages maximum
- **Purpose:** Executive summary and business case, NOT detailed specification
- **Anti-Bloat Principle:** If it's getting long, you're including too much detail
  - Detailed requirements → spec.md
  - Detailed architecture → plan.md & architecture.md
  - Detailed risks/assumptions → raidd-log.md
  - Detailed tasks → task files
  - Budget details → separate financial document (if needed)

**Charter as Foundation:**
- Charter is the FIRST document in the workflow
- All subsequent documents (spec, plan, tasks) derive from charter
- Charter remains the authoritative vision throughout project
- Charter should be stable once approved (major changes require re-approval)

**Charter vs. Specification:**
- **Charter** = WHY and WHAT (vision, purpose, scope, business case)
- **Specification** = HOW in detail (requirements, implementation details)
- **Plan** = HOW for deployment (architecture, strategy, detailed steps)

**Keeping Charters Focused:**
- Focus on strategic decisions, not tactical details
- Include only information stakeholders need for approval
- Remember: Readers should understand the project in 5 minutes
- Detailed tracking happens in RAIDD log and backlog, not charter

---

## 🔗 Related Documents

**Workflows:**
- [Charter Workflow](/home/agent0/HX-Infrastructure/procedures/charter-workflow.md)
- [Specification Workflow](/home/agent0/HX-Infrastructure/procedures/spec-workflow.md) (Next Phase)

**Templates:**
- [Charter Questions Template](/home/agent0/HX-Infrastructure/templates/charter-questions-template.md)
- [Knowledge Vault Research Template](/home/agent0/HX-Infrastructure/templates/knowledge-vault-research-template.md)
- [RAIDD Log Template](/home/agent0/HX-Infrastructure/templates/raidd-log-template.md)

**Reference:**
- [Constitution](/home/agent0/HX-Infrastructure/constitution.md)
- [Architecture Standards](/home/agent0/HX-Infrastructure/standards/architecture-standards.md)

---

**Template Version:** 1.0
**Last Updated:** 2025-11-18
**Used In:** Phase 1 of Project Lifecycle (Charter Creation)
**Maintained By:** Agent Zero (CC)
