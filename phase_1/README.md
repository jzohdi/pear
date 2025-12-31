# Phase 1: CLI Prototype

**Duration**: 2-3 weeks  
**Goal**: Validate that the structured pair-programming workflow feels right before investing in the VS Code fork.

---

## Document Index

This directory contains the complete specification for Phase 1 of Pear. Each document is self-contained and focuses on a specific aspect of the implementation.

### Core Documentation

| # | Document | Description |
|---|----------|-------------|
| 00 | [Overview](./00-overview.md) | Objectives, success criteria, and scope |
| 01 | [Architecture](./01-architecture.md) | High-level architecture and component relationships |
| 02 | [Components](./02-components.md) | Detailed specifications for each component |
| 03 | [Data Models](./03-data-models.md) | TypeScript interfaces and type definitions |
| 04 | [LLM Prompts](./04-llm-prompts.md) | System prompts for each workflow phase |
| 05 | [Response Parsing](./05-response-parsing.md) | How to parse and extract data from LLM responses |
| 06 | [Context Management](./06-context-management.md) | Token budgeting and conversation history |
| 07 | [Phase Transitions](./07-phase-transitions.md) | State machine and go-back logic |
| 08 | [User Interface](./08-user-interface.md) | CLI layout and user interactions |
| 09 | [File Structure](./09-file-structure.md) | Project organization and generated files |
| 10 | [Implementation Plan](./10-implementation-plan.md) | Weekly schedule and milestones |
| 11 | [Testing Strategy](./11-testing-strategy.md) | Unit, integration, and manual testing |
| 12 | [Error Handling](./12-error-handling.md) | Error types and recovery strategies |
| 13 | [Exit Criteria](./13-exit-criteria.md) | Completion requirements and deliverables |

### Supporting Files

| Directory | Contents |
|-----------|----------|
| `drafts/` | Working drafts and notes (not for implementation) |

---

## Quick Start

If you're implementing Phase 1, read the documents in this order:

1. **[00-overview.md](./00-overview.md)** — Understand what we're building and why
2. **[01-architecture.md](./01-architecture.md)** — See how components fit together
3. **[03-data-models.md](./03-data-models.md)** — Define your TypeScript types first
4. **[10-implementation-plan.md](./10-implementation-plan.md)** — Follow the day-by-day plan

---

## The Four-Phase Workflow

Phase 1 validates this core workflow:

```
┌──────────┐    ┌───────────┐    ┌────────────────┐    ┌─────────┐
│ Planning │───▶│ Interface │───▶│ Implementation │───▶│ Testing │───▶ Complete
└──────────┘    └───────────┘    └────────────────┘    └─────────┘
     │               │                  │                   │
     ▼               ▼                  ▼                   ▼
 DESIGN.md       types.ts           index.ts        TEST_EVIDENCE.md
               tests (skeleton)    tests (impl)
```

Each phase has:
- **Entry criteria**: What must be true to start
- **Human checkpoint**: Approval required to proceed
- **Artifacts**: Files created/modified
- **Exit criteria**: What must be true to finish

---

## Key Principles

1. **Tight Feedback Loops** — No autonomous work exceeds a single reviewable unit
2. **Artifact-Driven** — Each phase produces concrete files that persist in the repo
3. **Reversibility** — Can go back to earlier phases without losing work
4. **Locality of Behavior** — Design docs live with the code they describe

---

## Technology Decisions

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Language | TypeScript | Type safety, async support |
| LLM Provider | Anthropic Claude | Best reasoning for code |
| CLI Framework | Inquirer | Interactive prompts |
| Test Framework | Vitest | Fast, TypeScript-native |
| Storage | YAML files | Human-readable, matches README spec |

---

## Success Looks Like

After completing Phase 1:

- [ ] Built one real feature using the CLI
- [ ] All four phases feel natural and productive
- [ ] LLM output is usable with <20% revision needed
- [ ] Can pause and resume sessions reliably
- [ ] Clear list of improvements for Phase 2

