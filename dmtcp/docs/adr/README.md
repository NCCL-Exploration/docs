# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) that document significant architectural decisions made during DMTCP's development. Each ADR captures the context, decision, and consequences of important design choices.

## ADR Index

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [ADR-001](./001-ld-preload-wrappers.md) | LD_PRELOAD-based Wrapper Mechanism | Accepted | 2006-01-15 |
| [ADR-002](./002-coordinator-architecture.md) | Coordinator-based Architecture | Accepted | 2006-02-20 |
| [ADR-003](./003-plugin-system.md) | Plugin System Design | Accepted | 2008-06-10 |
| [ADR-004](./004-seven-stage-checkpoint.md) | Seven-Stage Checkpoint Algorithm | Accepted | 2009-03-15 |
| [ADR-005](./005-pid-virtualization.md) | PID Virtualization Strategy | Accepted | 2009-05-20 |
| [ADR-006](./006-split-process-mana.md) | Split-Process Architecture for MANA | Accepted | 2019-07-25 |

## ADR Template

When creating a new ADR, use this template:

```markdown
# ADR-XXX: [Title]

## Status
[Proposed/Accepted/Deprecated/Superceded]

## Context
[Describe the context and problem statement]

## Decision
[Describe the decision that was made]

## Consequences
[Describe the consequences of applying this decision]

## Implementation
[Describe how the decision was implemented]

## Alternatives Considered
[List and describe alternatives that were considered]
```

## ADR Process

1. **Proposal**: Create ADR as "Proposed" status
2. **Discussion**: Review with core developers
3. **Decision**: Update status to "Accepted" or "Rejected"
4. **Implementation**: Reference ADR in code commits
5. **Updates**: Modify ADR if decision evolves

## Contributing

When making significant architectural changes:

1. Create an ADR before implementation
2. Discuss the ADR with the community
3. Reference the ADR in pull requests
4. Update the ADR as implementation progresses

This ADR system helps maintain DMTCP's architectural integrity and provides historical context for design decisions.