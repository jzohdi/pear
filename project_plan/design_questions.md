# Design questions for Pear

Decisions are recorded here as we lock them in. Unresolved items are still in the phase docs.

---

## Phase 1: Foundation

| Question | Decision |
|----------|----------|
| **Tech stack** | **Python** + Textual (or equivalent TUI lib). Chosen for: agent/LLM training data and docs, strong ecosystem (HTTP, async, parsing), mature TUI (Textual), and community. Go would give a single binary and speed but a smaller TUI/LLM ecosystem; Node has good SDKs but weaker TUI maturity. |
| **Session persistence** | Multiple sessions over time; **only one session active at a time**. User works closely with a single session; other sessions are listable/resumable. Format TBD (e.g. one file per session under project or config dir). |
| **LLM provider** | **Anthropic** as primary. Implement **provider abstraction** if effort is reasonable; otherwise defer. Use [OpenCode](https://opencode.ai/) / [anomalyco/opencode](https://github.com/anomalyco/opencode) for inspiration on multi-provider abstraction. |
| **Config** | TBD (env vars, project config, `~/.config/pear`). To define in Phase 1. |
| **Sandboxing** | **Mandatory.** Pear TUI and agent are sandboxed to the **current project folder only**. No filesystem or execution outside that directory (except LLM API calls and minimal config). |

---

## Phase 2: Conversation & pair-programming loop

| Question | Decision |
|----------|----------|
| **Step granularity** | **Small:** max number of lines in a **single file** per step (exact cap TBD, likely configurable). |
| **Rejection flow** | **Reject + short reason** → agent tries again. **Accept with changes** → developer can manually edit the proposed change; agent is informed and should learn/remember what was changed. Optional: developer can confirm “remember this fix” so the agent stores the correction for future steps/sessions. |
| **Streaming** | **Yes:** stream the agent’s reply within a step; next step still gated on approval. |
| **Interruption** | Agent **finishes presenting** the current step before user can approve/reject. No mid-step interrupt (steps are small enough that this is acceptable). |

---

## Phase 3: Plan-first & testability

| Question | Decision |
|----------|----------|
| **Diagram format** | **Mermaid + ASCII.** Render **Mermaid in the TUI** (e.g. via a library). Also support ASCII diagrams. |
| **Plan template** | TBD (fixed vs free-form). |
| **Re-planning** | TBD. |
| **Scope of “task”** | TBD. |

---

## Phase 4: TDD & interface-first workflow

| Question | Decision |
|----------|----------|
| **Unit of work** | **Configurable/toggleable.** User can choose granularity: per file, per component/class, or per plan step. |
| **Test framework** | **Multi-framework:** infer from project or from config. |
| **Type system** | Default to **TypeScript**-friendly (primary user language); agent respects project type system. |
| **Strict TDD** | **Yes:** tests must be written to **fail first**, then implementation until green. |

---

## Phase 5: Polish & ergonomics

| Question | Decision |
|----------|----------|
| **Priority** | TBD. |
| **Test runner** | TBD. |
| **Tool use** | Any file edits or shell runs must respect **project-folder sandbox** and confirm-before-run where appropriate. |

---

## Cross-cutting

| Question | Decision |
|----------|----------|
| **Primary language** | **TypeScript** (user’s main language). Defaults for test runner, type checker, and prompts should be TS-friendly. |
| **Offline / local LLMs** | **Later** (not in v1). |
| **Existing tools** | **Differentiate** from existing tools (OpenCode, Cursor, Aider, Continue); no integration requirement. |

---

*Add new answers or open questions below as we go.*
