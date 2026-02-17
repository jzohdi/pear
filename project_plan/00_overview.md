# Pear — Project plan overview

This folder contains the phased implementation plan for Pear, the pair-programming LLM agent TUI. Each phase builds on the previous one.

## Design principles (reference)

- **Pair programming feel**: Human stays in the loop; no big opaque handoffs.
- **Plan first**: Whiteboard/diagram and testability baked into the plan.
- **Interface + TDD first**: Types and tests before implementation.
- **Nothing glossed over**: Every step is discussed and approved by the human.
- **Project sandboxed**: Pear and the agent operate only within the current project folder; no access outside (except LLM API and config).

## Phase summary

| Phase | Focus | Outcome |
|-------|--------|---------|
| 1 | Foundation | TUI shell, session model, basic LLM integration |
| 2 | Conversation & pair-programming loop | Turn-taking, step approval, “nothing glossed over” |
| 3 | Plan-first & testability | Whiteboarding, diagrams, testability in plans |
| 4 | TDD & interface-first workflow | Interfaces and tests before implementation |
| 5 | Polish & ergonomics | UX, performance, production readiness |

## Locked-in decisions

- **Stack**: Python + Textual (or equivalent). Anthropic first; provider abstraction if feasible (see [OpenCode](https://github.com/anomalyco/opencode) for inspiration).
- **Sessions**: Multiple over time; one active at a time. Sandbox: project folder only.
- **Steps**: Small (max lines per file); streaming within step; approve / reject / accept-with-changes (+ optional “remember this fix”).
- **Diagrams**: Mermaid (rendered in TUI) + ASCII. **TDD**: Strict (fail first, then green); granularity toggle (per file / component / plan step). **Primary language**: TypeScript. Local LLMs later; differentiate from existing tools.

Full list: [design_questions.md](design_questions.md).

## Open design questions

Remaining open items (config location, plan template, re-planning, task scope, Phase 5 priorities) are in each phase doc and in [design_questions.md](design_questions.md).

## How to use these docs

- Read **00_overview.md** (this file) and the README first.
- Work through phases in order; each phase doc has goals, scope, and exit criteria.
- Use the “Design questions” and “Out of scope / later” sections to keep scope clear and avoid creep.
