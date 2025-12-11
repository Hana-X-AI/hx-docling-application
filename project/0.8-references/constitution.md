# HX Infrastructure Base Constitution

## Core Principles

### I. Documentation-First (NON-NEGOTIABLE)
**Every infrastructure change starts with documentation before execution.**

- No service deployments without complete spec.md and plan.md
- No configuration changes without documented procedures
- No node modifications without updated node specifications
- All changes must be version controlled in hx-infra-base repository
- Documentation must be agent-readable and human-readable

**Rationale:** Infrastructure changes without documentation lead to knowledge loss, deployment drift, and operational chaos. Agents and humans must understand the current state before making changes.

---

### II. Test-Driven Deployment (NON-NEGOTIABLE)
**All services must have passing test suites before operational status.**

- Test suites are mandatory, not optional
- Test cases must validate spec.md requirements
- Test cases must verify plan.md implementation
- All tests must pass before promotion from non-operational to operational
- Test results must be documented with timestamps

**Test Suite Requirements:**
- Deployment validation tests
- Service functionality tests  
- Integration tests (if service integrates with others)
- Health check tests

**Workflow:**
1. Write spec.md and plan.md
2. Write test suite based on spec/plan
3. Tests FAIL (service not deployed yet)
4. Deploy service following plan.md tasks
5. Run test suite
6. Tests PASS → Service moves to operational/
7. Tests FAIL → Service stays in non-operational/, defects logged

**Rationale:** Untested deployments create technical debt and operational risk. Tests ensure deployments match specifications and function as intended.

---

### III. Spec-Driven Process
**All infrastructure work follows the spec → plan → tasks → test workflow.**

- **Spec:** What we're deploying and why (requirements, purpose, success criteria)
- **Plan:** How we'll deploy it (architecture, dependencies, configuration)
- **Tasks:** Step-by-step deployment actions (ordered, traceable)
- **Tests:** Validation that spec and plan are correctly implemented

**No shortcuts:** Skipping spec or plan leads to undocumented deployments and knowledge gaps.

**Rationale:** Structured process ensures consistency, traceability, and knowledge transfer. Agents can understand past decisions and execute future work correctly.

---

### IV. Single Responsibility Principle
**One service, one purpose. One node, one clear role.**

- Services must have clear, focused purposes defined in spec.md
- No monolithic "do-everything" services
- Nodes should have defined roles (database node, API node, worker node, etc.)
- Dependencies must be explicitly documented
- Avoid tight coupling between services

**Rationale:** Single responsibility enables independent deployment, testing, and troubleshooting. Simplifies understanding for both agents and humans.

---

### V. Operational Status Clarity
**Clear separation between operational and non-operational services.**

**Operational Services:**
- All tests passing
- Documented and verified
- Production-ready
- Actively maintained

**Non-Operational Services:**
- In planning or deployment
- Tests incomplete or failing
- Not ready for production use
- May have known defects

**No gray areas:** Services are either operational or non-operational. Partial deployments stay non-operational.

**Rationale:** Clear status prevents confusion about what's ready for use and what's still in progress.

---

### VI. Quality Over Speed
**Accuracy and completeness matter more than deployment velocity.**

- Take time to write complete specifications
- Don't rush testing
- Fix defects before promotion to operational
- Prefer thorough documentation over quick deployments
- Approach all work assuming lowest common denominator consumers

**Rationale:** Infrastructure mistakes compound over time. Rushing creates technical debt, outages, and knowledge gaps that cost more to fix later.

---

### VII. Defect Transparency
**All defects must be documented centrally and tracked to resolution.**

**Required for all defects:**
- Clear description of the issue
- Severity classification (critical, high, medium, low)
- Affected service(s)
- Steps to reproduce (if applicable)
- Resolution status
- Date opened and date resolved

**Critical and High severity defects:**
- Block promotion to operational status
- Require immediate attention
- Must be resolved before service can be considered operational

**Medium and Low severity defects:**
- Should be tracked but may not block operational promotion
- Should be prioritized in backlog

**Rationale:** Transparent defect tracking prevents issues from being forgotten and ensures quality standards are maintained.

---

### VIII. Agent-Optimized Documentation
**All documentation must be consumable by AI agents (Claude Code, GitHub Copilot).**

- Use clear, structured markdown
- Follow naming conventions religiously  
- Provide context and rationale, not just instructions
- Use templates consistently
- Avoid ambiguity - be explicit
- Reference dependencies and relationships clearly

**Agents must be able to:**
- Understand the current state of infrastructure
- Execute deployments following documented procedures
- Run test suites and interpret results
- Identify and log defects
- Make informed decisions based on documentation

**Rationale:** AI agents are key collaborators in infrastructure management. Documentation that agents can't parse or understand is incomplete.

---

## Infrastructure Standards

### Naming Conventions
**All files, directories, services, and nodes must follow naming standards defined in `templates/naming-conventions.md`.**

- Consistent naming enables automation
- Non-compliant names will be rejected
- No exceptions without documented justification

### Version Control
**All infrastructure documentation and configuration must be version controlled.**

- Use git for all changes
- Meaningful commit messages
- Branch naming follows conventions
- Pull requests required for changes
- No direct commits to main branch

### Knowledge Base Maintenance
**hx-knowledge/ directory must be kept current with relevant reference materials.**

- Repository documentation in `hx-knowledge/repos/`
- Supporting documentation in `hx-knowledge/docs/`
- Agents must reference knowledge base before deployments
- Outdated materials must be removed or archived

---

## Deployment Workflow

### Standard Deployment Process
1. **Document:** Create spec.md and plan.md in `services/non-operational/<service>/`
2. **Task Breakdown:** Create tasks following `<service>-task-<seq>-<description>.md` format
3. **Test Definition:** Write test suite in `tests/test-suite/` organized by test area
4. **Review:** Documentation reviewed and approved
5. **Execute:** Follow tasks in sequence
6. **Validate:** Run complete test suite
7. **Verify:** All tests must pass
8. **Promote:** Move service from non-operational/ to operational/
9. **Update Inventory:** Update `inventory/services.md` and `inventory/nodes.md`

### Service Promotion Criteria
A service can only move from non-operational to operational when:
- [ ] spec.md is complete and reviewed
- [ ] plan.md is complete and reviewed
- [ ] All tasks are completed
- [ ] All deployment tests pass
- [ ] All functionality tests pass
- [ ] All integration tests pass (if applicable)
- [ ] All health check tests pass
- [ ] No critical or high severity defects
- [ ] Inventory documents updated

---

## Review and Approval

### Documentation Review
All new services and significant changes require review:
- Specification completeness
- Plan feasibility and clarity
- Task breakdown logical sequencing
- Test coverage adequacy
- Naming convention compliance
- Standards adherence

### Test Review
All test suites require review:
- Tests validate spec requirements
- Tests verify plan implementation
- Test coverage is adequate
- Test cases are independent
- Test results are reproducible

---

## Defect Management

### Severity Definitions
**Critical:** System down, complete service failure, data loss, security breach
**High:** Major functionality broken, significant impact to operations
**Medium:** Functionality impaired, workaround available
**Low:** Minor issue, cosmetic, enhancement request

### Defect Resolution Requirements
- Critical: Must be resolved before operational promotion
- High: Must be resolved before operational promotion
- Medium: Should be resolved, may be accepted with documented justification
- Low: Can be backlogged

### Defect Documentation
All defects must be logged in `defects/` directory following naming convention:
`defect-<service>-<severity>-<sequence>-<brief-description>.md`

---

## Exception Process

### When to Request Exception
Exceptions to this constitution should be rare and require strong justification:
- Emergency operational situations
- Documented technical limitations
- Temporary workarounds (with sunset date)

### Exception Requirements
1. Document the exception request
2. Explain why constitution cannot be followed
3. Detail the proposed alternative approach
4. Define acceptance criteria and rollback plan
5. Set expiration/review date
6. Obtain approval from infrastructure team
7. Document the exception in the service directory

**Exception template:** `templates/exception-request-template.md` (to be created)

---

## Governance

### Constitution Authority
This constitution supersedes all other infrastructure practices and procedures. In case of conflict:
1. Constitution requirements take precedence
2. Standards documents provide implementation details
3. Procedures provide step-by-step guidance
4. Templates provide starting points

### Amendment Process
**Amendments to this constitution require:**
1. Documented proposal with rationale
2. Impact analysis on existing infrastructure
3. Migration plan for non-compliant services
4. Review and approval by infrastructure team
5. Version increment
6. Communication to all team members and agents

### Compliance
**All infrastructure work must comply with this constitution.**
- Pull requests must verify compliance
- Non-compliant work will be rejected
- Regular audits will verify adherence
- Continuous improvement based on lessons learned

### Living Document
This constitution is a living document that evolves based on:
- Operational experience
- Technology changes
- Team feedback
- Agent collaboration insights
- Industry best practices

**Review Cadence:** Quarterly reviews to assess effectiveness and identify improvement opportunities.

---

## Related Documents
- `templates/naming-conventions.md` - File and directory naming standards
- `standards/documentation-requirements.md` - Documentation completeness standards
- `standards/testing-requirements.md` - Test suite requirements
- `standards/deployment-requirements.md` - Deployment process standards
- `procedures/service-deployment.md` - Step-by-step deployment guide
- `procedures/service-promotion.md` - Process for promoting to operational
- `procedures/defect-management.md` - Defect tracking and resolution

---

**Version**: 1.0  
**Ratified**: 2025-11-15  
**Last Amended**: 2025-11-15  
**Repository**: https://github.com/Hana-X-AI/HX-Infrastructure.git

**Next Review Date**: 2026-02-15
