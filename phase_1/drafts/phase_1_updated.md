# Phase 1: CLI Prototype

**Duration**: 2-3 weeks  
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
8. [Response Parsing](#response-parsing)
9. [Context Management](#context-management)
10. [Phase Transitions](#phase-transitions)
11. [User Interface](#user-interface)
12. [File Structure](#file-structure)
13. [Implementation Plan](#implementation-plan)
14. [Testing Strategy](#testing-strategy)
15. [Error Handling](#error-handling)
16. [Risk Mitigation](#risk-mitigation)
17. [Exit Criteria](#exit-criteria)

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
- Completion handling (session archival, summary)
- Single LLM provider (Anthropic Claude)
- Single language (TypeScript)
- File creation and modification
- Session state persistence (JSON)
- Checkpoint/resume with crash recovery
- Human approval flow for each phase
- Go-back transitions with state preservation
- Test implementation (both production code and tests)
- Manual test recording
- TEST_EVIDENCE.md generation

### Out of Scope (Deferred to Later Phases)

- Multiple LLM providers
- Multiple language support
- Folder locking (single user for now)
- Project analysis/onboarding
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
│         │                 ┌───────┴───────┐                 │   │
│         │                 │               │                 │   │
│         │                 ▼               ▼                 │   │
│         │         ┌─────────────┐ ┌─────────────┐           │   │
│         │         │  Response   │ │  Context    │           │   │
│         │         │  Parser     │ │  Manager    │           │   │
│         │         └─────────────┘ └─────────────┘           │   │
│         │                                                   │   │
│         │                 ┌─────────────┐                   │   │
│         │                 │  Test       │                   │   │
│         │                 │  Runner     │                   │   │
│         │                 └─────────────┘                   │   │
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
      │              │  Context    │
      │              │  Manager    │
      │              └──────┬──────┘
      │                     │
      │                     ▼
      │              ┌─────────────┐
      │              │  LLM Client │
      │              └──────┬──────┘
      │                     │
      │                     ▼
      │              ┌─────────────┐
      │              │  Response   │
      │              │  Parser     │
      │              └──────┬──────┘
      │                     │
      │◀────────────────────┘
      │
      ▼
┌─────────────┐
│  Display    │──────▶ Terminal Output (Streaming)
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
  displayStreamingResponse(stream: AsyncIterable<string>): Promise<string>;
  displayProgress(state: WorkflowState): void;
  displayError(error: PearError): void;
  displayCompletionSummary(state: WorkflowState): void;
  
  // Input methods
  promptForFeatureDescription(): Promise<string>;
  promptForFeaturePath(): Promise<string>;
  promptForAction(actions: Action[]): Promise<Action>;
  promptForFreeformInput(prompt: string): Promise<string>;
  promptForConfirmation(message: string): Promise<boolean>;
  promptForManualTestResult(test: ManualTest): Promise<ManualTestResult>;
  
  // Formatting
  formatCode(code: string, language: string): string;
  formatMarkdown(markdown: string): string;
  formatDiff(oldContent: string, newContent: string): string;
}

type Action = 
  | { type: 'approve' }
  | { type: 'modify'; input: string }
  | { type: 'go_back'; targetPhase: Phase }
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
  goBackToPhase(phase: Phase): Promise<GoBackResult>;
  completeSession(): Promise<CompletionResult>;
  
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

interface GoBackResult {
  previousPhase: Phase;
  targetPhase: Phase;
  preservedState: Partial<WorkflowState>;
  invalidatedItems: InvalidatedItem[];
  userActionRequired: boolean;
  userPrompt?: string;
}

interface InvalidatedItem {
  type: 'unit' | 'test' | 'type';
  id: string;
  name: string;
  reason: string;
}

interface CompletionResult {
  success: boolean;
  summary: string;
  filesCreated: string[];
  filesModified: string[];
  testsPassed: number;
  sessionArchivePath: string;
}
```

**State Machine**:

```
                    ┌──────────────────────────────────────────┐
                    │                                          │
                    ▼                                          │
┌─────────┐    ┌─────────┐    ┌──────────────┐    ┌─────────┐ │
│ Planning│───▶│Interface│───▶│Implementation│───▶│ Testing │─┼──▶ Complete
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
  restoreFromBackup(sessionId: string): Promise<WorkflowState | null>;
  
  // Query
  listIncompleteSessions(): Promise<SessionSummary[]>;
  
  // Archival
  archiveSession(sessionId: string): Promise<string>;  // Returns archive path
  
  // Conversation history
  addMessage(sessionId: string, message: Message): void;
  getMessages(sessionId: string, limit?: number): Message[];
}

interface Message {
  role: 'user' | 'assistant' | 'system';
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
  status: 'active' | 'interrupted';
}
```

**Storage Format**:

```
.pear/
├── sessions/
│   ├── abc123.json           # Active session
│   ├── abc123.backup.json    # Previous checkpoint (for recovery)
│   └── done/                 # Archived sessions
│       └── def456.json
└── config.json               # CLI configuration
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
  analyzeeDependencies(types: string): AsyncIterable<string>;
  implementUnit(unit: ImplementationUnit, context: ImplementationContext): AsyncIterable<string>;
  implementTest(test: TestImplementation, context: ImplementationContext): AsyncIterable<string>;
  debugFailure(failure: TestFailure, context: ImplementationContext): AsyncIterable<string>;
  generateTestEvidence(state: WorkflowState): AsyncIterable<string>;
}

interface ChatOptions {
  systemPrompt: string;
  messages: Message[];
  temperature?: number;  // Defaults by phase (see PHASE_TEMPERATURES)
  maxTokens?: number;    // Defaults by phase (see PHASE_MAX_TOKENS)
}

// Default temperatures by phase
const PHASE_TEMPERATURES: Record<Phase, number> = {
  planning: 0.7,       // More creative for brainstorming
  interface: 0.3,      // Precise for type definitions
  implementation: 0.2, // Very precise for code generation
  testing: 0.3,        // Precise for debugging
  complete: 0.3,       // Precise for summaries
};

// Default max tokens by phase
const PHASE_MAX_TOKENS: Record<Phase, number> = {
  planning: 2000,
  interface: 3000,
  implementation: 1500,
  testing: 1500,
  complete: 1000,
};

interface ProjectContext {
  designDocument?: string;
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

### 6. Test Runner (`src/testing/runner.ts`)

**Responsibility**: Execute tests, parse results, provide structured output for Phase 4.

```typescript
interface TestRunner {
  // Detection
  detectTestFramework(projectPath: string): Promise<TestFramework>;
  
  // Execution
  runTests(options: TestRunOptions): Promise<TestRunResult>;
  runSingleTest(testPath: string, testName: string): Promise<TestResult>;
  
  // Parsing
  parseTestOutput(output: string, framework: TestFramework): TestRunResult;
}

interface TestRunOptions {
  projectPath: string;
  testPattern?: string;      // e.g., "**/*.test.ts"
  specificTests?: string[];  // Run only these tests
  timeout?: number;          // milliseconds
}

type TestFramework = 'vitest' | 'jest' | 'mocha' | 'unknown';

interface TestResult {
  name: string;
  fullName: string;          // Including describe block
  status: 'passed' | 'failed' | 'skipped';
  duration: number;          // milliseconds
  error?: {
    message: string;
    expected?: string;
    received?: string;
    stack?: string;
  };
}
```

**Implementation Notes**:
- For Phase 1, we only support Vitest (detected via `vitest.config.ts`)
- Test execution uses `child_process.spawn` to run `npx vitest run --reporter=json`
- JSON reporter provides structured output for parsing
- Future phases will add support for Jest, Mocha, etc.

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

// Unified status type for all units
type UnitStatus = 'pending' | 'in_progress' | 'awaiting_approval' | 'approved' | 'needs_review';
```

### Phase-Specific State

```typescript
interface PlanningState {
  designDocContent: string | null;
  designDocPath: string | null;  // e.g., "src/features/auth/DESIGN.md"
  approved: boolean;
}

interface InterfaceState {
  typesContent: string | null;
  typesPath: string | null;
  testsContent: string | null;
  testsPath: string | null;
  approved: boolean;
  
  // Dependency analysis for implementation order
  dependencyAnalysis: DependencyAnalysis | null;
}

interface DependencyAnalysis {
  graph: DependencyNode[];
  suggestedOrder: string[];  // Function names in implementation order
  orderApproved: boolean;
}

interface DependencyNode {
  functionName: string;
  dependsOn: string[];      // Other functions this calls
  dependedOnBy: string[];   // Functions that call this
}

interface ImplementationState {
  // Production code units
  units: ImplementationUnit[];
  currentUnitIndex: number;
  dependencyOrder: string[];  // Unit IDs in implementation order
  
  // Test implementations
  testImplementations: TestImplementation[];
  currentTestIndex: number;
  
  // Track which part we're working on
  stage: 'production' | 'tests';
}

interface ImplementationUnit {
  id: string;
  name: string;
  description: string;
  filePath: string;
  status: UnitStatus;
  implementation: string | null;
  dependencies: string[];  // IDs of units this depends on
}

interface TestImplementation {
  id: string;
  testName: string;         // e.g., "should create session from valid OAuth code"
  testFilePath: string;
  status: UnitStatus;
  implementation: string | null;
  relatedUnitIds: string[]; // Which production units this tests
}
```

### Testing State

```typescript
interface TestingState {
  // Automated test tracking
  testRuns: TestRunResult[];       // History of all test runs
  lastTestRun: TestRunResult | null;
  debugIterations: number;
  
  // Manual test tracking
  manualTests: ManualTest[];
  manualTestsComplete: boolean;
  
  // Evidence document
  testEvidencePath: string | null;
  testEvidenceGenerated: boolean;
}

interface TestRunResult {
  passed: number;
  failed: number;
  skipped: number;
  duration: number;
  failures: TestFailure[];
  timestamp: Date;
}

interface TestFailure {
  testName: string;
  expected: string;
  received: string;
  errorMessage: string;
  filePath?: string;
  lineNumber?: number;
}

interface ManualTest {
  id: string;
  description: string;
  expectedResult: string;
  status: 'pending' | 'passed' | 'failed' | 'skipped';
  verifiedBy: string | null;
  verifiedAt: Date | null;
  notes: string | null;
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

Output the design document in a code block with the file path:
\`\`\`markdown
// src/features/{feature}/DESIGN.md
[document content]
\`\`\`

Be concise. The design document should be <500 words.`;
```

### Interface Phase System Prompt

```typescript
const INTERFACE_SYSTEM_PROMPT = `You are Pear, an AI pair-programming assistant defining interfaces and tests.

Your role in this phase:
1. Define TypeScript types/interfaces based on the approved design
2. Analyze dependencies between functions to determine implementation order
3. Translate human-described behaviors into test cases
4. Suggest additional test cases with clear rationale

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

Output format for tests (skeletons only - implementations come in Phase 3):
\`\`\`typescript
// src/features/{feature}/__tests__/{feature}.test.ts
describe('[FeatureName]', () => {
  describe('[functionName]', () => {
    it('should [behavior]', async () => {
      // Implementation pending Phase 3
    });
  });
});
\`\`\`

After types are approved, provide dependency analysis:
## Implementation Order

Based on the type definitions, here's the suggested implementation order:

### Dependency Graph
[ASCII diagram showing which functions depend on which]

### Suggested Order
1. **[functionName]** - [reason this is first]
2. **[functionName]** - [dependencies]
...

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

You will be asked to implement TWO types of units:
- **Production units**: The actual feature code
- **Test implementations**: Fill in the test skeletons from Phase 2

Guidelines for production code:
- Implement ONLY the unit requested, nothing more
- Use the types defined in Phase 2
- If you need a helper function, note it as a dependency
- Keep implementations focused and readable

Guidelines for test implementations:
- Use the test skeleton from Phase 2 as your starting point
- Write assertions that verify the behavior described in the test name
- Use appropriate mocking for external dependencies
- Keep tests focused on one behavior each
- Follow Vitest patterns (describe, it, expect)

Output format for production code:
\`\`\`typescript
// src/features/{feature}/{filename}.ts
[code]
\`\`\`

Output format for test implementations:
\`\`\`typescript
// src/features/{feature}/__tests__/{feature}.test.ts
// Test: {test name}

it('should [behavior]', async () => {
  // Arrange
  [setup code]
  
  // Act
  [execution code]
  
  // Assert
  [assertions]
});
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
3. Generate the TEST_EVIDENCE.md document when all tests pass

When analyzing failures:
- Identify the root cause
- Propose a minimal fix
- Explain why the fix works

Output format for fixes:
**Issue**: [Description of the problem]
**Root Cause**: [Why this is happening]
**Proposed Fix**:
\`\`\`typescript
// {filepath}
[code change]
\`\`\`
**Explanation**: [Why this fixes it]

For manual tests, generate a checklist based on the feature's user-facing behaviors:
## Manual Test Checklist
- [ ] [Concrete action user takes] → [Expected result]
- [ ] [Another action] → [Expected result]
...

When all tests pass and manual verification is complete, generate TEST_EVIDENCE.md:
\`\`\`markdown
// src/features/{feature}/TEST_EVIDENCE.md

# Test Evidence: {Feature Name}

## Session Information
- **Session ID**: {sessionId}
- **Completed**: {date}

## Automated Tests

### Unit Tests
- **Run Date**: {timestamp}
- **Framework**: Vitest
- **Result**: {passed}/{total} passed
- **Duration**: {duration}ms

| Test Name | Status | Duration |
|-----------|--------|----------|
| {test name} | ✓ Pass | {ms}ms |

## Manual Tests

| Test Case | Status | Verified By | Date | Notes |
|-----------|--------|-------------|------|-------|
| {description} | ✓ Pass | {name} | {date} | {notes} |

## Sign-off
Feature verified and ready for merge.
\`\`\``;
```

---

## Response Parsing

### Response Parser (`src/llm/parser.ts`)

**Responsibility**: Extract structured data from LLM responses.

```typescript
interface ResponseParser {
  // Extract code blocks from markdown response
  parseCodeBlocks(response: string): ParsedCodeBlock[];
  
  // Extract artifacts (files to create/modify)
  parseArtifacts(response: string): Artifact[];
  
  // Extract structured sections
  parseSections(response: string): ParsedSection[];
  
  // Validate extracted content
  validateTypeScript(code: string): ValidationResult;
}

interface ParsedCodeBlock {
  language: string;         // 'typescript', 'markdown', etc.
  filePath: string | null;  // Extracted from comment
  content: string;          // The code itself
  startLine: number;        // Position in original response
  endLine: number;
}

interface ParsedSection {
  heading: string;          // e.g., "Implementation notes"
  level: number;            // 1 = #, 2 = ##, etc.
  content: string;
}

interface ValidationResult {
  valid: boolean;
  errors: string[];
}
```

**Parsing Rules**:

1. **Code Block Extraction**:
   - Match triple backticks with optional language tag
   - Extract file path from first line comment: `// path/to/file.ts`
   - Handle nested code blocks (escape sequences)

2. **File Path Patterns**:
   ```typescript
   const FILE_PATH_PATTERNS = [
     /^\/\/ (.+\.(ts|js|tsx|jsx|md))$/m,           // // src/file.ts
     /^\/\/ File: (.+)$/m,                          // // File: src/file.ts
     /^# (.+\.(md))$/m,                             // # DESIGN.md (markdown)
   ];
   ```

3. **Artifact Creation**:
   ```typescript
   function parseArtifacts(response: string): Artifact[] {
     const codeBlocks = parseCodeBlocks(response);
     
     return codeBlocks
       .filter(block => block.filePath !== null)
       .map(block => ({
         path: block.filePath!,
         content: block.content,
         action: 'create' as const,
       }));
   }
   ```

4. **Handling Ambiguity**:
   - If no file path in comment, ask user where to save
   - If multiple blocks for same file, concatenate in order
   - If language tag missing, infer from file extension

---

## Context Management

### Context Manager (`src/llm/context.ts`)

**Responsibility**: Build appropriate context for each LLM request, managing token limits.

```typescript
interface ContextManager {
  // Build context for a specific phase
  buildContext(state: WorkflowState, phase: Phase): ContextWindow;
  
  // Estimate tokens (rough approximation)
  estimateTokens(text: string): number;
  
  // Trim context to fit within limit
  trimToLimit(context: ContextWindow, maxTokens: number): ContextWindow;
}

interface ContextWindow {
  // Always included (high priority)
  systemPrompt: string;
  designDocument: string;         // Source of truth, always present
  
  // Phase-specific (medium priority)
  typeDefinitions?: string;       // From Phase 2 onwards
  relevantTests?: string;         // Current tests being worked on
  currentUnit?: string;           // Current implementation unit
  
  // Recent history (lower priority, trimmed first)
  recentMessages: Message[];      // Sliding window
  previousUnits?: string[];       // Last 2-3 implemented units
  
  // Metadata
  estimatedTokens: number;
}
```

**Token Estimation**:
```typescript
// Rough approximation: ~4 chars per token for English/code
const CHARS_PER_TOKEN = 4;
const MAX_CONTEXT_TOKENS = 100000;  // Claude Sonnet context limit
const RESERVED_FOR_RESPONSE = 4000; // Leave room for response

function estimateTokens(text: string): number {
  return Math.ceil(text.length / CHARS_PER_TOKEN);
}
```

**Context Building Strategy by Phase**:

| Phase | Always Include | Include if Space | Trim First |
|-------|---------------|------------------|------------|
| Planning | System prompt, Feature description | N/A | Older messages |
| Interface | System prompt, Design doc | Existing project types | Older messages |
| Implementation | System prompt, Design doc, Types, Current unit | Previous 2-3 units | Older messages |
| Testing | System prompt, Design doc, Types, Failed test | Implementation code | Older messages |

**Message Sliding Window**:

```typescript
function buildMessageWindow(
  allMessages: Message[], 
  maxTokens: number
): Message[] {
  const result: Message[] = [];
  let tokenCount = 0;
  
  // Always include first message (feature description)
  if (allMessages.length > 0) {
    result.push(allMessages[0]);
    tokenCount += estimateTokens(allMessages[0].content);
  }
  
  // Add recent messages from the end
  for (let i = allMessages.length - 1; i > 0; i--) {
    const msgTokens = estimateTokens(allMessages[i].content);
    if (tokenCount + msgTokens > maxTokens) break;
    
    result.splice(1, 0, allMessages[i]);
    tokenCount += msgTokens;
  }
  
  // Add summary marker if we skipped messages
  if (result.length < allMessages.length) {
    const skipped = allMessages.length - result.length;
    result.splice(1, 0, {
      role: 'system' as const,
      content: `[${skipped} earlier messages omitted]`,
      timestamp: new Date(),
      phase: result[1]?.phase || 'planning'
    });
  }
  
  return result;
}
```

---

## Phase Transitions

### Going Forward

Forward transitions are straightforward:

| From | To | Trigger | Action |
|------|-----|---------|--------|
| Planning | Interface | Design doc approved | Save DESIGN.md |
| Interface | Implementation | Types, tests, order approved | Save types.ts, test skeletons |
| Implementation | Testing | All units approved | Run initial tests |
| Testing | Complete | All tests pass, manual done | Generate TEST_EVIDENCE.md |

### Going Backward

Going backward requires careful state management to preserve work.

**Interface → Planning (Amend Design)**

```typescript
interface AmendDesignTransition {
  preservedState: {
    interfaceState: InterfaceState;
    implementationState: ImplementationState;
  };
  changes: {
    currentPhase: 'planning';
    phaseStatus: 'in_progress';
  };
}
```

Flow:
1. User requests "amend design" from Interface phase
2. System saves current Interface state
3. Design document reopened for modification
4. After re-approval, return to Interface phase
5. System asks: "Design changed. Review types/tests for compatibility?"

**Implementation → Interface (Modify API)**

```typescript
interface ModifyAPITransition {
  preservedState: {
    implementationUnits: ImplementationUnit[];
  };
  affectedUnits: string[];  // Units that may need reimplementation
}
```

Affected unit determination:
1. Parse old and new type definitions
2. Compare function signatures
3. Units using changed types marked as `'needs_review'`
4. Human confirms which units to reimplement

**Testing → Interface (Test Spec Wrong)**

```typescript
interface FixTestSpecTransition {
  currentPhase: 'interface';
  testToModify: string;
  preserveImplementation: true;
}
```

Flow:
1. Test fails, human determines test is wrong (not implementation)
2. Human selects "Fix test specification"
3. Return to Interface phase, modify test
4. Return to Testing phase, rerun

### State Preservation Rules

| Transition | Preserved | Potentially Invalidated |
|------------|-----------|------------------------|
| Interface → Planning | Interface state, Implementation state | Nothing |
| Implementation → Interface | Approved units | Units using changed types |
| Testing → Interface | All implementation | Modified test only |
| Testing → Implementation | Unaffected units | Unit being debugged |

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

**Implementation phase progress:**
```
═══════════════════════════════════════════════════════════════
Phase 3/4: Implementation
═══════════════════════════════════════════════════════════════

Progress: Production Code (2/5)
☑ getOAuthUrl()        [Approved]
☑ createSession()      [Approved]
☐ handleCallback()     [In Progress]  ← Current
☐ validateSession()    [Pending]
☐ logout()             [Pending]

Progress: Test Implementations (0/8)
☐ All tests pending implementation after production code

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
        - Uses createSession() we implemented earlier
        - Handles the OAuth callback flow as specified

? Action: (Use arrow keys)
❯ Approve - Save and continue to next unit
  Modify - Request changes
  Back - Return to Interface phase
  Quit - Save and exit
```

**Manual test recording:**
```
═══════════════════════════════════════════════════════════════
Phase 4/4: Testing - Manual Verification
═══════════════════════════════════════════════════════════════

Automated tests: ✓ 12/12 passed

Manual test checklist:

  1. [ ] Navigate to /login, click "Sign in with Google"
         → Should redirect to Google OAuth page
  
  2. [ ] Complete Google sign-in with valid credentials
         → Should redirect back with session created
  
  3. [ ] Access protected route (/dashboard)
         → Should display dashboard content

? Select a test to record: (Use arrow keys)
❯ 1. Navigate to /login... [pending]
  2. Complete Google sign-in... [pending]
  3. Access protected route... [pending]
  ────────────────────────────────────
  Mark all as passed
  Skip manual tests
```

**Completion:**
```
═══════════════════════════════════════════════════════════════
🎉 Feature Complete: User Authentication
═══════════════════════════════════════════════════════════════

Summary:
├── Design: src/features/auth/DESIGN.md
├── Types: src/features/auth/types.ts  
├── Implementation: 5 production units, 8 tests
├── Tests: 12 automated (all passing), 3 manual (all passing)
└── Evidence: src/features/auth/TEST_EVIDENCE.md

Files created:
  ✓ src/features/auth/DESIGN.md
  ✓ src/features/auth/types.ts
  ✓ src/features/auth/index.ts
  ✓ src/features/auth/__tests__/auth.test.ts
  ✓ src/features/auth/TEST_EVIDENCE.md

Session archived: .pear/sessions/done/abc123.json

Tip: Don't forget to commit these changes to git!
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
│   │   ├── prompts.ts           # User prompts
│   │   └── streaming.ts         # Stream output handling
│   │
│   ├── workflow/
│   │   ├── controller.ts        # Main workflow orchestration
│   │   ├── phases/
│   │   │   ├── planning.ts      # Planning phase logic
│   │   │   ├── interface.ts     # Interface phase logic
│   │   │   ├── implementation.ts# Implementation phase logic
│   │   │   ├── testing.ts       # Testing phase logic
│   │   │   └── completion.ts    # Completion handling
│   │   └── transitions.ts       # Phase transition logic (including go-back)
│   │
│   ├── state/
│   │   ├── manager.ts           # State management
│   │   ├── types.ts             # State type definitions
│   │   └── storage.ts           # JSON persistence + backup
│   │
│   ├── llm/
│   │   ├── client.ts            # LLM client wrapper
│   │   ├── prompts.ts           # System prompts
│   │   ├── parser.ts            # Response parsing
│   │   └── context.ts           # Context window management
│   │
│   ├── testing/
│   │   ├── runner.ts            # Test execution
│   │   └── parsers/
│   │       └── vitest.ts        # Vitest output parser
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
│   │   ├── controller.test.ts
│   │   └── transitions.test.ts
│   ├── state/
│   │   └── manager.test.ts
│   ├── llm/
│   │   ├── parser.test.ts
│   │   └── context.test.ts
│   └── testing/
│       └── runner.test.ts
│
└── .pear/                       # Created at runtime (in CLI directory for dev)
    ├── config.json
    └── sessions/
```

### Generated Project Files

When using the CLI on a project:

```
target-project/
├── .pear/
│   ├── config.json              # Created on first run
│   └── sessions/
│       ├── {session-id}.json    # Active session
│       └── done/                # Archived sessions
│           └── {old-session}.json
│
└── src/features/{feature}/
    ├── DESIGN.md                # Phase 1 output
    ├── TEST_EVIDENCE.md         # Phase 4 output
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
| 1 | Project setup, dependencies, error types | Working TypeScript project with error handling |
| 2 | State manager + JSON storage + checkpointing | Sessions persist with backup checkpoints |
| 3 | LLM client + Anthropic integration + streaming | Can send/receive streamed messages |
| 4 | Response parser + artifact extraction | Can parse code blocks and file paths |
| 5 | CLI handler basics + display formatting | Interactive prompts with syntax highlighting |

### Week 2: Core Workflow

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | Planning phase + context management | Can complete Phase 1 end-to-end |
| 2 | Interface phase + dependency analysis | Types, tests, and implementation order |
| 3 | Implementation phase (production units) | Unit-by-unit implementation with approval |
| 4 | Implementation phase (test implementations) | Tests implemented and validated |
| 5 | Test runner + Testing phase | Tests run, failures analyzed |

### Week 3: Polish & Validation

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | Debug loop + manual test UI | Full testing phase flow |
| 2 | Completion phase + TEST_EVIDENCE.md | Session archival, summary generation |
| 3 | Go-back transitions + state preservation | Can modify earlier phases safely |
| 4 | Integration testing with real feature | Complete one real feature |
| 5 | Bug fixes, prompt refinement, documentation | Stable, documented CLI |

### Daily Checkpoints

Each day should end with:
1. Working code committed
2. Manual test of the day's feature
3. Notes on what worked/didn't work
4. Updated prompts if needed
5. State checkpoint verified (can kill and resume)

---

## Testing Strategy

### Unit Tests

```typescript
// tests/state/manager.test.ts
describe('StateManager', () => {
  it('should create new session with initial state', () => {});
  it('should persist session to JSON', () => {});
  it('should load session from JSON', () => {});
  it('should create backup before save', () => {});
  it('should restore from backup on corruption', () => {});
  it('should list incomplete sessions', () => {});
  it('should archive completed sessions', () => {});
});

// tests/workflow/controller.test.ts
describe('WorkflowController', () => {
  it('should start in planning phase', () => {});
  it('should transition to interface after planning approval', () => {});
  it('should not allow skipping phases', () => {});
  it('should preserve state when going back', () => {});
  it('should identify affected units on type change', () => {});
});

// tests/llm/parser.test.ts
describe('ResponseParser', () => {
  it('should extract code blocks with file paths', () => {});
  it('should handle multiple code blocks', () => {});
  it('should handle missing file paths', () => {});
  it('should parse markdown sections', () => {});
});

// tests/llm/context.test.ts
describe('ContextManager', () => {
  it('should always include design document', () => {});
  it('should respect token limits', () => {});
  it('should implement message sliding window', () => {});
});
```

### Integration Tests

```typescript
// tests/integration/full-workflow.test.ts
describe('Full Workflow', () => {
  it('should complete all phases for a simple feature', async () => {
    // Mock LLM responses
    // Run through all phases
    // Verify all artifacts created
  });
  
  it('should handle go-back and resume', async () => {
    // Complete through implementation
    // Go back to interface
    // Verify state preserved
    // Complete again
  });
  
  it('should recover from crash', async () => {
    // Start session
    // Simulate crash (kill process)
    // Resume session
    // Verify state intact
  });
});
```

### Manual Testing

Use the CLI to build a real feature:

**Test Feature**: Add a simple utility module with 2-3 functions

1. Start new session
2. Complete planning phase (design doc created)
3. Complete interface phase (types, test skeletons, dependency order)
4. Implement each production unit
5. Implement each test
6. Run tests, fix any failures
7. Complete manual tests
8. Verify TEST_EVIDENCE.md generated
9. Verify session archived

---

## Error Handling

### Error Types (`src/utils/errors.ts`)

```typescript
// Base error class
export class PearError extends Error {
  constructor(
    message: string,
    public readonly code: ErrorCode,
    public readonly recoverable: boolean = true,
    public readonly context?: Record<string, unknown>
  ) {
    super(message);
    this.name = 'PearError';
  }
}

type ErrorCode = 
  | 'SESSION_NOT_FOUND'
  | 'SESSION_CORRUPTED'
  | 'PHASE_TRANSITION_INVALID'
  | 'LLM_CONNECTION_FAILED'
  | 'LLM_RATE_LIMITED'
  | 'LLM_RESPONSE_INVALID'
  | 'FILE_READ_FAILED'
  | 'FILE_WRITE_FAILED'
  | 'ARTIFACT_CONFLICT'
  | 'TEST_RUNNER_FAILED'
  | 'PARSE_ERROR'
  | 'CHECKPOINT_FAILED';

// Specific error classes
export class SessionError extends PearError {
  constructor(message: string, code: 'SESSION_NOT_FOUND' | 'SESSION_CORRUPTED') {
    super(message, code, code === 'SESSION_NOT_FOUND');
  }
}

export class LLMError extends PearError {
  constructor(
    message: string, 
    code: 'LLM_CONNECTION_FAILED' | 'LLM_RATE_LIMITED' | 'LLM_RESPONSE_INVALID',
    public readonly retryable: boolean = true
  ) {
    super(message, code, true);
  }
}

export class FileError extends PearError {
  constructor(
    message: string,
    code: 'FILE_READ_FAILED' | 'FILE_WRITE_FAILED' | 'ARTIFACT_CONFLICT',
    public readonly filePath: string
  ) {
    super(message, code, true, { filePath });
  }
}

export class WorkflowError extends PearError {
  constructor(message: string, code: 'PHASE_TRANSITION_INVALID') {
    super(message, code, false);
  }
}
```

### Error Handling Strategy

| Error Type | Recovery Strategy | User Experience |
|------------|-------------------|-----------------|
| `SESSION_NOT_FOUND` | List available sessions | "Session not found. Available: ..." |
| `SESSION_CORRUPTED` | Restore from backup | "Restoring from last checkpoint..." |
| `LLM_CONNECTION_FAILED` | Retry with backoff | "Retrying in 5s..." |
| `LLM_RATE_LIMITED` | Wait and retry | "Rate limited. Waiting 60s..." |
| `FILE_WRITE_FAILED` | Show error, retry option | "Couldn't write. Check permissions." |
| `ARTIFACT_CONFLICT` | Show diff, ask user | "File exists. Overwrite?" |
| `TEST_RUNNER_FAILED` | Show raw output | "Test runner error. Output: ..." |
| `CHECKPOINT_FAILED` | Warn, continue | "Warning: Checkpoint failed." |

### Retry Logic

```typescript
async function withRetry<T>(
  operation: () => Promise<T>,
  options: {
    maxRetries: number;
    backoffMs: number;
    retryOn: ErrorCode[];
  }
): Promise<T> {
  let lastError: PearError;
  
  for (let attempt = 0; attempt < options.maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (error instanceof PearError && options.retryOn.includes(error.code)) {
        lastError = error;
        const waitTime = options.backoffMs * Math.pow(2, attempt);
        console.log(`Retrying in ${waitTime}ms... (${attempt + 1}/${options.maxRetries})`);
        await sleep(waitTime);
      } else {
        throw error;
      }
    }
  }
  
  throw lastError!;
}
```

---

## Risk Mitigation

### Risk 1: LLM Output Quality

**Risk**: LLM produces inconsistent or incorrect code

**Mitigation**:
- Iterate on prompts until output is reliable
- Add examples to system prompts if needed
- Use phase-specific temperatures (0.2 for code, 0.7 for planning)
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
- Keep backup of previous checkpoint
- Validate state structure on load
- Clear error messages for recovery

### Risk 4: Context Window Limits

**Risk**: Conversation gets too long for LLM

**Mitigation**:
- Implement sliding window for messages
- Always include design doc (source of truth)
- Summarize rather than truncate
- Monitor token usage per request

### Risk 5: Test Runner Integration

**Risk**: Test runner fails or produces unparseable output

**Mitigation**:
- Use Vitest JSON reporter for structured output
- Fall back to showing raw output on parse failure
- Allow human to interpret results and proceed

---

## Exit Criteria

Phase 1 is complete when:

1. **Functional**: Can complete all four phases + completion for a real feature
2. **Stable**: No crashes during normal operation, can recover from interruption
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
6. Was the go-back functionality useful or rarely needed?
7. Did TEST_EVIDENCE.md add value?

---

## Appendix: Dependencies

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.30.0",
    "chalk": "^5.3.0",
    "inquirer": "^9.2.0",
    "marked": "^12.0.0",
    "marked-terminal": "^7.0.0",
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

**Note**: Removed `yaml` dependency since we're using JSON for session storage in Phase 1. YAML will be added in Phase 2 for project configuration.

