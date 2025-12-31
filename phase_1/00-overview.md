# Phase 1 Overview

**Document**: `00-overview.md`  
**Purpose**: Define objectives, success criteria, and scope for Phase 1

---

## What We're Building

Phase 1 is a **CLI prototype** that validates the structured pair-programming workflow before investing in the full VS Code fork. It's intentionally minimal—just enough to experience the workflow on a real feature.

The CLI will guide a developer through four phases:
1. **Planning** — Collaborate with LLM to produce a design document
2. **Interface** — Define types and test cases before implementation
3. **Implementation** — Implement units one at a time with human review
4. **Testing** — Run tests, debug failures, complete manual verification

---

## Objectives

### Primary Objectives

| # | Objective | How We'll Know |
|---|-----------|----------------|
| 1 | **Validate the workflow** | Complete a real feature using all four phases |
| 2 | **Test LLM prompts** | LLM output is usable without major revision |
| 3 | **Feel the feedback loops** | Each approval feels valuable, not tedious |
| 4 | **Identify friction points** | Document what doesn't work for Phase 2 |

### Secondary Objectives

| # | Objective | How We'll Know |
|---|-----------|----------------|
| 1 | Build reusable code | Components can be extracted to core library |
| 2 | Document LLM behavior | Notes on what prompts work/don't work |
| 3 | Establish patterns | Reference implementation for future phases |

---

## Success Criteria

| Criteria | Target | Measurement |
|----------|--------|-------------|
| Feature completion | 1+ features | Real feature works, tests pass |
| Workflow productivity | Positive | Would you use this again? (subjective) |
| No phase blockers | 0 | Each phase completes without manual workarounds |
| LLM output quality | <20% revision | Percentage of output requiring significant changes |
| Time efficiency | <2x manual | Compared to coding without Pear |

### What "Success" Means

Phase 1 succeeds if we can answer these questions:

1. **Does the four-phase workflow make sense?**  
   The progression from planning → interface → implementation → testing should feel natural.

2. **Is unit-by-unit approval the right granularity?**  
   Reviewing each function individually should feel productive, not tedious.

3. **Can the LLM follow the design?**  
   The implementation should match the approved design document.

4. **Is the feedback loop tight enough?**  
   You should never feel lost or disconnected from what the LLM is doing.

---

## Scope

### In Scope

These features will be implemented in Phase 1:

| Category | Features |
|----------|----------|
| **Workflow** | All four phases (Planning, Interface, Implementation, Testing) |
| | Completion handling (session archival, summary) |
| | Go-back transitions with state preservation |
| **LLM** | Single provider (Anthropic Claude) |
| | Streaming responses |
| | Phase-specific prompts |
| **Language** | TypeScript only |
| **Testing** | Vitest only |
| | Test implementation in Phase 3 |
| | Manual test recording |
| | TEST_EVIDENCE.md generation |
| **Persistence** | JSON session storage |
| | Checkpoint/resume |
| | Crash recovery (backup checkpoints) |
| **UI** | Terminal-based CLI |
| | Syntax-highlighted code display |
| | Progress indicators |

### Out of Scope

These features are deferred to later phases:

| Feature | Reason | Target Phase |
|---------|--------|--------------|
| Multiple LLM providers | Adds complexity without validation value | Phase 2 |
| Multiple languages | TypeScript sufficient for validation | Phase 2 |
| Folder locking | Single user is sufficient | Phase 2 |
| Project analysis/onboarding | Not needed for CLI validation | Phase 2 |
| Git integration | Manual commits acceptable | Phase 3 |
| VS Code UI | This is what we're validating toward | Phase 2+ |

---

## Constraints

### Technical Constraints

| Constraint | Impact |
|------------|--------|
| Single user | No concurrent sessions on same project |
| TypeScript only | LLM prompts optimized for TS patterns |
| Vitest only | Test runner integration is single-framework |
| Local storage | Sessions are machine-specific |

### Design Constraints

| Constraint | Rationale |
|------------|-----------|
| Human approval at every checkpoint | Core principle we're validating |
| One feature per session | Maintains focus, prevents context pollution |
| Design doc always in context | Source of truth for LLM decisions |
| Artifacts in feature folder | Locality of behavior principle |

---

## Timeline

**Estimated Duration**: 2-3 weeks

| Week | Focus | Exit State |
|------|-------|------------|
| 1 | Foundation | State manager, LLM client, basic CLI working |
| 2 | Core Workflow | All four phases functional |
| 3 | Polish & Validation | One real feature built, bugs fixed |

See [10-implementation-plan.md](./10-implementation-plan.md) for detailed daily schedule.

---

## Dependencies

### External Dependencies

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.30.0",
    "chalk": "^5.3.0",
    "inquirer": "^9.2.0",
    "marked": "^12.0.0",
    "marked-terminal": "^7.0.0",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "@types/inquirer": "^9.0.0",
    "@types/node": "^20.0.0",
    "@types/uuid": "^9.0.0",
    "typescript": "^5.4.0",
    "vitest": "^1.4.0",
    "tsx": "^4.7.0"
  }
}
```

### Environment Requirements

| Requirement | Details |
|-------------|---------|
| Node.js | v18+ (LTS recommended) |
| API Key | `ANTHROPIC_API_KEY` environment variable |
| File System | Write access to project directory |

---

## Risks Summary

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| LLM output quality | Medium | High | Iterate prompts, document versions |
| Workflow feels tedious | Medium | High | Track approval times, consider batching |
| State corruption | Low | High | Checkpoints, backup files |
| Context window limits | Low | Medium | Sliding window, summarization |

See [12-error-handling.md](./12-error-handling.md) for detailed error strategies.

---

## Related Documents

- [01-architecture.md](./01-architecture.md) — System design
- [10-implementation-plan.md](./10-implementation-plan.md) — Day-by-day schedule
- [13-exit-criteria.md](./13-exit-criteria.md) — Completion requirements

