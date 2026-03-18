---
name: create-technical-spec
description: Generates a technical specification document for a new feature or product.
---

# Create Technical Spec

This skill guides the creation of technical specification documents using a standardized template.

## Instructions

1.  **Analyze the Requirement**: Understand the feature or product being requested.
2.  **Determine File Path**: The technical spec must be saved in the `specs` folder at the root of the workspace.
    -   Target Directory: `specs/` (Create this directory if it doesn't exist)
    -   Filename Format: `<feature-name>.md` (e.g., `specs/user-authentication.md`)
3.  **Generate Content**: Use the template below to structure the document.
    -   **Flexible Application**: Not every use case requires every section of the template. Use your best judgment to determine which sections are relevant and necessary for the specific task. Omit sections that are not applicable.
    -   **Professional Tone**: Maintain a clear, concise, and professional technical writing style.
4.  **Save File**: Write the generated content to the target file.

## Technical Spec Template

```markdown
# Technical Spec

**Title**: [Feature/Product Name] - Technical Spec
**Author**: [Tech Lead] | **Status**: Draft/Approved/Implemented | **Date**: [YYYY-MM-DD]
**Product Spec**: [Link]

---

## Overview
**Product Context**: 1-2 sentences on WHAT we're building (link to product spec if any)
**Technical Approach**: 2-3 sentences on HOW we'll build it
**Key Technologies**: Languages, frameworks, databases, infrastructure

---

## Goals & Non-Goals
**Goals**: Technical objectives (scalability, performance targets, app builds fine with no introduced lint or type errors, etc.)
**Non-Goals**: What we're explicitly not doing in this iteration

---

## Current System
**Existing Architecture**: Brief description or diagram of relevant current state
**Limitations**: What doesn't work today
**What We'll Leverage**: Existing code, patterns, infrastructure we can reuse

---

## Proposed Architecture
**Diagram**: High-level architecture showing major components and data flow
**Component Overview**: Table of components, their technology, responsibility, scaling strategy

---

## API Design
For each endpoint:
- **`METHOD /path`**: Purpose, auth requirements
- **Request**: Schema/format
- **Response**: Success and error formats
- **Validation**: What's checked
- **Error Codes**: 400, 401, 404, 500, etc.

---

## Data Models
For each model:
- **Schema**: Table/collection structure with types and constraints
- **Indexes**: Which columns/fields, for what query patterns
- **Migrations**: What needs to change in the database

---

## Architecture Decisions (ADRs)
For each major decision:
- **Decision**: What we're deciding
- **Options**: 2-3 alternatives considered with pros/cons
- **Choice**: Which we chose and why
- **Trade-offs**: What we're accepting

---

## Security
- **Auth/Authz**: How users authenticate, what permissions exist
- **Input Validation**: Server-side validation approach
- **Data Protection**: Encryption at rest/transit, PII handling
- **Security Testing**: OWASP checks, dependency scanning

---

## Performance & Scale
- **Targets**: Response time, throughput, uptime SLAs
- **Optimization**: Caching strategy, database optimization, API efficiency
- **Scaling**: How we scale horizontally/vertically, bottlenecks and mitigation

---

## Error Handling & Reliability
- **Error Categories**: User errors (4xx) vs system errors (5xx)
- **Retry Strategy**: When to retry, backoff configuration
- **Fallback Behavior**: What happens when dependencies fail

---

## Monitoring
- **Key Metrics**: Request rate, error rate, latency (with alert thresholds)
- **Logging**: What to log, what NOT to log, retention
- **Dashboards**: Links to system health and feature analytics dashboards
- **Alerts**: Critical vs warning alerts and routing

---

## Testing
- **Unit Tests**: Coverage targets, what to test
- **Integration Tests**: API flows, database operations
- **E2E Tests**: Critical user flows
- **Performance Tests**: Load/stress/soak test scenarios

---

## Implementation Plan
For each phase:
- **Goal**: What we achieve in this phase
- **Tasks**: Key work items
- **Deliverables**: What's shipped
- **Acceptance Criteria**: How we know it's done

---

## Deployment
- **Environments**: Dev, staging, production setup
- **Process**: Automated steps from merge to production
- **Feature Flags**: Gradual rollout strategy
- **Rollback Plan**: When to rollback and how

---

## Database Migrations
For each migration:
- **Forward**: Script to apply changes
- **Rollback**: Script to undo changes
- **Data Migration**: How to backfill/transform existing data
- **Validation**: How to verify success

---

## Operations & Runbook
- **Common Operations**: Scaling, cache clearing, etc.
- **Incident Response**: Symptoms, likely causes, first steps

---

## Dependencies & Risks
- **External Dependencies**: Services we rely on, SLAs, fallback strategies
- **Technical Risks**: Risk | Impact | Likelihood | Mitigation
- **Timeline Risks**: Critical path items, mitigation strategies

---

## Documentation
- [ ] API docs (OpenAPI/Swagger)
- [ ] Architecture diagrams
- [ ] Runbook for on-call
- [ ] Developer guide
```
