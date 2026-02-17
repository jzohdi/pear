# Phase 2: Conversation & pair-programming loop

**Goal:** The agent works in **steps** and **pauses for your approval** before continuing. Nothing is glossed over—every meaningful action is proposed, shown, and explicitly accepted (or rejected/edited) by you.

---

## Scope

### In scope

- **Structured turns**
  - Agent outputs are structured into “steps” (e.g. “here’s my plan for this sub-task,” “here’s a proposed diff,” “here’s what I’m about to run”).
  - Each step is presented clearly in the TUI and requires an explicit continue/approve (or reject/edit) from the user before the agent proceeds.

- **Step size**
  - Steps are **small**: maximum number of lines (in a single file) per step, configurable. One approval per step; no giant multi-file dumps.

- **Step types (minimal set for this phase)**
  - **Message/explanation**: Agent explains or asks; user reads and continues.
  - **Proposed change**: Agent shows a diff or edit; user can **Approve**, **Reject** (with short reason; agent tries again), or **Accept with changes** (developer edits the proposal manually; agent is informed and should learn/remember—optional “remember this fix” so the agent stores the correction for future use).
  - Optional: **Confirmation** (e.g. “I will run X; proceed?”). Can be folded into “proposed change” or “message” if we want to keep the model small.

- **TUI for steps**
  - Steps are visible in the conversation (e.g. collapsible or clearly separated).
  - **Streaming**: Agent’s reply streams within the current step; the *next* step is still gated on approval.
  - Clear affordance for “Approve / Reject / Accept with changes” so the user never feels the agent “ran away” with the task.
  - No mid-step interrupt: agent finishes presenting the step before user acts (steps are small enough).

- **Context and history**
  - Agent receives conversation history (and possibly project context) so it can refer to prior steps and your feedback. No “memory” beyond the current session for this phase.

### Out of scope (for this phase)

- Automatic generation of diagrams or whiteboards (Phase 3).
- Enforcing “plan first” or “tests first” (Phases 3–4).
- Rich code editing inside the TUI (e.g. inline editor); showing diffs and allowing approve/reject is enough.

---

## Exit criteria

- [ ] Agent emits discrete “steps” (not one long stream until done).
- [ ] User can approve or reject (and optionally give feedback on) each step before the agent continues.
- [ ] Rejection or feedback is sent back to the agent and the next response respects it.
- [ ] TUI makes the current “waiting for approval” state obvious.
- [ ] User can “Accept with changes” (edit the proposal) and optionally tell the agent to “remember this fix.”
- [ ] Agent reply streams within a step; next step requires approval.

---

## Design decisions (locked in)

- **Step granularity**: Small; max lines per **single file** per step (configurable).
- **Rejection**: Reject + short reason → agent retries. Accept with changes → developer edits; agent learns/remembers; optional “remember this fix” to persist the correction.
- **Streaming**: Yes within step; next step gated on approval.
- **Interruption**: Agent finishes presenting the step; no mid-step interrupt.

---

## Implementation notes

- Response parsing: we need a way to know when the agent has “finished” one step and is waiting. Options: explicit delimiters in the prompt (e.g. `<!-- STEP_END -->`), structured output (e.g. JSON blocks), or heuristics. Design choice affects reliability.
- Persistence: store steps and approval state so a resumed session shows “we’re waiting for approval on step 5.”
