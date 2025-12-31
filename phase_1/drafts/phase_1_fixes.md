# Phase 1 Fixes

This document contains all the fixes needed for `phase_1.md` to align with the README.md vision and address identified gaps.

---

## Table of Contents

1. [Fix #1: Directory Naming (.pear-cli → .pear)](#fix-1-directory-naming)
2. [Fix #2: Add Test Runner Component](#fix-2-add-test-runner-component)
3. [Fix #3: Add Test Implementation to Phase 3](#fix-3-add-test-implementation-to-phase-3)
4. [Fix #4: Add TEST_EVIDENCE.md Artifact](#fix-4-add-test_evidencemd-artifact)
5. [Fix #5: Add Completion Phase](#fix-5-add-completion-phase)
6. [Fix #6: Define Go-Back State Preservation Logic](#fix-6-define-go-back-state-preservation-logic)
7. [Fix #7: Specify Dependency Order Determination](#fix-7-specify-dependency-order-determination)
8. [Fix #8: Define LLM Response Parsing Strategy](#fix-8-define-llm-response-parsing-strategy)
9. [Fix #9: Add Error Types Specification](#fix-9-add-error-types-specification)
10. [Fix #10: Standardize Status Values](#fix-10-standardize-status-values)
11. [Fix #11: Add Context Window Management](#fix-11-add-context-window-management)
12. [Fix #12: Improve Manual Test Tracking](#fix-12-improve-manual-test-tracking)
13. [Fix #13: Specify Temperature Values](#fix-13-specify-temperature-values)
14. [Fix #14: Update Implementation Plan](#fix-14-update-implementation-plan)
15. [Summary of All Changes](#summary-of-all-changes)

---

## Fix #1: Directory Naming

### Problem
phase_1.md uses `.pear-cli/` but README.md uses `.pear/` consistently.

### Solution
Replace ALL occurrences of `.pear-cli` with `.pear` throughout the document.

### Changes

**In "Storage Format" section (line ~286-294):**

Replace:
```
.pear-cli/
├── sessions/
│   ├── abc123.json      # Active session
│   └── def456.json      # Another session
└── config.json          # CLI configuration
```

With:
```
.pear/
├── sessions/
│   ├── abc123.json      # Active session
│   └── def456.json      # Another session
└── config.json          # CLI configuration
```

**In "CLI Application Structure" section (line ~737-740):**

Replace:
```
└── .pear-cli/                   # Created at runtime
    ├── config.json
    └── sessions/
```

With:
```
└── .pear/                       # Created at runtime (in target project)
    ├── config.json
    └── sessions/
```

---

## Fix #2: Add Test Runner Component

### Problem
No component exists for running tests in Phase 4.

### Solution
Add a new `TestRunner` component to the architecture.

### Add to Architecture Diagram (after line ~109)

Update the high-level architecture to include TestRunner:

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
│         │                         │                         │   │
│         │                 ┌───────────────┐                 │   │
│         │                 │  Test Runner  │                 │   │
│         │                 └───────────────┘                 │   │
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

### Add New Component Section (after File Manager, ~line 365)

```markdown
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
  watch?: boolean;           // For future use
}

type TestFramework = 'vitest' | 'jest' | 'mocha' | 'unknown';

interface TestResult {
  name: string;
  status: 'passed' | 'failed' | 'skipped';
  duration: number;         // milliseconds
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

**Dependencies**:
- No additional npm dependencies (uses Node.js `child_process`)
```

### Update File Structure (line ~687)

Add to `src/` structure:
```
│   ├── testing/
│   │   ├── runner.ts           # Test execution
│   │   └── parsers/
│   │       └── vitest.ts       # Vitest output parser
```

---

## Fix #3: Add Test Implementation to Phase 3

### Problem
Phase 2 creates test skeletons, but it's unclear who implements the actual test code.

### Solution
Clarify that Phase 3 implements BOTH production code AND test implementations. Add a new unit type for tests.

### Update Data Models Section

**Replace ImplementationState (line ~414-418):**

```typescript
interface ImplementationState {
  units: ImplementationUnit[];
  currentUnitIndex: number;
  dependencyOrder: string[];  // IDs in implementation order
  testImplementations: TestImplementation[];  // NEW
  currentTestIndex: number;  // NEW
}

// NEW: Separate tracking for test implementations
interface TestImplementation {
  id: string;
  testName: string;         // e.g., "should create session from valid OAuth code"
  testFilePath: string;
  status: 'pending' | 'in_progress' | 'awaiting_approval' | 'approved';
  implementation: string | null;
  relatedUnitIds: string[]; // Which production units this tests
}
```

### Update Implementation Phase Prompt

**Replace IMPLEMENTATION_SYSTEM_PROMPT (line ~527-553):**

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
- Follow the testing patterns of the project (Vitest for Phase 1)

Output format for production code:
\`\`\`typescript
// src/features/{feature}/{filename}.ts
[code]
\`\`\`

Output format for test implementations:
\`\`\`typescript
// src/features/{feature}/__tests__/{feature}.test.ts
// Test: {test name}
[test implementation]
\`\`\`

**Implementation notes:**
- [Brief explanation of key decisions]
- [Any assumptions made]
- [Dependencies on other units]

If anything is unclear about the requirements, ASK before implementing.`;
```

### Add Implementation Order Clarification

**Add new subsection under Phase 3 workflow (after line ~246):**

```markdown
#### Implementation Order

Phase 3 proceeds in this order:

1. **Production units** (in dependency order, leaf → root)
2. **Test implementations** (in order of the production units they test)

This ensures:
- Production code exists before its tests are implemented
- Tests can import and use the actual implementations
- Each test can run immediately after implementation

Example order for Auth feature:
```
Production:
  1. getOAuthUrl()        ← No dependencies
  2. createSession()      ← No dependencies  
  3. handleCallback()     ← Depends on getOAuthUrl, createSession
  4. validateSession()    ← Depends on createSession
  5. logout()             ← Depends on validateSession

Tests:
  6. test: getOAuthUrl should return valid URL
  7. test: getOAuthUrl should include state parameter
  8. test: createSession should create with correct expiry
  9. test: handleCallback should create session
  10. test: handleCallback should reject invalid code
  ... (remaining tests)
```
```

---

## Fix #4: Add TEST_EVIDENCE.md Artifact

### Problem
README mentions TEST_EVIDENCE.md as a Phase 4 artifact, but it's missing from phase_1.md.

### Solution
Add TEST_EVIDENCE.md generation to Phase 4 and update TestingState.

### Update TestingState (line ~430-434)

**Replace:**
```typescript
interface TestingState {
  lastTestRun: TestRunResult | null;
  debugIterations: number;
  manualTestsComplete: boolean;
}
```

**With:**
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
  testEvidencePath: string | null;  // e.g., "src/features/auth/TEST_EVIDENCE.md"
  testEvidenceGenerated: boolean;
}

interface ManualTest {
  id: string;
  description: string;
  status: 'pending' | 'passed' | 'failed' | 'skipped';
  verifiedBy: string | null;       // Human name/identifier
  verifiedAt: Date | null;
  notes: string | null;
}
```

### Update Testing Phase Prompt

**Replace TESTING_SYSTEM_PROMPT (line ~558-584):**

```typescript
const TESTING_SYSTEM_PROMPT = `You are Pear, an AI pair-programming assistant verifying implementations.

Your role in this phase:
1. Analyze test failures and propose fixes
2. Generate manual test checklists
3. Generate the TEST_EVIDENCE.md document

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

When all tests pass, generate TEST_EVIDENCE.md:
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
| {test name} | ✓ Pass | {ms}ms |

## Manual Tests

| Test Case | Status | Verified By | Date | Notes |
|-----------|--------|-------------|------|-------|
| {description} | ✓ Pass | {name} | {date} | {notes} |

## Sign-off
Feature verified and ready for merge.
\`\`\``;
```

### Update Generated Project Files (line ~742-759)

**Replace:**
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

**With:**
```
target-project/
├── .pear/
│   ├── config.json              # Created on first run
│   └── sessions/
│       └── {session-id}.json    # Session state
│
└── src/features/{feature}/
    ├── DESIGN.md                # Phase 1 output
    ├── TEST_EVIDENCE.md         # Phase 4 output (NEW)
    ├── types.ts                 # Phase 2 output
    ├── index.ts                 # Phase 3 output
    └── __tests__/
        └── {feature}.test.ts    # Phase 2-3 output
```

---

## Fix #5: Add Completion Phase

### Problem
`Phase` type includes `'complete'` but there's no handling for it.

### Solution
Add completion phase specification, even if simplified for CLI.

### Add Completion Phase Section (after Testing Phase Prompt, ~line 584)

```markdown
### Completion Phase Handling

When all tests pass and manual verification is complete, the session transitions to `'complete'` status.

**Completion Actions (Simplified for CLI)**:

```typescript
interface CompletionActions {
  // Archive session
  archiveSession(sessionId: string): Promise<void>;
  
  // Generate summary
  generateCompletionSummary(state: WorkflowState): string;
}
```

**Completion Flow**:

```
[All tests pass]
      │
      ▼
┌─────────────────┐
│ Generate        │
│ TEST_EVIDENCE.md│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Display Summary │
│ - Files created │
│ - Tests passed  │
│ - Time taken    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Archive Session │
│ (move to        │
│  sessions/done/)│
└────────┬────────┘
         │
         ▼
   Session Complete
```

**CLI Output on Completion**:

```
═══════════════════════════════════════════════════════════════
🎉 Feature Complete: User Authentication
═══════════════════════════════════════════════════════════════

Summary:
├── Design: src/features/auth/DESIGN.md
├── Types: src/features/auth/types.ts  
├── Implementation: 5 units across 2 files
├── Tests: 12 unit tests (all passing)
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

**Note**: Full completion features (lock release, project map update) are deferred to Phase 2 when folder locking is implemented.
```

### Update File Structure for Archived Sessions

**Update Storage Format (line ~286-294):**

```
.pear/
├── sessions/
│   ├── abc123.json      # Active session
│   └── done/            # Archived sessions (NEW)
│       └── def456.json
└── config.json          # CLI configuration
```

---

## Fix #6: Define Go-Back State Preservation Logic

### Problem
Going back between phases lacks specification for state preservation.

### Solution
Add detailed go-back logic specification.

### Add New Section: Phase Transitions (after State Machine, ~line 246)

```markdown
### Phase Transition Logic

#### Going Forward

Forward transitions are straightforward:
- Planning → Interface: Design doc approved, save to file
- Interface → Implementation: Types and tests approved, save to files
- Implementation → Testing: All units approved, run initial tests
- Testing → Complete: All tests pass, manual verification done

#### Going Backward

Going backward requires careful state management to preserve work while allowing changes.

**Interface → Planning (Amend Design)**

```typescript
interface AmendDesignTransition {
  // What's preserved
  preservedState: {
    interfaceState: InterfaceState;      // Types and tests saved
    implementationState: ImplementationState;  // Any approved units
  };
  
  // What changes
  changes: {
    currentPhase: 'planning';
    phaseStatus: 'in_progress';
    // Design doc reopened for editing
  };
}
```

User flow:
1. User requests "amend design" from Interface phase
2. System saves current Interface state
3. Design document reopened for modification
4. After re-approval, return to Interface phase
5. System asks: "Design changed. Review types/tests for compatibility?"

**Implementation → Interface (Modify API)**

```typescript
interface ModifyAPITransition {
  // What's preserved
  preservedState: {
    implementationUnits: ImplementationUnit[];  // Approved units saved
  };
  
  // What's tracked for invalidation
  affectedUnits: string[];  // Unit IDs that may need reimplementation
  
  // How affected units are determined
  determineAffected(oldTypes: string, newTypes: string): string[];
}
```

Affected unit determination:
1. Parse old and new type definitions
2. Compare function signatures, type shapes
3. Units that use changed types are marked as `'needs_review'`
4. Human confirms which units to reimplement

Example:
```
[Pear]: You changed the Session interface (added 'refreshToken' field).
        
        These units may be affected:
        ☐ createSession() - creates Session objects
        ☐ handleCallback() - returns Session
        ☐ validateSession() - reads Session
        
        Which units need reimplementation?
        
        [All of them] [Select specific] [None - changes are backward compatible]
```

**Testing → Interface (Test Spec Wrong)**

When a test itself is incorrect (not the implementation):

```typescript
interface FixTestSpecTransition {
  // Return to Interface phase
  currentPhase: 'interface';
  
  // Mark specific test for modification
  testToModify: string;  // Test name/ID
  
  // Preserve implementation
  preserveImplementation: true;
}
```

User flow:
1. Test fails, but human determines test is wrong
2. Human selects "Fix test specification"
3. Return to Interface phase, specific test highlighted
4. Modify test, get approval
5. Return to Testing phase, rerun tests

#### State Preservation Rules

| Transition | Preserved | Potentially Invalidated |
|------------|-----------|------------------------|
| Interface → Planning | Interface state, Implementation state | Nothing (design is source of truth) |
| Implementation → Interface | Approved units | Units using changed types |
| Testing → Interface | All implementation | Modified test |
| Testing → Implementation | Unaffected units | Unit being debugged |

#### Implementation in WorkflowController

```typescript
interface WorkflowController {
  // ... existing methods ...
  
  // Go back with state preservation
  goBackToPhase(targetPhase: Phase): Promise<GoBackResult>;
}

interface GoBackResult {
  previousPhase: Phase;
  targetPhase: Phase;
  preservedState: Partial<WorkflowState>;
  invalidatedItems: InvalidatedItem[];
  userActionRequired: boolean;
  userPrompt?: string;  // What to ask the user
}

interface InvalidatedItem {
  type: 'unit' | 'test' | 'type';
  id: string;
  name: string;
  reason: string;
}
```
```

---

## Fix #7: Specify Dependency Order Determination

### Problem
It's unclear how implementation dependency order is determined.

### Solution
Specify that the LLM determines order during Interface phase, with human override.

### Add to Interface Phase Section (new subsection after Interface Prompt)

```markdown
### Dependency Order Determination

During Phase 2 (Interface), after types are approved, the LLM analyzes the type definitions to determine implementation order.

**Process**:

1. LLM examines function signatures and their parameter/return types
2. Builds a dependency graph based on type relationships
3. Proposes an implementation order (leaf nodes first)
4. Human can approve or modify the order

**LLM Output Format for Dependency Analysis**:

```markdown
## Implementation Order

Based on the type definitions, here's the suggested implementation order:

### Dependency Graph
```
createSession() ─────┐
                     ├──▶ handleCallback()
getOAuthUrl() ───────┘         │
                               ▼
validateSession() ◀───── uses Session type
        │
        ▼
   logout()
```

### Suggested Order
1. **getOAuthUrl()** - No internal dependencies
2. **createSession()** - No internal dependencies  
3. **handleCallback()** - Calls getOAuthUrl(), createSession()
4. **validateSession()** - Uses Session type from createSession
5. **logout()** - Calls validateSession()

### Rationale
- Leaf functions (no dependencies) implemented first
- Each function can be tested immediately after implementation
- No forward references required

[Approve Order] [Modify Order]
```

**Data Model Update**:

```typescript
interface InterfaceState {
  typesContent: string | null;
  typesPath: string | null;
  testsContent: string | null;
  testsPath: string | null;
  approved: boolean;
  
  // NEW: Dependency analysis
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
```

**Human Override**:

The human can modify the order if they have domain knowledge the LLM lacks:

```
? The suggested order is:
  1. getOAuthUrl
  2. createSession
  3. handleCallback
  4. validateSession
  5. logout

  Do you want to modify this order? (y/N)
> y

? Enter the new order (comma-separated):
> createSession, getOAuthUrl, handleCallback, validateSession, logout

Order updated. Proceeding to Implementation phase.
```
```

---

## Fix #8: Define LLM Response Parsing Strategy

### Problem
`src/llm/parser.ts` is listed but not specified.

### Solution
Add detailed parser specification.

### Add New Section: Response Parsing (after LLM Client section, ~line 332)

```markdown
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
  
  // Parse test results from test runner output
  parseTestResults(output: string, framework: TestFramework): TestRunResult;
}

interface ParsedCodeBlock {
  language: string;         // 'typescript', 'markdown', etc.
  filePath: string | null;  // Extracted from comment, e.g., "// src/features/auth/types.ts"
  content: string;          // The code itself
  startLine: number;        // Position in original response
  endLine: number;
}

interface ParsedSection {
  heading: string;          // e.g., "Implementation notes"
  level: number;            // 1 = #, 2 = ##, etc.
  content: string;
}
```

**Parsing Rules**:

1. **Code Block Extraction**:
   - Match triple backticks with optional language tag
   - Extract file path from first line comment: `// path/to/file.ts`
   - Handle nested code blocks (rare, but possible)

2. **File Path Patterns**:
   ```typescript
   const FILE_PATH_PATTERNS = [
     /^\/\/ (.+\.(ts|js|tsx|jsx|md))$/m,           // // src/file.ts
     /^\/\/ File: (.+)$/m,                          // // File: src/file.ts
     /^# (.+\.(md))$/m,                             // # DESIGN.md (for markdown)
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
         action: 'create' as const,  // Default; could detect 'modify' from context
       }));
   }
   ```

4. **Handling Ambiguity**:
   - If no file path in comment, ask user where to save
   - If multiple blocks for same file, merge or ask user
   - If language tag missing, infer from file extension

**Example Parse**:

Input:
```markdown
Here are the type definitions:

\`\`\`typescript
// src/features/auth/types.ts
export interface AuthConfig {
  clientId: string;
}
\`\`\`

And here's the implementation:

\`\`\`typescript
// src/features/auth/index.ts
import { AuthConfig } from './types';

export function init(config: AuthConfig) {
  // ...
}
\`\`\`
```

Output:
```typescript
[
  {
    language: 'typescript',
    filePath: 'src/features/auth/types.ts',
    content: 'export interface AuthConfig {\n  clientId: string;\n}',
    startLine: 5,
    endLine: 8
  },
  {
    language: 'typescript',
    filePath: 'src/features/auth/index.ts',
    content: 'import { AuthConfig } from \'./types\';\n\nexport function init(config: AuthConfig) {\n  // ...\n}',
    startLine: 14,
    endLine: 20
  }
]
```
```

---

## Fix #9: Add Error Types Specification

### Problem
`src/utils/errors.ts` is listed but not specified.

### Solution
Define error types and handling strategy.

### Add New Section: Error Handling (before Appendix, ~line 915)

```markdown
## Error Handling

### Error Types (`src/utils/errors.ts`)

```typescript
// Base error class for Pear errors
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
| `SESSION_NOT_FOUND` | List available sessions | "Session not found. Available sessions: ..." |
| `SESSION_CORRUPTED` | Offer to restore from backup | "Session corrupted. Restore from last checkpoint?" |
| `LLM_CONNECTION_FAILED` | Retry with backoff | "Connection failed. Retrying in 5s..." |
| `LLM_RATE_LIMITED` | Wait and retry | "Rate limited. Waiting 60s before retry..." |
| `LLM_RESPONSE_INVALID` | Retry with same prompt | "Invalid response. Retrying..." |
| `FILE_WRITE_FAILED` | Show error, ask to retry | "Couldn't write {file}. Check permissions." |
| `ARTIFACT_CONFLICT` | Show diff, ask user | "File exists with different content. Overwrite?" |
| `TEST_RUNNER_FAILED` | Show raw output | "Test runner failed. Raw output: ..." |
| `CHECKPOINT_FAILED` | Warn, continue | "Warning: Checkpoint failed. Progress may be lost." |

### CLI Error Display

```typescript
function displayError(error: PearError): void {
  console.error(chalk.red(`\n❌ Error: ${error.message}`));
  
  if (error.context) {
    console.error(chalk.gray(`   Context: ${JSON.stringify(error.context)}`));
  }
  
  if (error.recoverable) {
    console.log(chalk.yellow(`\n   This error is recoverable.`));
    // Offer recovery options based on error type
  } else {
    console.log(chalk.red(`\n   This error requires manual intervention.`));
  }
}
```

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
        console.log(chalk.yellow(`Retrying in ${waitTime}ms... (attempt ${attempt + 1}/${options.maxRetries})`));
        await sleep(waitTime);
      } else {
        throw error;
      }
    }
  }
  
  throw lastError!;
}
```
```

---

## Fix #10: Standardize Status Values

### Problem
`PhaseStatus` uses `'awaiting_approval'` but `ImplementationUnit.status` uses `'awaiting_review'`.

### Solution
Standardize on `'awaiting_approval'` throughout.

### Update ImplementationUnit (line ~420-428)

**Replace:**
```typescript
interface ImplementationUnit {
  id: string;
  name: string;
  description: string;
  filePath: string;
  status: 'pending' | 'in_progress' | 'awaiting_review' | 'approved';
  implementation: string | null;
  dependencies: string[];  // IDs of units this depends on
}
```

**With:**
```typescript
interface ImplementationUnit {
  id: string;
  name: string;
  description: string;
  filePath: string;
  status: UnitStatus;
  implementation: string | null;
  dependencies: string[];  // IDs of units this depends on
}

// Unified status type for consistency
type UnitStatus = 'pending' | 'in_progress' | 'awaiting_approval' | 'approved' | 'needs_review';

// Note: 'needs_review' is used when going back from Implementation to Interface
// to mark units that may need reimplementation due to type changes
```

---

## Fix #11: Add Context Window Management

### Problem
Context window management is mentioned as a risk but has no concrete implementation.

### Solution
Add specification for context management.

### Add New Section: Context Management (after LLM Client, ~line 332)

```markdown
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

// Token estimation (rough: ~4 chars per token for English/code)
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
| Implementation | System prompt, Design doc, Types, Current unit spec | Previous 2-3 units | Older messages, older units |
| Testing | System prompt, Design doc, Types, Failed test details | Implementation code | Older messages |

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
  
  // Add recent messages from the end, newest first
  for (let i = allMessages.length - 1; i > 0; i--) {
    const msgTokens = estimateTokens(allMessages[i].content);
    if (tokenCount + msgTokens > maxTokens) break;
    
    result.splice(1, 0, allMessages[i]);  // Insert after first message
    tokenCount += msgTokens;
  }
  
  // Add summary if we skipped messages
  if (result.length < allMessages.length) {
    const skipped = allMessages.length - result.length;
    result.splice(1, 0, {
      role: 'system',
      content: `[${skipped} earlier messages summarized: Discussion of feature requirements and design decisions]`,
      timestamp: new Date(),
      phase: result[1].phase
    });
  }
  
  return result;
}
```

**Token Budget Allocation**:

```
Total Budget: 100,000 tokens
├── Reserved for Response: 4,000
├── System Prompt: ~500
├── Design Document: ~1,000
├── Type Definitions: ~500-2,000
├── Current Unit Context: ~500-1,000
├── Test Context: ~500-1,000
├── Message History: ~10,000-20,000
└── Buffer: ~70,000+ remaining
```

For Phase 1, we have plenty of headroom. Context management becomes critical in Phase 2+ when:
- Multiple features exist
- Project map is included
- Longer conversation histories accumulate
```

---

## Fix #12: Improve Manual Test Tracking

### Problem
`manualTestsComplete: boolean` is too simple for proper tracking.

### Solution
Already addressed in Fix #4, but let's ensure the CLI handles manual tests properly.

### Add Manual Test UI Flow (to User Interface section)

```markdown
### Manual Test Recording

After automated tests pass, the CLI guides manual testing:

```
═══════════════════════════════════════════════════════════════
Phase 4/4: Testing - Manual Verification
═══════════════════════════════════════════════════════════════

Automated tests: ✓ 12/12 passed

Now let's verify the feature manually. Here's the checklist:

  1. [ ] Navigate to /login, click "Sign in with Google"
       → Should redirect to Google OAuth page
  
  2. [ ] Complete Google sign-in with valid credentials
       → Should redirect back to app with session created
  
  3. [ ] Access a protected route (/dashboard)
       → Should display dashboard content
  
  4. [ ] Access protected route in incognito (no session)
       → Should redirect to /login
  
  5. [ ] Click "Logout" button
       → Should clear session and redirect to home

? Select a test to record result: (Use arrow keys)
❯ 1. Navigate to /login... [pending]
  2. Complete Google sign-in... [pending]
  3. Access protected route... [pending]
  4. Access protected route in incognito... [pending]
  5. Click "Logout"... [pending]
  ──────────────────────────────────
  Mark all as passed
  Skip manual tests
```

**Recording a Test Result**:

```
? Test: Navigate to /login, click "Sign in with Google"
  Expected: Should redirect to Google OAuth page
  
  Result: (Use arrow keys)
❯ ✓ Pass
  ✗ Fail
  ⊘ Skip

? Notes (optional):
> Redirect took ~2 seconds, but worked correctly

✓ Test recorded: Pass

? Select next test: ...
```

**Data Recording**:

```typescript
interface ManualTestRecord {
  id: string;
  description: string;
  expectedResult: string;
  status: 'pending' | 'passed' | 'failed' | 'skipped';
  notes: string | null;
  recordedAt: Date;
}
```
```

---

## Fix #13: Specify Temperature Values

### Problem
Temperature values are only mentioned in Risk Mitigation as "0.2-0.3".

### Solution
Add explicit temperature configuration per phase.

### Add to LLM Client Section (update ChatOptions interface)

**Update ChatOptions:**
```typescript
interface ChatOptions {
  systemPrompt: string;
  messages: Message[];
  temperature?: number;
  maxTokens?: number;
}

// Default temperatures by phase
const PHASE_TEMPERATURES: Record<Phase, number> = {
  planning: 0.7,      // More creative for brainstorming and questions
  interface: 0.3,     // Precise for type definitions
  implementation: 0.2, // Very precise for code generation
  testing: 0.3,       // Precise for debugging, slightly creative for suggestions
  complete: 0.3,      // Precise for summaries
};

// Default max tokens by phase
const PHASE_MAX_TOKENS: Record<Phase, number> = {
  planning: 2000,      // Design docs are concise
  interface: 3000,     // Types + tests can be longer
  implementation: 1500, // One unit at a time
  testing: 1500,       // Debug suggestions
  complete: 1000,      // Summary
};
```

---

## Fix #14: Update Implementation Plan

### Problem
Implementation plan doesn't explicitly include checkpoint/recovery work.

### Solution
Update the implementation plan with specific checkpoint and error handling days.

### Replace Implementation Plan Section (line ~762-793)

```markdown
## Implementation Plan

### Week 1: Foundation

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | Project setup, dependencies, error types | Working TypeScript project with error handling |
| 2 | State manager + JSON storage + checkpointing | Sessions persist with backup checkpoints |
| 3 | LLM client + Anthropic integration + streaming | Can send/receive streamed messages |
| 4 | Response parser + artifact extraction | Can parse code blocks and file paths |
| 5 | CLI handler basics + display formatting | Interactive prompts with syntax highlighting |

### Week 2: Complete Workflow

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | Planning phase + context management | Can complete Phase 1 end-to-end |
| 2 | Interface phase + dependency analysis | Types, tests, and implementation order |
| 3 | Implementation phase (production units) | Unit-by-unit implementation with approval |
| 4 | Implementation phase (test units) + Test runner | Tests implemented and runnable |
| 5 | Testing phase + debug loop + manual tests | Test failures analyzed, manual test UI |

### Week 3: Polish & Validation (Buffer)

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | Completion phase + TEST_EVIDENCE.md | Full workflow end-to-end |
| 2 | Session recovery + go-back transitions | Can resume interrupted sessions, modify earlier phases |
| 3 | Integration testing with real feature | Complete one real feature using CLI |
| 4 | Bug fixes, prompt refinement | Stable, usable CLI |
| 5 | Documentation, learnings capture | Ready for Phase 2 decision |

### Daily Checkpoints

Each day should end with:
1. Working code committed
2. Manual test of the day's feature
3. Notes on what worked/didn't work
4. Updated prompts if needed
5. State checkpoint verified (can kill and resume)

### Risk Buffer

Week 3 provides buffer for:
- LLM prompt iteration (if quality is poor)
- Unexpected bugs in state management
- Context window issues
- Integration challenges
```

---

## Summary of All Changes

### Files to Modify in phase_1.md:

1. **Directory naming**: `.pear-cli/` → `.pear/` (multiple locations)

2. **Architecture diagram**: Add TestRunner component

3. **New Component**: Add Test Runner section with interface

4. **Data Models**:
   - Update `ImplementationState` with test tracking
   - Add `TestImplementation` interface
   - Update `TestingState` with manual tests and evidence
   - Add `ManualTest` interface
   - Add `UnitStatus` type (standardized)
   - Add `DependencyAnalysis` to `InterfaceState`

5. **LLM Prompts**:
   - Update `IMPLEMENTATION_SYSTEM_PROMPT` for test implementations
   - Update `TESTING_SYSTEM_PROMPT` for TEST_EVIDENCE.md

6. **New Sections**:
   - Response Parser specification
   - Context Manager specification
   - Error Handling section with error types
   - Phase Transition Logic (go-back rules)
   - Completion Phase Handling
   - Dependency Order Determination
   - Manual Test Recording UI

7. **Updated Sections**:
   - Generated Project Files (add TEST_EVIDENCE.md)
   - Storage Format (add done/ folder)
   - Implementation Plan (add checkpointing, Week 3)
   - File Structure (add testing/, context.ts)

8. **Constants**:
   - Add `PHASE_TEMPERATURES`
   - Add `PHASE_MAX_TOKENS`

### Estimated Additional Work:

These fixes add approximately 2-3 days of implementation work:
- Test Runner: 0.5 days
- Response Parser: 0.5 days  
- Context Manager: 0.5 days
- Error Types: 0.25 days
- Go-Back Logic: 0.5 days
- Manual Test UI: 0.25 days
- TEST_EVIDENCE.md generation: 0.25 days

The updated 3-week plan accounts for this additional scope.

