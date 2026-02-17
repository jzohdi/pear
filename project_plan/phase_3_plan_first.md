# Phase 3: Plan-first & testability

**Goal:** Every task **starts with a plan** (whiteboard/diagram walkthrough). Testability is part of the plan. The agent doesn’t jump into code without an explicit, agreed plan.

---

## Scope

### In scope

- **Plan-first workflow**
  - When the user gives a task, the agent’s first obligation is to produce a **plan** (and optionally a diagram).
  - Plan is shown to the user and must be approved (or revised) before the agent moves to “implementation” (Phase 4 workflow).

- **Plan structure**
  - Plan includes: high-level steps, components/areas of the codebase, and **how we’ll test** (e.g. “we’ll add unit tests for X,” “we’ll validate Y with integration test”).
  - **Diagrams**: Support both **Mermaid** (rendered in the TUI via a library) and **ASCII** in the plan. Plan text + diagram in the conversation; no separate whiteboard app.

- **Testability in the plan**
  - Each major part of the plan should have a “how we’ll test this” clause. The agent is prompted to never leave this out.
  - This sets up Phase 4 (TDD): the plan already commits to tests, so implementation can follow “tests first.”

- **TUI for plans**
  - Plan (and diagram if present) is clearly rendered (e.g. dedicated section or message type) and easy to approve or send back with feedback.

### Out of scope (for this phase)

- Actual test execution or TDD automation (Phase 4).
- Rich diagram editor; pasted/emitted diagrams (Mermaid, ASCII) are sufficient.
- Enforcing “only one plan per task” vs. “revised plan creates a new version”; we can keep it simple (one plan per task, revisions = edit and re-approve).

---

## Exit criteria

- [ ] For a given task, the agent produces a plan (with testability) before any implementation.
- [ ] User can approve or reject the plan and give feedback; agent revises until approved.
- [ ] Plan is visible and readable in the TUI (and optionally includes a diagram).
- [ ] “Implementation” phase does not start until the plan is approved.
- [ ] Mermaid diagrams are rendered in the TUI; ASCII diagrams are shown as-is.

---

## Design decisions (locked in)

- **Diagram format**: **Mermaid** (rendered in TUI) + **ASCII** (displayed in terminal).

---

## Open design questions

- **Plan template**: Fixed (Overview, Steps, Components, Test strategy) or free-form with guidelines?
- **Re-planning**: If the plan turns out wrong during implementation, “pause and revise plan” or “abort and new task”?
- **Scope of “task”**: One user message = one plan, or one task = “do A, B, C” with one plan for all?

---

## Implementation notes

- Plans can be a dedicated message or step type in the conversation model (from Phase 2). Storing them makes it easy to show “current plan” and “approved plan” in the UI.
- Prompt engineering: system prompt should state that the agent must output a plan with testability before writing code, and that the user will approve it.
