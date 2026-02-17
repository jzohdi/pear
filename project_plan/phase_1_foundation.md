# Phase 1: Foundation

**Goal:** A minimal but real TUI that can run an LLM agent session and display a conversation. No pair-programming workflow yet—just “it runs, we can talk, we can see output.”

---

## Scope

### In scope

- **TUI shell**
  - Terminal UI that renders a session (messages, agent output, maybe a simple input area).
  - Keyboard-driven where it makes sense (e.g. focus, scrolling).
  - Runs in the user’s terminal (no separate “app” window required).

- **Session model**
  - Notion of a “session”: one workspace/repo context, one conversation thread (or a clear model for threads).
  - Persistence: sessions can be saved and resumed (format TBD).

- **Basic LLM integration**
  - Connect to at least one LLM provider (e.g. OpenAI, Anthropic, or local).
  - Send user message + context (e.g. system prompt, recent messages); display assistant reply in the TUI.
  - No multi-step agentic loop yet—single request/response is enough for this phase.

- **Minimal “project” awareness (optional for MVP)**
  - Ability to specify a project root (directory) so the agent knows the codebase path. No file editing or tool use in Phase 1 unless we explicitly add a minimal slice.

- **Project sandboxing (required)**
  - Pear TUI and LLM agent are **sandboxed to the current project folder only**. No read/write/execute outside that directory. Network only for LLM API and config. This is a non-negotiable constraint from day one.

### Out of scope (for this phase)

- Step-by-step approval flow.
- Whiteboarding, diagrams, or structured planning UI.
- File editing, shell commands, or tools.
- TDD/interface-first workflow enforcement.
- Multiple concurrent sessions or complex session management.

---

## Exit criteria

- [ ] User can start Pear and see a TUI.
- [ ] User can type a message and get an LLM response shown in the TUI.
- [ ] Session can be persisted and resumed (even if format is simple).
- [ ] One LLM provider works end-to-end (configurable, no hardcoding of keys in code).
- [ ] All file and process access is restricted to the project root (sandbox enforced).

---

## Design decisions (locked in)

- **Tech stack**: **Python** + Textual (or equivalent). Python chosen for agent/LLM ecosystem, documentation, and community; TUI library TBD but Textual is the leading option.
- **Sessions**: Multiple sessions over time; **one active session at a time**. Persistence format TBD (e.g. one file per session).
- **LLM provider**: **Anthropic** first. Implement **provider abstraction** if effort is reasonable; use [OpenCode](https://github.com/anomalyco/opencode) for inspiration on provider abstraction and API design. If abstraction is too high effort, defer and ship Anthropic only.
- **Config**: TBD—where config lives and what it contains (API key, model, project root).
- **Sandboxing**: Mandatory; see “Project sandboxing” in scope.

---

## Implementation notes

- Keep the “conversation” data model simple: list of messages with role (user/assistant/system) and content. This will extend in Phase 2 for structured steps and approval.
- TUI should be structured so we can later add panels (e.g. plan view, diff view, approval buttons) without a full rewrite.
