# Components

**Document**: `02-components.md`  
**Purpose**: Detailed specifications for each component in the system

---

## Component Overview

| Component | Location | Responsibility |
|-----------|----------|----------------|
| CLI Handler | `src/cli/` | User interaction, display |
| Workflow Controller | `src/workflow/controller.ts` | Phase orchestration |
| State Manager | `src/state/manager.ts` | Session persistence |
| LLM Client | `src/llm/client.ts` | API communication |
| Context Manager | `src/llm/context.ts` | Token budgeting |
| Response Parser | `src/llm/parser.ts` | Extract artifacts from responses |
| File Manager | `src/files/manager.ts` | File operations |
| Test Runner | `src/testing/runner.ts` | Execute tests |

---

## 1. CLI Handler

**Location**: `src/cli/`  
**Files**: `index.ts`, `display.ts`, `prompts.ts`, `streaming.ts`

### Responsibility

Handle all terminal input/output:
- Display formatted LLM responses (code highlighting, markdown)
- Stream responses in real-time
- Prompt for user actions and free-form input
- Show progress indicators

### Interface

```typescript
interface CLIHandler {
  // ─────────────────────────────────────────────────────────────
  // Display Methods
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Display the current phase header
   * Example: "═══ Phase 2/4: Interface & Tests ═══"
   */
  displayPhaseHeader(phase: Phase, phaseNumber: number): void;
  
  /**
   * Display a complete LLM response (after streaming finishes)
   * Renders markdown and highlights code blocks
   */
  displayLLMResponse(response: string): void;
  
  /**
   * Stream LLM response tokens to terminal as they arrive
   * Returns the complete response when done
   */
  displayStreamingResponse(stream: AsyncIterable<string>): Promise<string>;
  
  /**
   * Display current workflow progress
   * Shows completed phases, current unit, etc.
   */
  displayProgress(state: WorkflowState): void;
  
  /**
   * Display an error with appropriate formatting
   */
  displayError(error: PearError): void;
  
  /**
   * Display completion summary when feature is done
   */
  displayCompletionSummary(result: CompletionResult): void;
  
  // ─────────────────────────────────────────────────────────────
  // Input Methods
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Prompt user to describe the feature they want to build
   */
  promptForFeatureDescription(): Promise<string>;
  
  /**
   * Prompt user for the feature folder path
   */
  promptForFeaturePath(): Promise<string>;
  
  /**
   * Prompt user to select an action from available options
   */
  promptForAction(actions: Action[]): Promise<Action>;
  
  /**
   * Prompt for free-form text input
   */
  promptForFreeformInput(prompt: string): Promise<string>;
  
  /**
   * Prompt for yes/no confirmation
   */
  promptForConfirmation(message: string): Promise<boolean>;
  
  /**
   * Prompt user to record manual test result
   */
  promptForManualTestResult(test: ManualTest): Promise<ManualTestResult>;
  
  /**
   * Prompt user to select from a list of incomplete sessions
   */
  promptForSessionSelection(sessions: SessionSummary[]): Promise<string | null>;
  
  // ─────────────────────────────────────────────────────────────
  // Formatting Methods
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Format code with syntax highlighting
   */
  formatCode(code: string, language: string): string;
  
  /**
   * Format markdown for terminal display
   */
  formatMarkdown(markdown: string): string;
  
  /**
   * Format a diff between two strings
   */
  formatDiff(oldContent: string, newContent: string): string;
}
```

### Action Type

```typescript
type Action = 
  | { type: 'approve' }
  | { type: 'modify'; input: string }
  | { type: 'go_back'; targetPhase: Phase }
  | { type: 'quit' };
```

### Dependencies

| Package | Purpose |
|---------|---------|
| `inquirer` | Interactive prompts |
| `chalk` | Colored terminal output |
| `marked` + `marked-terminal` | Markdown rendering |
| `highlight.js` (via marked) | Code syntax highlighting |

### Implementation Notes

- Use `process.stdout.write()` for streaming (not `console.log`)
- Clear line with `\r` for progress updates
- Handle terminal resize gracefully
- Support both interactive and non-interactive modes (for testing)

---

## 2. Workflow Controller

**Location**: `src/workflow/controller.ts`

### Responsibility

Central orchestrator that:
- Manages the four-phase workflow
- Enforces approval gates between phases
- Coordinates all other components
- Handles go-back transitions

### Interface

```typescript
interface WorkflowController {
  // ─────────────────────────────────────────────────────────────
  // Lifecycle
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Start a new session for a feature
   * Creates session, initializes state, acquires any needed resources
   */
  startNewSession(
    featureDescription: string, 
    featurePath: string
  ): Promise<Session>;
  
  /**
   * Resume an existing session
   * Loads state, validates integrity, returns to last position
   */
  resumeSession(sessionId: string): Promise<Session>;
  
  // ─────────────────────────────────────────────────────────────
  // Phase Execution
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Execute the current phase
   * Returns LLM response, artifacts, and available actions
   */
  runCurrentPhase(): Promise<PhaseResult>;
  
  /**
   * Get available actions for current state
   * Varies by phase and progress
   */
  getAvailableActions(): Action[];
  
  // ─────────────────────────────────────────────────────────────
  // State Transitions
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Approve current step and advance
   * May trigger phase transition if step is last in phase
   */
  approveCurrentStep(): Promise<void>;
  
  /**
   * Request changes to current step
   * Re-runs LLM with feedback included
   */
  requestChanges(feedback: string): Promise<void>;
  
  /**
   * Go back to an earlier phase
   * Preserves appropriate state based on transition rules
   */
  goBackToPhase(phase: Phase): Promise<GoBackResult>;
  
  /**
   * Complete the session after all tests pass
   * Archives session, generates summary
   */
  completeSession(): Promise<CompletionResult>;
  
  // ─────────────────────────────────────────────────────────────
  // Getters
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Get current workflow state
   */
  getCurrentState(): WorkflowState;
  
  /**
   * Get current phase
   */
  getCurrentPhase(): Phase;
  
  /**
   * Check if session is complete
   */
  isComplete(): boolean;
}
```

### Phase Result

```typescript
interface PhaseResult {
  /** Formatted LLM response for display */
  output: string;
  
  /** Files to create/modify (if any) */
  artifacts?: Artifact[];
  
  /** Available user actions */
  nextActions: Action[];
  
  /** Whether this completes the current phase */
  phaseComplete: boolean;
}

interface Artifact {
  path: string;
  content: string;
  action: 'create' | 'modify' | 'delete';
}
```

### Go-Back Result

```typescript
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
```

### Implementation Notes

- Create phase-specific handlers in `src/workflow/phases/`
- Use state machine pattern for transitions
- Always checkpoint after state changes
- Validate transitions before executing

---

## 3. State Manager

**Location**: `src/state/manager.ts`

### Responsibility

Manage session state:
- Create and update sessions
- Persist to JSON files
- Handle checkpoints and recovery
- Archive completed sessions

### Interface

```typescript
interface StateManager {
  // ─────────────────────────────────────────────────────────────
  // Session CRUD
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Create a new session with initial state
   */
  createSession(initial: Partial<WorkflowState>): WorkflowState;
  
  /**
   * Get a session by ID (from memory or disk)
   */
  getSession(sessionId: string): WorkflowState | null;
  
  /**
   * Update session state
   * Automatically updates lastActivityAt
   */
  updateSession(
    sessionId: string, 
    updates: Partial<WorkflowState>
  ): WorkflowState;
  
  // ─────────────────────────────────────────────────────────────
  // Persistence
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Save checkpoint to disk
   * Creates backup of previous checkpoint first
   */
  saveCheckpoint(state: WorkflowState): Promise<void>;
  
  /**
   * Load checkpoint from disk
   */
  loadCheckpoint(sessionId: string): Promise<WorkflowState | null>;
  
  /**
   * Restore from backup checkpoint
   * Used when primary checkpoint is corrupted
   */
  restoreFromBackup(sessionId: string): Promise<WorkflowState | null>;
  
  // ─────────────────────────────────────────────────────────────
  // Query
  // ─────────────────────────────────────────────────────────────
  
  /**
   * List all incomplete sessions
   */
  listIncompleteSessions(): Promise<SessionSummary[]>;
  
  /**
   * Check if a session exists
   */
  sessionExists(sessionId: string): Promise<boolean>;
  
  // ─────────────────────────────────────────────────────────────
  // Archival
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Archive a completed session
   * Moves to done/ folder, returns archive path
   */
  archiveSession(sessionId: string): Promise<string>;
  
  // ─────────────────────────────────────────────────────────────
  // Conversation History
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Add a message to the session's conversation history
   */
  addMessage(sessionId: string, message: Message): void;
  
  /**
   * Get conversation messages (optionally limited)
   */
  getMessages(sessionId: string, limit?: number): Message[];
}
```

### Session Summary

```typescript
interface SessionSummary {
  sessionId: string;
  featureName: string;
  featurePath: string;
  currentPhase: Phase;
  lastActivity: Date;
  status: 'active' | 'interrupted';
}
```

### Storage Layout

```
.pear/
├── sessions/
│   ├── abc123.json           # Active session
│   ├── abc123.backup.json    # Previous checkpoint
│   └── done/                 # Archived sessions
│       └── def456.json
└── config.json
```

### Implementation Notes

- Generate session IDs with `uuid.v4()`
- Always create backup before overwriting checkpoint
- Validate JSON structure on load
- Handle missing/corrupted files gracefully

---

## 4. LLM Client

**Location**: `src/llm/client.ts`

### Responsibility

Communicate with Anthropic API:
- Send prompts with appropriate context
- Stream responses
- Handle errors and retries

### Interface

```typescript
interface LLMClient {
  // ─────────────────────────────────────────────────────────────
  // Core Method
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Send a chat request and stream the response
   */
  chat(options: ChatOptions): AsyncIterable<string>;
  
  // ─────────────────────────────────────────────────────────────
  // Phase-Specific Convenience Methods
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Planning: Generate clarifying questions or design document
   */
  planFeature(
    description: string, 
    context: ProjectContext
  ): AsyncIterable<string>;
  
  /**
   * Interface: Generate type definitions
   */
  designInterface(
    designDoc: string, 
    context: ProjectContext
  ): AsyncIterable<string>;
  
  /**
   * Interface: Analyze dependencies for implementation order
   */
  analyzeDependencies(types: string): AsyncIterable<string>;
  
  /**
   * Implementation: Generate code for a single unit
   */
  implementUnit(
    unit: ImplementationUnit, 
    context: ImplementationContext
  ): AsyncIterable<string>;
  
  /**
   * Implementation: Generate test implementation
   */
  implementTest(
    test: TestImplementation, 
    context: ImplementationContext
  ): AsyncIterable<string>;
  
  /**
   * Testing: Analyze failures and propose fixes
   */
  debugFailure(
    failure: TestFailure, 
    context: ImplementationContext
  ): AsyncIterable<string>;
  
  /**
   * Completion: Generate TEST_EVIDENCE.md
   */
  generateTestEvidence(state: WorkflowState): AsyncIterable<string>;
}
```

### Chat Options

```typescript
interface ChatOptions {
  systemPrompt: string;
  messages: Message[];
  temperature?: number;
  maxTokens?: number;
}

// Default temperatures by phase
const PHASE_TEMPERATURES: Record<Phase, number> = {
  planning: 0.7,       // Creative for brainstorming
  interface: 0.3,      // Precise for types
  implementation: 0.2, // Very precise for code
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
```

### Context Types

```typescript
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

interface FileContent {
  path: string;
  content: string;
}
```

### Implementation Notes

- Use `@anthropic-ai/sdk` for API communication
- Implement retry logic with exponential backoff
- Handle rate limiting gracefully
- Log token usage for monitoring

---

## 5. Context Manager

**Location**: `src/llm/context.ts`

### Responsibility

Build appropriate context for LLM requests:
- Manage token budgets
- Implement sliding window for messages
- Prioritize relevant context

See [06-context-management.md](./06-context-management.md) for detailed specification.

### Interface

```typescript
interface ContextManager {
  /**
   * Build context window for a specific phase
   */
  buildContext(state: WorkflowState, phase: Phase): ContextWindow;
  
  /**
   * Estimate tokens for a piece of text
   */
  estimateTokens(text: string): number;
  
  /**
   * Trim context to fit within token limit
   */
  trimToLimit(context: ContextWindow, maxTokens: number): ContextWindow;
}

interface ContextWindow {
  systemPrompt: string;
  designDocument: string;
  typeDefinitions?: string;
  relevantTests?: string;
  currentUnit?: string;
  recentMessages: Message[];
  previousUnits?: string[];
  estimatedTokens: number;
}
```

---

## 6. Response Parser

**Location**: `src/llm/parser.ts`

### Responsibility

Extract structured data from LLM responses:
- Parse code blocks with file paths
- Extract artifacts for file creation
- Parse structured sections

See [05-response-parsing.md](./05-response-parsing.md) for detailed specification.

### Interface

```typescript
interface ResponseParser {
  /**
   * Extract code blocks from markdown response
   */
  parseCodeBlocks(response: string): ParsedCodeBlock[];
  
  /**
   * Extract artifacts (files to create/modify)
   */
  parseArtifacts(response: string): Artifact[];
  
  /**
   * Extract structured sections (headings + content)
   */
  parseSections(response: string): ParsedSection[];
  
  /**
   * Validate TypeScript syntax (basic check)
   */
  validateTypeScript(code: string): ValidationResult;
}

interface ParsedCodeBlock {
  language: string;
  filePath: string | null;
  content: string;
  startLine: number;
  endLine: number;
}

interface ParsedSection {
  heading: string;
  level: number;
  content: string;
}

interface ValidationResult {
  valid: boolean;
  errors: string[];
}
```

---

## 7. File Manager

**Location**: `src/files/manager.ts`

### Responsibility

Handle all file system operations:
- Read and write files
- Create directories
- Apply artifacts from LLM responses
- Detect project configuration

### Interface

```typescript
interface FileManager {
  // ─────────────────────────────────────────────────────────────
  // Read Operations
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Read file contents
   * Returns null if file doesn't exist
   */
  readFile(path: string): Promise<string | null>;
  
  /**
   * List directory contents
   */
  readDirectory(path: string): Promise<string[]>;
  
  /**
   * Check if file exists
   */
  fileExists(path: string): Promise<boolean>;
  
  /**
   * Check if directory exists
   */
  directoryExists(path: string): Promise<boolean>;
  
  // ─────────────────────────────────────────────────────────────
  // Write Operations
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Write content to file
   * Creates parent directories if needed
   */
  writeFile(path: string, content: string): Promise<void>;
  
  /**
   * Create directory (recursive)
   */
  createDirectory(path: string): Promise<void>;
  
  /**
   * Delete file
   */
  deleteFile(path: string): Promise<void>;
  
  // ─────────────────────────────────────────────────────────────
  // Artifact Operations
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Apply multiple artifacts (create/modify/delete files)
   * Returns results for each operation
   */
  applyArtifacts(artifacts: Artifact[]): Promise<ApplyResult>;
  
  /**
   * Generate preview of what artifacts would do
   * For user confirmation before applying
   */
  previewArtifacts(artifacts: Artifact[]): string;
  
  // ─────────────────────────────────────────────────────────────
  // Project Detection
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Detect project type from config files
   */
  detectProjectType(): Promise<ProjectType>;
  
  /**
   * Detect test framework from config files
   */
  detectTestFramework(): Promise<TestFramework>;
}

interface ApplyResult {
  created: string[];
  modified: string[];
  deleted: string[];
  failed: Array<{ path: string; error: string }>;
}

type ProjectType = 'typescript' | 'javascript' | 'unknown';
type TestFramework = 'vitest' | 'jest' | 'mocha' | 'unknown';
```

### Implementation Notes

- Use `fs/promises` for async operations
- Always use absolute paths internally
- Handle permission errors gracefully
- Create parent directories automatically

---

## 8. Test Runner

**Location**: `src/testing/runner.ts`

### Responsibility

Execute tests and parse results:
- Run Vitest with JSON reporter
- Parse structured output
- Provide results for Phase 4

### Interface

```typescript
interface TestRunner {
  // ─────────────────────────────────────────────────────────────
  // Detection
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Detect test framework in project
   */
  detectTestFramework(projectPath: string): Promise<TestFramework>;
  
  // ─────────────────────────────────────────────────────────────
  // Execution
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Run all tests matching options
   */
  runTests(options: TestRunOptions): Promise<TestRunResult>;
  
  /**
   * Run a single specific test
   */
  runSingleTest(
    testPath: string, 
    testName: string
  ): Promise<TestResult>;
  
  // ─────────────────────────────────────────────────────────────
  // Parsing
  // ─────────────────────────────────────────────────────────────
  
  /**
   * Parse test runner output into structured result
   */
  parseTestOutput(
    output: string, 
    framework: TestFramework
  ): TestRunResult;
}

interface TestRunOptions {
  projectPath: string;
  testPattern?: string;       // e.g., "**/*.test.ts"
  specificTests?: string[];   // Run only these tests
  timeout?: number;           // milliseconds
}

interface TestRunResult {
  passed: number;
  failed: number;
  skipped: number;
  duration: number;           // milliseconds
  results: TestResult[];
  failures: TestFailure[];
  timestamp: Date;
}

interface TestResult {
  name: string;
  fullName: string;           // Including describe block
  status: 'passed' | 'failed' | 'skipped';
  duration: number;
  error?: TestError;
}

interface TestError {
  message: string;
  expected?: string;
  received?: string;
  stack?: string;
}

interface TestFailure {
  testName: string;
  fullName: string;
  expected: string;
  received: string;
  errorMessage: string;
  filePath?: string;
  lineNumber?: number;
}
```

### Implementation Notes

- Use `child_process.spawn` to run `npx vitest run --reporter=json`
- Parse JSON output for structured results
- Fall back to raw output if parsing fails
- Set reasonable timeout (default: 60s)

---

## Related Documents

- [03-data-models.md](./03-data-models.md) — TypeScript interfaces
- [01-architecture.md](./01-architecture.md) — System overview
- [09-file-structure.md](./09-file-structure.md) — Project organization

