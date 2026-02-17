# Phase 5: Polish & ergonomics

**Goal:** Pear feels solid for daily use: fast, predictable, and pleasant. Production-ready in the sense of “a developer can rely on it every day.”

---

## Scope (candidates; to be refined)

- **Performance**
  - Responsive TUI even with long conversations and large diffs (e.g. virtualized scrolling, lazy load).
  - LLM latency: streaming (already in Phase 2?) and clear “thinking” or “waiting” states.

- **Test runner integration (optional)**
  - Run project tests from Pear (e.g. `pytest`, `go test`, `npm test`) and show results in the TUI. Failing tests can be tied back to the approval flow (“tests failed; revise implementation?”).

- **Type-checking / lint (optional)**
  - Option to run project type-checker or linter and surface errors in the TUI so the agent (and you) can fix before “done.”

- **Session and project UX**
  - Multiple sessions, naming, switching. Clear “current project root” and “current session.”
  - Search or scroll in conversation history. Maybe “jump to step” or “jump to last approval.”

- **Configuration and safety**
  - API keys and secrets not in code; clear config story. Optional “dry run” or “confirm before running shell commands” if we have tool use.

- **Error handling and recovery**
  - Agent errors (e.g. API failure, parse failure) don’t crash the TUI; user gets a clear message and can retry or edit.
  - Corrupted or partial session recovery where possible.

- **Accessibility and ergonomics**
  - Keyboard-first, consistent shortcuts. Optional theming (e.g. light/dark). No hard requirement for full screen reader support in v1, but avoid obvious blockers.

---

## Exit criteria (TBD)

- [ ] TUI remains responsive with long sessions and large content.
- [ ] Session and project handling is clear and non-frustrating.
- [ ] Error states are visible and recoverable.
- [ ] At least one “nice-to-have” (test runner, type-check, or similar) is implemented if we committed to it for Phase 5.

---

## Design questions

1. **Priority**: What hurts most today in your workflow—latency, session management, or something else? That can set Phase 5 priorities.
2. **Test runner**: Must we run tests inside Pear, or is “agent suggests command, you run in another terminal” acceptable for v1?
3. **Tool use**: Will the agent ever run shell commands or edit files on disk in this phase? If yes, we need a clear “confirm before run” and sandboxing story.

---

## Implementation notes

- Phase 5 is the place to fix tech debt from Phases 1–4 (e.g. refactor response parsing, unify step types). Keep a short list so we don’t forget.
