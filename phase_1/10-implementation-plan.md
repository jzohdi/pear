# Implementation Plan

**Document**: `10-implementation-plan.md`  
**Purpose**: Day-by-day implementation schedule and milestones

---

## Overview

**Total Duration**: 3 weeks (15 working days)  
**Buffer**: Week 3 includes buffer for iteration and real-world testing

---

## Week 1: Foundation

Build the core infrastructure: state management, LLM integration, and basic CLI.

### Day 1: Project Setup

**Goal**: Working TypeScript project with proper configuration

**Tasks**:
- [ ] Initialize npm project with package.json
- [ ] Configure TypeScript (tsconfig.json)
- [ ] Set up Vitest for testing
- [ ] Create directory structure
- [ ] Set up ESLint
- [ ] Create .env.example with ANTHROPIC_API_KEY
- [ ] Write basic entry point (src/index.ts)

**Deliverable**: `pear --version` works

**Checkpoint**:
```bash
npm run build    # Compiles successfully
npm run test     # Test framework runs (no tests yet)
tsx src/index.ts # Entry point executes
```

---

### Day 2: State Manager + Storage

**Goal**: Sessions persist to JSON and can be recovered

**Tasks**:
- [ ] Define all types in src/types/index.ts
- [ ] Implement StateManager class
  - [ ] createSession()
  - [ ] getSession()
  - [ ] updateSession()
- [ ] Implement JSON storage
  - [ ] saveCheckpoint()
  - [ ] loadCheckpoint()
  - [ ] restoreFromBackup()
- [ ] Add session listing (listIncompleteSessions)
- [ ] Write unit tests

**Deliverable**: Can create, save, load, and list sessions

**Checkpoint**:
```typescript
// This should work:
const manager = new StateManager();
const session = manager.createSession({ featureName: 'Test' });
await manager.saveCheckpoint(session);
const loaded = await manager.loadCheckpoint(session.sessionId);
assert(loaded.featureName === 'Test');
```

---

### Day 3: LLM Client + Anthropic Integration

**Goal**: Can send prompts and receive streamed responses

**Tasks**:
- [ ] Install @anthropic-ai/sdk
- [ ] Implement LLMClient class
  - [ ] chat() method with streaming
  - [ ] Error handling (rate limits, connection errors)
  - [ ] Retry logic with exponential backoff
- [ ] Create system prompts (src/llm/prompts.ts)
- [ ] Test with real API calls (manual)

**Deliverable**: Can stream a response from Claude

**Checkpoint**:
```typescript
// This should work:
const client = new LLMClient();
for await (const chunk of client.chat({
  systemPrompt: 'You are a helpful assistant.',
  messages: [{ role: 'user', content: 'Hello!' }],
})) {
  process.stdout.write(chunk);
}
```

---

### Day 4: Response Parser + Artifacts

**Goal**: Can extract code blocks and file paths from LLM output

**Tasks**:
- [ ] Implement parseCodeBlocks()
- [ ] Implement parseArtifacts()
- [ ] Implement parseSections()
- [ ] Handle edge cases (no file path, multiple blocks, etc.)
- [ ] Write comprehensive unit tests

**Deliverable**: Parser handles all expected LLM output formats

**Checkpoint**:
```typescript
const response = `
Here's the code:

\`\`\`typescript
// src/utils/helper.ts
export function helper() { return true; }
\`\`\`
`;

const artifacts = parseArtifacts(response);
assert(artifacts[0].path === 'src/utils/helper.ts');
assert(artifacts[0].content.includes('helper'));
```

---

### Day 5: CLI Handler + Display

**Goal**: Interactive terminal prompts with formatted output

**Tasks**:
- [ ] Set up inquirer for prompts
- [ ] Implement CLIHandler class
  - [ ] displayPhaseHeader()
  - [ ] displayLLMResponse() (with markdown rendering)
  - [ ] displayStreamingResponse()
  - [ ] promptForAction()
  - [ ] promptForFreeformInput()
- [ ] Set up chalk for colors
- [ ] Set up marked + marked-terminal for markdown
- [ ] Test interactive flow

**Deliverable**: CLI displays formatted output and accepts input

**Checkpoint**:
```bash
tsx src/index.ts new
# Should prompt for feature description
# Should display formatted output
```

---

## Week 2: Core Workflow

Implement all four phases with human approval gates.

### Day 6: Planning Phase

**Goal**: Complete Phase 1 end-to-end

**Tasks**:
- [ ] Implement planning phase handler
- [ ] Wire up LLM for clarifying questions
- [ ] Wire up LLM for design document generation
- [ ] Implement Context Manager (basic version)
- [ ] Handle approve/modify flow
- [ ] Save DESIGN.md artifact
- [ ] Integrate with CLI for full flow

**Deliverable**: Can start session, answer questions, get design doc, approve

**Checkpoint**:
```bash
pear new
# Enter feature description
# Answer clarifying questions
# Approve design document
# File created: src/features/{feature}/DESIGN.md
```

---

### Day 7: Interface Phase

**Goal**: Generate types and test skeletons

**Tasks**:
- [ ] Implement interface phase handler
- [ ] Wire up LLM for type generation
- [ ] Wire up LLM for test case generation
- [ ] Implement dependency analysis prompting
- [ ] Handle approve/modify for each step
- [ ] Save types.ts and test file artifacts

**Deliverable**: Types and test skeletons generated and saved

**Checkpoint**:
```bash
# After planning approval:
# Approve types
# Approve tests
# Approve implementation order
# Files created: types.ts, __tests__/{feature}.test.ts
```

---

### Day 8: Implementation Phase (Production Units)

**Goal**: Implement production code unit by unit

**Tasks**:
- [ ] Implement implementation phase handler
- [ ] Create units from dependency analysis
- [ ] Implement unit-by-unit LLM prompting
- [ ] Show progress (3/5 units)
- [ ] Handle approve/modify per unit
- [ ] Accumulate code in appropriate files

**Deliverable**: Can implement and approve production units

**Checkpoint**:
```bash
# After interface approval:
# See progress: "Unit 1/5: getOAuthUrl()"
# Approve each unit
# Code accumulated in implementation files
```

---

### Day 9: Implementation Phase (Test Units)

**Goal**: Implement test code after production code

**Tasks**:
- [ ] Extend implementation phase for tests
- [ ] Track test implementations separately
- [ ] Switch stage from 'production' to 'tests'
- [ ] LLM prompting for test implementations
- [ ] Update test file with implementations

**Deliverable**: Test skeletons filled in with real assertions

**Checkpoint**:
```bash
# After production units:
# "Now implementing tests"
# See progress: "Test 1/8: should return valid URL"
# Approve each test
# Tests have real implementations
```

---

### Day 10: Test Runner Integration

**Goal**: Run tests and show results

**Tasks**:
- [ ] Implement TestRunner class
- [ ] Detect Vitest (vitest.config.ts)
- [ ] Run tests via child_process
- [ ] Parse JSON output
- [ ] Display results in CLI
- [ ] Handle test runner errors

**Deliverable**: Can run tests and see pass/fail

**Checkpoint**:
```bash
# After implementation:
# "Running tests..."
# "✓ 10 passed, 2 failed"
# See failure details
```

---

## Week 3: Polish & Validation

Debug loop, manual tests, go-back transitions, and real-world validation.

### Day 11: Testing Phase (Debug Loop)

**Goal**: Debug failing tests with LLM assistance

**Tasks**:
- [ ] Implement testing phase handler
- [ ] Feed failures to LLM for analysis
- [ ] Display proposed fixes
- [ ] Apply fixes with approval
- [ ] Re-run tests after fixes
- [ ] Loop until all pass

**Deliverable**: Can debug test failures with LLM help

**Checkpoint**:
```bash
# After test run with failures:
# "Analyzing 2 failures..."
# See proposed fixes
# Approve fixes
# Re-run tests
# "✓ 12 passed, 0 failed"
```

---

### Day 12: Manual Tests + Completion

**Goal**: Record manual tests and complete session

**Tasks**:
- [ ] Generate manual test checklist (LLM)
- [ ] Implement manual test recording UI
- [ ] Implement completion handler
- [ ] Generate TEST_EVIDENCE.md
- [ ] Archive session to done/ folder
- [ ] Display completion summary

**Deliverable**: Full workflow from start to completion

**Checkpoint**:
```bash
# After automated tests pass:
# See manual test checklist
# Record each test result
# Generate evidence document
# "🎉 Feature Complete!"
# Session archived
```

---

### Day 13: Go-Back Transitions

**Goal**: Can return to earlier phases without losing work

**Tasks**:
- [ ] Implement goBackToPhase() in controller
- [ ] Interface → Planning (amend design)
- [ ] Implementation → Interface (modify API)
- [ ] Testing → Interface (fix test spec)
- [ ] Track invalidated items
- [ ] Prompt user about affected units
- [ ] Write transition tests

**Deliverable**: Can go back and modify earlier decisions

**Checkpoint**:
```bash
# During implementation:
# Select "Back to Interface"
# Confirm preserved units
# Modify types
# Return to implementation
# See which units need re-work
```

---

### Day 14: Integration Test (Real Feature)

**Goal**: Build an actual feature using the CLI

**Tasks**:
- [ ] Choose a simple feature to build
- [ ] Use CLI end-to-end
- [ ] Document issues encountered
- [ ] Fix any blocking bugs
- [ ] Iterate on prompts if needed

**Deliverable**: One complete feature built with Pear

**Suggested Test Feature**:
- Add a simple string utility module (3-4 functions)
- OR add a simple validation module
- Keep it small to complete in one day

---

### Day 15: Bug Fixes + Documentation

**Goal**: Stable, documented CLI ready for evaluation

**Tasks**:
- [ ] Fix bugs from Day 14 testing
- [ ] Refine prompts based on testing
- [ ] Ensure all tests pass
- [ ] Document any known issues
- [ ] Write evaluation notes
- [ ] Prepare for Phase 2 decision

**Deliverable**: Stable CLI + learnings document

---

## Milestones Summary

| Week | Day | Milestone |
|------|-----|-----------|
| 1 | 1 | Project compiles |
| 1 | 2 | Sessions persist |
| 1 | 3 | LLM responses stream |
| 1 | 5 | CLI accepts input |
| 2 | 6 | Planning phase works |
| 2 | 7 | Interface phase works |
| 2 | 9 | Implementation phase works |
| 2 | 10 | Tests run |
| 3 | 12 | Full workflow completes |
| 3 | 13 | Go-back works |
| 3 | 14 | Real feature built |
| 3 | 15 | Ready for evaluation |

---

## Daily Checkpoint Routine

End each day with:

1. **Commit code**: Working state committed to git
2. **Manual test**: Run through implemented features
3. **Update notes**: Document what worked/didn't
4. **Checkpoint save**: If state management is implemented
5. **Tomorrow's plan**: Know what you're building next

---

## Risk Checkpoints

### End of Week 1

**Check**: Is the foundation solid?
- [ ] State manager handles edge cases
- [ ] LLM responses are reliable
- [ ] CLI displays properly

**If issues**: Allocate Day 6 to fix foundation before proceeding.

### End of Week 2

**Check**: Is the core workflow usable?
- [ ] All phases can be completed
- [ ] Approval flow feels natural
- [ ] LLM output is usable

**If issues**: Use Week 3 buffer for iteration.

### End of Week 3

**Check**: Is Phase 1 ready for evaluation?
- [ ] Completed at least one real feature
- [ ] No blocking bugs
- [ ] Clear learnings documented

---

## Related Documents

- [00-overview.md](./00-overview.md) — Success criteria
- [11-testing-strategy.md](./11-testing-strategy.md) — What to test
- [13-exit-criteria.md](./13-exit-criteria.md) — Completion requirements

