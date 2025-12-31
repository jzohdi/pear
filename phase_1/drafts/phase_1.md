# Phase 1: CLI Prototype

**Duration**: 1-2 weeks  
**Goal**: Validate that the structured pair-programming workflow feels right before investing in the VS Code fork.

---

## Table of Contents

1. [Objectives](#objectives)
2. [Success Criteria](#success-criteria)
3. [Scope](#scope)
4. [Architecture](#architecture)
5. [Components](#components)
6. [Data Models](#data-models)
7. [LLM Prompts](#llm-prompts)
8. [User Interface](#user-interface)
9. [File Structure](#file-structure)
10. [Implementation Plan](#implementation-plan)
11. [Testing Strategy](#testing-strategy)
12. [Risk Mitigation](#risk-mitigation)
13. [Exit Criteria](#exit-criteria)

---

## Objectives

### Primary Objectives

1. **Validate the workflow**: Experience the four-phase flow (Planning → Interface → Implementation → Testing) on a real feature
2. **Test LLM prompts**: Refine the system prompts for each phase until they produce high-quality, consistent output
3. **Feel the feedback loops**: Determine if approving each unit feels productive or tedious
4. **Identify friction points**: Discover workflow issues before building the full editor

### Secondary Objectives

1. Build foundational code that can be extracted into the core library (Phase 2)
2. Create a reference implementation for how conversations flow
3. Document learnings about LLM behavior for each phase

---

## Success Criteria

| Criteria | Measurement |
|----------|-------------|
| Complete one real feature using the CLI | Feature works, tests pass |
| Workflow feels productive | Subjective: would you use this again? |
| No phase feels stuck | Each phase completes without manual workarounds |
| LLM output is usable | <20% of LLM output requires significant revision |
| Time is reasonable | Feature takes <2x time vs. manual coding |

---

## Scope

### In Scope

- Four-phase workflow (Planning, Interface, Implementation, Testing)
- Single LLM provider (Anthropic Claude)
- Single language (TypeScript)
- File creation and modification
- Basic session state persistence (JSON)
- Basic checkpoint/resume
- Human approval flow for each phase

### Out of Scope (Deferred to Later Phases)

- Multiple LLM providers
- Multiple language support
- Folder locking (single user for now)
- Project analysis/onboarding
- Advanced session recovery
- Git integration
- Any UI beyond terminal

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       CLI APPLICATION                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │   CLI Handler   │────▶│  Workflow       │                   │
│  │   (Inquirer)    │     │  Controller     │                   │
│  └─────────────────┘     └────────┬────────┘                   │
│                                   │                             │
│         ┌─────────────────────────┼─────────────────────────┐   │
│         │                         │                         │   │
│         ▼                         ▼                         ▼   │
│  ┌─────────────┐         ┌─────────────┐         ┌───────────┐ │
│  │  State      │         │  LLM        │         │  File     │ │
│  │  Manager    │         │  Client     │         │  Manager  │ │
│  └─────────────┘         └─────────────┘         └───────────┘ │
│         │                         │                         │   │
│         ▼                         ▼                         ▼   │
│  ┌─────────────┐         ┌─────────────┐         ┌───────────┐ │
│  │  Session    │         │  Anthropic  │         │  Project  │ │
│  │  Storage    │         │  API        │         │  Files    │ │
│  │  (JSON)     │         │             │         │           │ │
│  └─────────────┘         └─────────────┘         └───────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User Input (CLI)
      │
      ▼
┌─────────────┐
│  Workflow   │──────▶ Current Phase Logic
│  Controller │              │
└─────────────┘              │
      │                      ▼
      │              ┌─────────────┐
      │              │  LLM Client │
      │              └──────┬──────┘
      │                     │
      │                     ▼
      │              ┌─────────────┐
      │              │  Anthropic  │
      │              │  API        │
      │              └──────┬──────┘
      │                     │
      │◀────────────────────┘
      │
      ▼
┌─────────────┐
│  Display    │──────▶ Terminal Output
│  Response   │
└─────────────┘
      │
      ▼
┌─────────────┐
│  Prompt for │──────▶ [Approve] [Modify] [Go Back]
│  Action     │
└─────────────┘
      │
      ▼
State Update & Checkpoint
```

---

## Components

### 1. CLI Handler (`src/cli/index.ts`)

**Responsibility**: Handle terminal input/output, display formatted responses, prompt for user actions.

```typescript
interface CLIHandler {
  // Display methods
  displayPhaseHeader(phase: Phase): void;
  displayLLMResponse(response: string): void;
  displayProgress(state: WorkflowState): void;
  displayError(error: Error): void;
  
  // Input methods
  promptForFeatureDescription(): Promise<string>;
  promptForAction(actions: Action[]): Promise<Action>;
  promptForFreeformInput(prompt: string): Promise<string>;
  promptForConfirmation(message: string): Promise<boolean>;
  
  // Formatting
  formatCode(code: string, language: string): string;
  formatMarkdown(markdown: string): string;
}

type Action = 
  | { type: 'approve' }
  | { type: 'modify'; input: string }
  | { type: 'go_back' }
  | { type: 'quit' };
```

**Dependencies**: 
- `inquirer` for interactive prompts
- `chalk` for colored output
- `marked` + `marked-terminal` for markdown rendering
- `highlight.js` for code syntax highlighting

### 2. Workflow Controller (`src/workflow/controller.ts`)

**Responsibility**: Orchestrate the four-phase workflow, manage state transitions, enforce gates.

```typescript
interface WorkflowController {
  // Lifecycle
  startNewSession(featureDescription: string, featurePath: string): Promise<Session>;
  resumeSession(sessionId: string): Promise<Session>;
  
  // Phase execution
  runCurrentPhase(): Promise<PhaseResult>;
  
  // State transitions
  approveCurrentStep(): Promise<void>;
  requestChanges(feedback: string): Promise<void>;
  goBackToPhase(phase: Phase): Promise<void>;
  
  // Getters
  getCurrentState(): WorkflowState;
  getAvailableActions(): Action[];
}

interface PhaseResult {
  output: string;           // LLM response (formatted)
  artifacts?: Artifact[];   // Files to create/modify
  nextActions: Action[];    // Available user actions
}

interface Artifact {
  path: string;
  content: string;
  action: 'create' | 'modify' | 'delete';
}
```

**State Machine**:

```
                    ┌──────────────────────────────────────────┐
                    │                                          │
                    ▼                                          │
┌─────────┐    ┌─────────┐    ┌──────────────┐    ┌─────────┐ │
│ Planning│───▶│Interface│───▶│Implementation│───▶│ Testing │ │
└─────────┘    └─────────┘    └──────────────┘    └─────────┘ │
     │              │               │                   │      │
     │              │               │                   │      │
     │              │◀──────────────┘                   │      │
     │              │    (modify API)                   │      │
     │              │                                   │      │
     │              │◀──────────────────────────────────┘      │
     │              │         (test spec wrong)                │
     │                                                         │
     └─────────────────────────────────────────────────────────┘
                         (amend design)
```

### 3. State Manager (`src/state/manager.ts`)

**Responsibility**: Manage workflow state, persist to disk, handle checkpoints.

```typescript
interface StateManager {
  // CRUD
  createSession(initial: Partial<WorkflowState>): WorkflowState;
  getSession(sessionId: string): WorkflowState | null;
  updateSession(sessionId: string, updates: Partial<WorkflowState>): WorkflowState;
  
  // Persistence
  saveCheckpoint(state: WorkflowState): Promise<void>;
  loadCheckpoint(sessionId: string): Promise<WorkflowState | null>;
  
  // Query
  listIncompleteSessions(): Promise<SessionSummary[]>;
  
  // Conversation history
  addMessage(sessionId: string, message: Message): void;
  getMessages(sessionId: string, limit?: number): Message[];
}

interface Message {
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
  phase: Phase;
}

interface SessionSummary {
  sessionId: string;
  featureName: string;
  featurePath: string;
  currentPhase: Phase;
  lastActivity: Date;
}
```

**Storage Format**:

```
.pear-cli/
├── sessions/
│   ├── abc123.json      # Active session
│   └── def456.json      # Another session
└── config.json          # CLI configuration
```

### 4. LLM Client (`src/llm/client.ts`)

**Responsibility**: Interact with Anthropic API, construct prompts, stream responses.

```typescript
interface LLMClient {
  // Core method
  chat(options: ChatOptions): AsyncIterable<string>;
  
  // Convenience methods
  planFeature(description: string, context: ProjectContext): AsyncIterable<string>;
  designInterface(designDoc: string, context: ProjectContext): AsyncIterable<string>;
  implementUnit(unit: ImplementationUnit, context: ImplementationContext): AsyncIterable<string>;
  debugFailure(failure: TestFailure, context: ImplementationContext): AsyncIterable<string>;
}

interface ChatOptions {
  systemPrompt: string;
  messages: Message[];
  temperature?: number;
  maxTokens?: number;
}

interface ProjectContext {
  projectMap?: ProjectMap;
  existingTypes?: string;
  relevantFiles?: FileContent[];
}

interface ImplementationContext extends ProjectContext {
  designDocument: string;
  typeDefinitions: string;
  testCases: string;
  previousUnits: ImplementationUnit[];
}
```

**Streaming**: All LLM responses are streamed to provide immediate feedback. The CLI displays tokens as they arrive.

### 5. File Manager (`src/files/manager.ts`)

**Responsibility**: Read/write project files, manage artifacts.

```typescript
interface FileManager {
  // Read
  readFile(path: string): Promise<string | null>;
  readDirectory(path: string): Promise<string[]>;
  fileExists(path: string): Promise<boolean>;
  
  // Write
  writeFile(path: string, content: string): Promise<void>;
  createDirectory(path: string): Promise<void>;
  
  // Artifacts
  applyArtifacts(artifacts: Artifact[]): Promise<ApplyResult>;
  previewArtifacts(artifacts: Artifact[]): string;
  
  // Project detection
  detectProjectType(): Promise<ProjectType>;
  detectTestFramework(): Promise<TestFramework>;
}

interface ApplyResult {
  created: string[];
  modified: string[];
  failed: Array<{ path: string; error: string }>;
}
```

---

## Data Models

### WorkflowState

```typescript
interface WorkflowState {
  // Identity
  sessionId: string;
  featureName: string;
  featurePath: string;
  
  // Timing
  createdAt: Date;
  lastActivityAt: Date;
  
  // Current position
  currentPhase: Phase;
  phaseStatus: PhaseStatus;
  
  // Phase-specific state
  planning: PlanningState;
  interface: InterfaceState;
  implementation: ImplementationState;
  testing: TestingState;
  
  // Conversation
  messages: Message[];
}

type Phase = 'planning' | 'interface' | 'implementation' | 'testing' | 'complete';
type PhaseStatus = 'pending' | 'in_progress' | 'awaiting_approval' | 'approved';

interface PlanningState {
  designDocContent: string | null;
  designDocPath: string | null;
  approved: boolean;
}

interface InterfaceState {
  typesContent: string | null;
  typesPath: string | null;
  testsContent: string | null;
  testsPath: string | null;
  approved: boolean;
}

interface ImplementationState {
  units: ImplementationUnit[];
  currentUnitIndex: number;
  dependencyOrder: string[];
}

interface ImplementationUnit {
  id: string;
  name: string;
  description: string;
  filePath: string;
  status: 'pending' | 'in_progress' | 'awaiting_review' | 'approved';
  implementation: string | null;
  dependencies: string[];  // IDs of units this depends on
}

interface TestingState {
  lastTestRun: TestRunResult | null;
  debugIterations: number;
  manualTestsComplete: boolean;
}

interface TestRunResult {
  passed: number;
  failed: number;
  failures: TestFailure[];
  timestamp: Date;
}

interface TestFailure {
  testName: string;
  expected: string;
  received: string;
  errorMessage: string;
}
```

---

## LLM Prompts

### Planning Phase System Prompt

```typescript
const PLANNING_SYSTEM_PROMPT = `You are Pear, an AI pair-programming assistant helping design a software feature.

Your role in this phase:
1. Ask clarifying questions to understand the feature requirements
2. Produce a structured design document

When asking questions:
- Ask 3-5 focused questions maximum
- Questions should clarify scope, constraints, and technical approach
- Don't ask obvious questions the developer would know

When producing the design document, use this structure:

# Feature: [Feature Name]

## Problem Statement
[1-2 sentences describing what problem this solves]

## Proposed Solution
[1-2 sentences describing the high-level approach]

## Technical Approach
[Numbered list of implementation steps]

## Scope
[Bullet list of what's included]

## Out of Scope
[Bullet list of what's explicitly not included]

Be concise. The design document should be <500 words.`;
```

### Interface Phase System Prompt

```typescript
const INTERFACE_SYSTEM_PROMPT = `You are Pear, an AI pair-programming assistant defining interfaces and tests.

Your role in this phase:
1. Define TypeScript types/interfaces based on the approved design
2. Translate human-described behaviors into test cases
3. Suggest additional test cases with clear rationale

Guidelines:
- Types should be minimal but complete
- Function signatures should match the design document
- Test cases should be behavior-focused, not implementation-focused
- Explain why you're suggesting additional tests

Output format for types:
\`\`\`typescript
// src/features/{feature}/types.ts
[type definitions]
\`\`\`

Output format for tests:
\`\`\`typescript
// src/features/{feature}/__tests__/{feature}.test.ts
describe('[FeatureName]', () => {
  // Test skeletons with descriptive names
});
\`\`\`

When suggesting additional tests, format as:
- **[Test name]**: [Why this test is valuable]`;
```

### Implementation Phase System Prompt

```typescript
const IMPLEMENTATION_SYSTEM_PROMPT = `You are Pear, an AI pair-programming assistant implementing code.

Your role in this phase:
1. Implement the specific unit you're asked to implement
2. Follow the type definitions exactly
3. Explain your implementation choices briefly

Guidelines:
- Implement ONLY the unit requested, nothing more
- Use the types defined in Phase 2
- If you need a helper function, note it as a dependency
- Keep implementations focused and readable

Output format:
\`\`\`typescript
// Implementation
[code]
\`\`\`

**Implementation notes:**
- [Brief explanation of key decisions]
- [Any assumptions made]
- [Dependencies on other units]

If anything is unclear about the requirements, ASK before implementing.`;
```

### Testing Phase System Prompt

```typescript
const TESTING_SYSTEM_PROMPT = `You are Pear, an AI pair-programming assistant verifying implementations.

Your role in this phase:
1. Analyze test failures and propose fixes
2. Generate manual test checklists
3. Document test evidence

When analyzing failures:
- Identify the root cause
- Propose a minimal fix
- Explain why the fix works

Output format for fixes:
**Issue**: [Description of the problem]
**Root Cause**: [Why this is happening]
**Proposed Fix**:
\`\`\`typescript
[code change]
\`\`\`
**Explanation**: [Why this fixes it]

For manual tests, format as:
## Manual Test Checklist
- [ ] [Test step 1]
- [ ] [Test step 2]
...`;
```

---

## User Interface

### Terminal Layout

```
╔══════════════════════════════════════════════════════════════════╗
║  🍐 Pear CLI - Feature: User Authentication                      ║
║  Phase: 2/4 - Interface & Tests                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  [Pear]: Based on the design document, here are the proposed     ║
║          type definitions:                                       ║
║                                                                  ║
║  ┌────────────────────────────────────────────────────────────┐  ║
║  │ // src/features/auth/types.ts                              │  ║
║  │                                                            │  ║
║  │ export interface AuthConfig {                              │  ║
║  │   provider: 'google';                                      │  ║
║  │   clientId: string;                                        │  ║
║  │   ...                                                      │  ║
║  │ }                                                          │  ║
║  └────────────────────────────────────────────────────────────┘  ║
║                                                                  ║
║  Would you like to approve these types or request changes?       ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║  [A] Approve   [M] Modify   [B] Back to Planning   [Q] Quit      ║
╚══════════════════════════════════════════════════════════════════╝
```

### Key Interactions

**Starting a new feature:**
```
$ pear new

🍐 Pear CLI

? What feature would you like to build?
> Add Google OAuth authentication to protect certain routes

? Where should this feature live?
> src/features/auth

Starting new session...
Session ID: abc123

═══════════════════════════════════════════════════════════════
Phase 1/4: Planning
═══════════════════════════════════════════════════════════════

[Pear]: I'll help you design Google OAuth authentication. 
        Let me ask a few clarifying questions:

        1. Should this protect all routes or specific ones?
        2. How long should sessions last?
        3. Do you have a database for user records?

? Your response:
> 
```

**Approving implementation:**
```
═══════════════════════════════════════════════════════════════
Phase 3/4: Implementation (2/5)
═══════════════════════════════════════════════════════════════

Current unit: handleCallback()

[Pear]: Here's the implementation for handleCallback():

        ┌──────────────────────────────────────────────────────┐
        │ async function handleCallback(code: string) {        │
        │   const tokens = await exchangeCodeForTokens(code);  │
        │   const userInfo = await fetchUserInfo(tokens);      │
        │   const session = await createSession(userInfo);     │
        │   return session;                                    │
        │ }                                                    │
        └──────────────────────────────────────────────────────┘
        
        Implementation notes:
        - Uses the createSession() unit we implemented earlier
        - Handles the OAuth callback flow as specified
        - Returns session object matching the type definition

? Action: (Use arrow keys)
❯ Approve - Save and continue to next unit
  Modify - Request changes
  Back - Return to previous phase
  Quit - Save and exit
```

---

## File Structure

### CLI Application Structure

```
pear-cli/
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── .env.example                 # ANTHROPIC_API_KEY
│
├── src/
│   ├── index.ts                 # Entry point
│   │
│   ├── cli/
│   │   ├── index.ts             # CLI handler
│   │   ├── display.ts           # Formatting and display
│   │   └── prompts.ts           # User prompts
│   │
│   ├── workflow/
│   │   ├── controller.ts        # Main workflow orchestration
│   │   ├── phases/
│   │   │   ├── planning.ts      # Planning phase logic
│   │   │   ├── interface.ts     # Interface phase logic
│   │   │   ├── implementation.ts# Implementation phase logic
│   │   │   └── testing.ts       # Testing phase logic
│   │   └── transitions.ts       # Phase transition logic
│   │
│   ├── state/
│   │   ├── manager.ts           # State management
│   │   ├── types.ts             # State type definitions
│   │   └── storage.ts           # JSON persistence
│   │
│   ├── llm/
│   │   ├── client.ts            # LLM client wrapper
│   │   ├── prompts.ts           # System prompts
│   │   └── parser.ts            # Response parsing
│   │
│   ├── files/
│   │   ├── manager.ts           # File operations
│   │   └── artifacts.ts         # Artifact handling
│   │
│   └── utils/
│       ├── logger.ts            # Logging
│       └── errors.ts            # Error types
│
├── tests/
│   ├── workflow/
│   │   └── controller.test.ts
│   ├── state/
│   │   └── manager.test.ts
│   └── llm/
│       └── prompts.test.ts      # Prompt output validation
│
└── .pear-cli/                   # Created at runtime
    ├── config.json
    └── sessions/
```

### Generated Project Files

When using the CLI on a project, these files are created:

```
target-project/
├── .pear/
│   ├── config.yaml              # Created on first run
│   └── sessions/
│       └── {session-id}.yaml    # Session state
│
└── src/features/{feature}/
    ├── DESIGN.md                # Phase 1 output
    ├── types.ts                 # Phase 2 output
    ├── index.ts                 # Phase 3 output
    └── __tests__/
        └── {feature}.test.ts    # Phase 2-3 output
```

---

## Implementation Plan

### Week 1: Foundation

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | Project setup, dependencies | Working TypeScript project |
| 2 | State manager + storage | Sessions persist to JSON |
| 3 | LLM client + Anthropic integration | Can send/receive messages |
| 4 | CLI handler basics | Interactive prompts work |
| 5 | Planning phase | Can complete Phase 1 end-to-end |

### Week 2: Complete Workflow

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | Interface phase | Types and tests generation |
| 2 | Implementation phase (part 1) | Unit-by-unit implementation |
| 3 | Implementation phase (part 2) | Dependency ordering, back navigation |
| 4 | Testing phase | Test running, debug loop |
| 5 | Integration testing | Complete one real feature |

### Daily Checkpoints

Each day should end with:
1. Working code committed
2. Manual test of the day's feature
3. Notes on what worked/didn't work
4. Updated prompts if needed

---

## Testing Strategy

### Unit Tests

```typescript
// tests/state/manager.test.ts
describe('StateManager', () => {
  it('should create new session with initial state', () => {});
  it('should persist session to JSON', () => {});
  it('should load session from JSON', () => {});
  it('should list incomplete sessions', () => {});
});

// tests/workflow/controller.test.ts
describe('WorkflowController', () => {
  it('should start in planning phase', () => {});
  it('should transition to interface after planning approval', () => {});
  it('should not allow skipping phases', () => {});
  it('should allow going back to previous phase', () => {});
});
```

### Integration Tests

```typescript
// tests/integration/full-workflow.test.ts
describe('Full Workflow', () => {
  it('should complete all phases for a simple feature', async () => {
    // Mock LLM responses
    // Run through all phases
    // Verify artifacts created
  });
});
```

### Manual Testing

Use the CLI to build a real feature:

**Test Feature**: Add a simple utility function to an existing project
1. Start new session
2. Complete planning phase
3. Complete interface phase
4. Implement each unit
5. Run tests
6. Verify files created correctly

---

## Risk Mitigation

### Risk 1: LLM Output Quality

**Risk**: LLM produces inconsistent or incorrect code

**Mitigation**:
- Iterate on prompts until output is reliable
- Add examples to system prompts if needed
- Keep temperature low (0.2-0.3) for code generation
- Document prompt versions and their effectiveness

### Risk 2: Workflow Feels Tedious

**Risk**: Approving every unit feels like too much friction

**Mitigation**:
- Track time per approval
- If consistently <10 seconds, consider batching
- Note where tedium occurs for Phase 2 improvements
- Consider "approve all remaining" option

### Risk 3: State Corruption

**Risk**: Crashes or bugs corrupt session state

**Mitigation**:
- Checkpoint after every state change
- Validate state on load
- Keep backup of previous checkpoint
- Clear error messages for recovery

### Risk 4: Context Window Limits

**Risk**: Conversation gets too long for LLM

**Mitigation**:
- Implement sliding window for messages
- Summarize older phases
- Keep design doc always in context
- Monitor token usage

---

## Exit Criteria

Phase 1 is complete when:

1. **Functional**: Can complete all four phases for a real feature
2. **Stable**: No crashes during normal operation
3. **Usable**: A developer (you) would use this again
4. **Documented**: Learnings captured for Phase 2

### Deliverables

- [ ] Working CLI application
- [ ] At least one feature built using the CLI
- [ ] Refined LLM prompts for each phase
- [ ] List of friction points and improvement ideas
- [ ] Decision on whether to proceed to Phase 2

### Questions to Answer

After completing Phase 1:

1. Does the four-phase workflow make sense?
2. Is approving each unit the right granularity?
3. Which prompts need the most improvement?
4. What was surprising about LLM behavior?
5. What should change before building the full editor?

---

## Appendix: Dependencies

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.24.0",
    "chalk": "^5.3.0",
    "inquirer": "^9.2.0",
    "marked": "^12.0.0",
    "marked-terminal": "^7.0.0",
    "yaml": "^2.4.0",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "@types/inquirer": "^9.0.0",
    "@types/node": "^20.0.0",
    "@types/uuid": "^9.0.0",
    "typescript": "^5.4.0",
    "vitest": "^1.4.0",
    "tsx": "^4.7.0"
  }
}
```

