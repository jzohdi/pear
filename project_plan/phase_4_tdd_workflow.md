# Phase 4: TDD & interface-first workflow

**Goal:** Implementation follows **interfaces and tests first**, then code. The agent proposes interfaces and tests, you approve them, then it implements against those contracts. This keeps the agent aligned with your types and test expectations.

---

## Scope

### In scope

- **Interface-first**
  - For a given sub-task (from the approved plan), the agent first proposes **interfaces** (function signatures, types, modules). You approve or edit.
  - No implementation code until interfaces are approved.

- **Test-driven**
  - After interfaces, the agent proposes **tests** (failing or placeholder). You approve or edit.
  - Then the agent implements to satisfy the tests (and keeps tests green). Each implementation step is still subject to the Phase 2 approval loop.

- **Integration with plan**
  - The plan (Phase 3) already called out “how we’ll test”; here we turn that into concrete interfaces and test cases. Order of operations: Plan → Interfaces → Tests → Implementation. **Granularity is configurable:** user can toggle whether the unit of work is per file, per component/class, or per plan step.

- **Strict TDD**
  - Tests must be written to **fail first**; then implementation until green. No “tests and impl together.”

- **TUI**
  - Clear display of “current phase” (e.g. “Interfaces,” “Tests,” “Implementation”) and of the current proposal (interfaces snippet, test snippet, or diff). Approval flow reuses Phase 2. **Toggle** for TDD granularity (per file / per component / per plan step).

### Out of scope (for this phase)

- Automatic test runner integration (e.g. “run pytest and show results in TUI”)—nice-to-have; we can add in Phase 5.
- Enforcing that tests actually run and pass before “done”; we can start with “human runs tests locally” and add automation later.
- Full type-checking in the TUI (e.g. running mypy); again, Phase 5 or later.

---

## Exit criteria

- [ ] For each implementation unit (as defined by the plan), the agent proposes interfaces first; user approves before tests.
- [ ] Agent proposes tests next; user approves before implementation.
- [ ] Implementation steps are still gated by the Phase 2 approval loop.
- [ ] TUI makes it clear whether we’re in “interfaces,” “tests,” or “implementation” for the current unit.
- [ ] User can toggle TDD granularity (per file / per component / per plan step).
- [ ] Strict TDD: tests written to fail first, then implementation until green.

---

## Design decisions (locked in)

- **Unit of work**: **Configurable/toggleable**—per file, per component/class, or per plan step.
- **Test framework**: **Multi-framework;** infer from project or from config.
- **Type system**: **TypeScript** as primary (user’s main language); agent respects project type system.
- **Strict TDD**: Yes—tests must fail first, then implement until green.

---

## Implementation notes

- Conversation model may need “phase” or “stage” per sub-task: `interfaces_proposed`, `interfaces_approved`, `tests_proposed`, `tests_approved`, `implementation_in_progress`, `done`. This helps the agent and the TUI stay in sync.
- Storing approved interfaces and tests in session context gives the agent a clear “contract” to implement against and reduces drift.
