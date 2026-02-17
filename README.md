# Pear

**The pair-programming TUI for LLM agents.**

Pear is an LLM agent that runs in the terminal and is designed so that working with it feels like **pair programming with another developer**—you stay in the loop at every step instead of reviewing a large, opaque diff at the end.

---

## The problem

Current “vibe coding” workflows often leave the developer too far out of the loop:

- When the agent finishes a task, **reviewing the code is cumbersome** and hard to reason about.
- The agent can **drift from how you would implement** the same solution.
- There’s no natural moment to **steer, correct, or align** on design and style as work happens.

Pear is built so that you’re **closely tied to all of the agent’s work**: design, interfaces, tests, and implementation are done *with* you, not *for* you after the fact.

---

## Core philosophies

These principles are baked into how Pear structures tasks and conversations:

1. **Plan before code**  
   Every task starts with whiteboarding or a diagram walkthrough. Plans are explicit, and **testability is part of the design** from the start.

2. **Interfaces and tests first**  
   Implementation begins with programming interfaces and test-driven development. A clear type system and tests make LLM-generated code more maintainable and reliable.

3. **Nothing is glossed over**  
   All code, design, and decisions are **talked through with you**. You are never left behind; the agent explains, proposes, and waits for your input at each meaningful step.

---

## What Pear is (and isn’t)

- **TUI-first**: Terminal user interface as the primary experience (keyboard-driven, fast, stays in your existing workflow).
- **Conversation-driven**: The agent proposes steps, shows diffs, asks for approval or changes, and only proceeds when you’re aligned.
- **Structured workflow**: Built-in phases (plan → interfaces/tests → implementation) so sessions feel like pair programming with a disciplined partner.
- **Project-sandboxed**: The Pear TUI and the LLM agent are **sandboxed to the current project folder only**. They never read, write, or execute outside that directory. No access to the rest of the filesystem or the network except as required for LLM API calls and config.

Pear is designed to **differentiate from** existing AI coding tools (e.g. [OpenCode](https://opencode.ai/), Cursor, Aider, Continue) by making the pair-programming loop and plan-first, TDD workflow the core product—not an optional mode.

Implementation details, tech stack, and phased rollout are described in the **project plan** under `project_plan/`.

---

## Project plan

The implementation is broken into phases in the `project_plan/` directory:

- **[Phase 1: Foundation](project_plan/phase_1_foundation.md)** — TUI shell, session model, basic LLM integration.
- **[Phase 2: Conversation & Pair-Programming Loop](project_plan/phase_2_conversation.md)** — Turn-taking, step-by-step approval, “nothing glossed over.”
- **[Phase 3: Plan-First & Testability](project_plan/phase_3_plan_first.md)** — Whiteboarding, diagrams, encoding testability into plans.
- **[Phase 4: TDD & Interface-First Workflow](project_plan/phase_4_tdd_workflow.md)** — Interfaces and tests before implementation.
- **[Phase 5: Polish & Ergonomics](project_plan/phase_5_polish.md)** — UX, performance, and production readiness.

Start with [Phase 1](project_plan/phase_1_foundation.md) for scope and success criteria. Open design questions that affect the plan are collected in [Design questions](project_plan/design_questions.md).

---

## Contributing & status

This project is in early design and planning. See `project_plan/` for the current roadmap and open design questions.
