# Data Models

**Document**: `03-data-models.md`  
**Purpose**: Complete TypeScript type definitions for Phase 1

---

## Overview

This document contains all TypeScript interfaces and types needed to implement Phase 1. These should be created as the first step of implementation to establish type safety throughout the codebase.

---

## Core Types

### Phase and Status

```typescript
/**
 * The five phases of the workflow
 * Note: 'complete' is a terminal state, not an active phase
 */
type Phase = 
  | 'planning' 
  | 'interface' 
  | 'implementation' 
  | 'testing' 
  | 'complete';

/**
 * Status of the current phase
 */
type PhaseStatus = 
  | 'pending'            // Not yet started
  | 'in_progress'        // Active work happening
  | 'awaiting_approval'  // LLM produced output, waiting for human
  | 'approved';          // Human approved, ready to proceed

/**
 * Status for implementation units and tests
 */
type UnitStatus = 
  | 'pending'            // Not yet started
  | 'in_progress'        // Currently being implemented
  | 'awaiting_approval'  // Implementation ready for review
  | 'approved'           // Human approved
  | 'needs_review';      // May need reimplementation (after go-back)
```

### Session Identity

```typescript
/**
 * Summary of a session for display/selection
 */
interface SessionSummary {
  sessionId: string;
  featureName: string;
  featurePath: string;
  currentPhase: Phase;
  lastActivity: Date;
  status: 'active' | 'interrupted';
}

/**
 * A running session (returned by startNewSession/resumeSession)
 */
interface Session {
  sessionId: string;
  state: WorkflowState;
}
```

---

## Workflow State

### Main State Object

```typescript
/**
 * Complete state of a workflow session
 * This is what gets persisted to JSON
 */
interface WorkflowState {
  // ─────────────────────────────────────────────────────────────
  // Identity
  // ─────────────────────────────────────────────────────────────
  
  /** Unique session identifier (UUID) */
  sessionId: string;
  
  /** Human-readable feature name (from initial description) */
  featureName: string;
  
  /** File system path for the feature (e.g., "src/features/auth") */
  featurePath: string;
  
  // ─────────────────────────────────────────────────────────────
  // Timing
  // ─────────────────────────────────────────────────────────────
  
  /** When the session was created */
  createdAt: Date;
  
  /** When the session was last active (updated on any state change) */
  lastActivityAt: Date;
  
  // ─────────────────────────────────────────────────────────────
  // Current Position
  // ─────────────────────────────────────────────────────────────
  
  /** Current phase of the workflow */
  currentPhase: Phase;
  
  /** Status within the current phase */
  phaseStatus: PhaseStatus;
  
  // ─────────────────────────────────────────────────────────────
  // Phase-Specific State
  // ─────────────────────────────────────────────────────────────
  
  planning: PlanningState;
  interface: InterfaceState;
  implementation: ImplementationState;
  testing: TestingState;
  
  // ─────────────────────────────────────────────────────────────
  // Conversation History
  // ─────────────────────────────────────────────────────────────
  
  /** All messages in the session */
  messages: Message[];
}
```

### Planning State

```typescript
/**
 * State for Phase 1: Planning
 */
interface PlanningState {
  /** Content of the design document (markdown) */
  designDocContent: string | null;
  
  /** Path where design doc will be/is saved */
  designDocPath: string | null;
  
  /** Whether the design has been approved */
  approved: boolean;
}
```

### Interface State

```typescript
/**
 * State for Phase 2: Interface & Tests
 */
interface InterfaceState {
  /** Content of the types file */
  typesContent: string | null;
  
  /** Path where types will be/are saved */
  typesPath: string | null;
  
  /** Content of the test file (skeletons) */
  testsContent: string | null;
  
  /** Path where tests will be/are saved */
  testsPath: string | null;
  
  /** Whether types and tests have been approved */
  approved: boolean;
  
  /** Dependency analysis for implementation order */
  dependencyAnalysis: DependencyAnalysis | null;
}

/**
 * Analysis of function dependencies for implementation ordering
 */
interface DependencyAnalysis {
  /** Dependency graph nodes */
  graph: DependencyNode[];
  
  /** Function names in suggested implementation order */
  suggestedOrder: string[];
  
  /** Whether the human approved this order */
  orderApproved: boolean;
}

/**
 * A node in the dependency graph
 */
interface DependencyNode {
  /** Name of the function */
  functionName: string;
  
  /** Functions this one calls (dependencies) */
  dependsOn: string[];
  
  /** Functions that call this one (dependents) */
  dependedOnBy: string[];
}
```

### Implementation State

```typescript
/**
 * State for Phase 3: Implementation
 */
interface ImplementationState {
  // ─────────────────────────────────────────────────────────────
  // Production Code
  // ─────────────────────────────────────────────────────────────
  
  /** Production code units to implement */
  units: ImplementationUnit[];
  
  /** Index of the current unit being worked on */
  currentUnitIndex: number;
  
  /** Order in which to implement units (by ID) */
  dependencyOrder: string[];
  
  // ─────────────────────────────────────────────────────────────
  // Test Implementation
  // ─────────────────────────────────────────────────────────────
  
  /** Test implementations (filling in skeletons from Phase 2) */
  testImplementations: TestImplementation[];
  
  /** Index of the current test being implemented */
  currentTestIndex: number;
  
  // ─────────────────────────────────────────────────────────────
  // Progress Tracking
  // ─────────────────────────────────────────────────────────────
  
  /** Current stage: production code or tests */
  stage: 'production' | 'tests';
}

/**
 * A single unit of implementation (function, class, etc.)
 */
interface ImplementationUnit {
  /** Unique identifier */
  id: string;
  
  /** Function/class name */
  name: string;
  
  /** Brief description of what this unit does */
  description: string;
  
  /** File where this will be implemented */
  filePath: string;
  
  /** Current status */
  status: UnitStatus;
  
  /** The implementation code (null until generated) */
  implementation: string | null;
  
  /** IDs of units this depends on */
  dependencies: string[];
}

/**
 * A test implementation (filling in a skeleton)
 */
interface TestImplementation {
  /** Unique identifier */
  id: string;
  
  /** The test name (from the 'it' description) */
  testName: string;
  
  /** File where this test lives */
  testFilePath: string;
  
  /** Current status */
  status: UnitStatus;
  
  /** The test implementation code (null until generated) */
  implementation: string | null;
  
  /** IDs of production units this test covers */
  relatedUnitIds: string[];
}
```

### Testing State

```typescript
/**
 * State for Phase 4: Testing
 */
interface TestingState {
  // ─────────────────────────────────────────────────────────────
  // Automated Tests
  // ─────────────────────────────────────────────────────────────
  
  /** History of all test runs */
  testRuns: TestRunResult[];
  
  /** Most recent test run (convenience) */
  lastTestRun: TestRunResult | null;
  
  /** Number of debug iterations (fixes attempted) */
  debugIterations: number;
  
  // ─────────────────────────────────────────────────────────────
  // Manual Tests
  // ─────────────────────────────────────────────────────────────
  
  /** Manual tests to verify */
  manualTests: ManualTest[];
  
  /** Whether all manual tests are complete */
  manualTestsComplete: boolean;
  
  // ─────────────────────────────────────────────────────────────
  // Evidence Document
  // ─────────────────────────────────────────────────────────────
  
  /** Path to TEST_EVIDENCE.md (once generated) */
  testEvidencePath: string | null;
  
  /** Whether the evidence document has been generated */
  testEvidenceGenerated: boolean;
}

/**
 * Result of running tests
 */
interface TestRunResult {
  /** Number of tests that passed */
  passed: number;
  
  /** Number of tests that failed */
  failed: number;
  
  /** Number of tests that were skipped */
  skipped: number;
  
  /** Total duration in milliseconds */
  duration: number;
  
  /** Individual test results */
  results: TestResult[];
  
  /** Details of failures (for debugging) */
  failures: TestFailure[];
  
  /** When this run happened */
  timestamp: Date;
}

/**
 * Result of a single test
 */
interface TestResult {
  /** Test name (from 'it' block) */
  name: string;
  
  /** Full name including describe blocks */
  fullName: string;
  
  /** Pass/fail/skip status */
  status: 'passed' | 'failed' | 'skipped';
  
  /** Duration in milliseconds */
  duration: number;
  
  /** Error details (if failed) */
  error?: TestError;
}

/**
 * Error from a test failure
 */
interface TestError {
  message: string;
  expected?: string;
  received?: string;
  stack?: string;
}

/**
 * Structured failure for debugging
 */
interface TestFailure {
  /** Test name */
  testName: string;
  
  /** Full test name */
  fullName: string;
  
  /** Expected value (if assertion failure) */
  expected: string;
  
  /** Received value (if assertion failure) */
  received: string;
  
  /** Error message */
  errorMessage: string;
  
  /** File where test is defined */
  filePath?: string;
  
  /** Line number of failure */
  lineNumber?: number;
}

/**
 * A manual test to be verified by the human
 */
interface ManualTest {
  /** Unique identifier */
  id: string;
  
  /** What to do (e.g., "Click login button") */
  description: string;
  
  /** What should happen (e.g., "Redirects to OAuth page") */
  expectedResult: string;
  
  /** Current status */
  status: 'pending' | 'passed' | 'failed' | 'skipped';
  
  /** Who verified this test */
  verifiedBy: string | null;
  
  /** When it was verified */
  verifiedAt: Date | null;
  
  /** Any notes from verification */
  notes: string | null;
}
```

---

## Messages and Conversation

```typescript
/**
 * A message in the conversation history
 */
interface Message {
  /** Who sent this message */
  role: 'user' | 'assistant' | 'system';
  
  /** Message content */
  content: string;
  
  /** When the message was sent */
  timestamp: Date;
  
  /** Which phase this message was sent during */
  phase: Phase;
}
```

---

## Actions and Results

### User Actions

```typescript
/**
 * Actions a user can take at any checkpoint
 */
type Action = 
  | { type: 'approve' }
  | { type: 'modify'; input: string }
  | { type: 'go_back'; targetPhase: Phase }
  | { type: 'quit' };
```

### Phase Results

```typescript
/**
 * Result of running a phase step
 */
interface PhaseResult {
  /** Formatted output for display */
  output: string;
  
  /** Files to create/modify */
  artifacts?: Artifact[];
  
  /** Available actions for the user */
  nextActions: Action[];
  
  /** Whether this completes the current phase */
  phaseComplete: boolean;
}

/**
 * A file to be created, modified, or deleted
 */
interface Artifact {
  /** File path (relative to project root) */
  path: string;
  
  /** File content */
  content: string;
  
  /** What to do with this file */
  action: 'create' | 'modify' | 'delete';
}

/**
 * Result of applying artifacts
 */
interface ApplyResult {
  /** Files that were created */
  created: string[];
  
  /** Files that were modified */
  modified: string[];
  
  /** Files that were deleted */
  deleted: string[];
  
  /** Operations that failed */
  failed: Array<{ path: string; error: string }>;
}
```

### Go-Back Results

```typescript
/**
 * Result of going back to an earlier phase
 */
interface GoBackResult {
  /** Phase we were in */
  previousPhase: Phase;
  
  /** Phase we're going to */
  targetPhase: Phase;
  
  /** State that was preserved */
  preservedState: Partial<WorkflowState>;
  
  /** Items that may need re-work */
  invalidatedItems: InvalidatedItem[];
  
  /** Whether user needs to take action */
  userActionRequired: boolean;
  
  /** Prompt to show user (if action required) */
  userPrompt?: string;
}

/**
 * An item that may need to be re-implemented
 */
interface InvalidatedItem {
  /** Type of item */
  type: 'unit' | 'test' | 'type';
  
  /** Item identifier */
  id: string;
  
  /** Human-readable name */
  name: string;
  
  /** Why it may need re-work */
  reason: string;
}
```

### Completion Result

```typescript
/**
 * Result of completing a session
 */
interface CompletionResult {
  /** Whether completion succeeded */
  success: boolean;
  
  /** Human-readable summary */
  summary: string;
  
  /** Files that were created */
  filesCreated: string[];
  
  /** Files that were modified */
  filesModified: string[];
  
  /** Number of tests that passed */
  testsPassed: number;
  
  /** Where the session was archived */
  sessionArchivePath: string;
}
```

---

## Parser Types

```typescript
/**
 * A code block extracted from LLM response
 */
interface ParsedCodeBlock {
  /** Language tag (e.g., "typescript", "markdown") */
  language: string;
  
  /** File path extracted from comment (null if not found) */
  filePath: string | null;
  
  /** The code content */
  content: string;
  
  /** Starting line in original response */
  startLine: number;
  
  /** Ending line in original response */
  endLine: number;
}

/**
 * A section extracted from markdown
 */
interface ParsedSection {
  /** Section heading text */
  heading: string;
  
  /** Heading level (1 = #, 2 = ##, etc.) */
  level: number;
  
  /** Section content */
  content: string;
}

/**
 * Result of validating code
 */
interface ValidationResult {
  /** Whether the code is valid */
  valid: boolean;
  
  /** Validation errors (if any) */
  errors: string[];
}
```

---

## Context Types

```typescript
/**
 * Context window for LLM request
 */
interface ContextWindow {
  /** System prompt for the phase */
  systemPrompt: string;
  
  /** Design document (always included) */
  designDocument: string;
  
  /** Type definitions (Phase 2+) */
  typeDefinitions?: string;
  
  /** Relevant test cases */
  relevantTests?: string;
  
  /** Current unit being implemented */
  currentUnit?: string;
  
  /** Recent conversation messages */
  recentMessages: Message[];
  
  /** Previous units for context */
  previousUnits?: string[];
  
  /** Estimated total tokens */
  estimatedTokens: number;
}

/**
 * Generic project context for LLM
 */
interface ProjectContext {
  designDocument?: string;
  existingTypes?: string;
  relevantFiles?: FileContent[];
}

/**
 * Extended context for implementation
 */
interface ImplementationContext extends ProjectContext {
  designDocument: string;
  typeDefinitions: string;
  testCases: string;
  previousUnits: ImplementationUnit[];
}

/**
 * Content of a file for context
 */
interface FileContent {
  path: string;
  content: string;
}
```

---

## LLM Types

```typescript
/**
 * Options for LLM chat request
 */
interface ChatOptions {
  /** System prompt */
  systemPrompt: string;
  
  /** Conversation messages */
  messages: Message[];
  
  /** Temperature (0-1, lower = more deterministic) */
  temperature?: number;
  
  /** Maximum tokens in response */
  maxTokens?: number;
}
```

---

## Test Runner Types

```typescript
/**
 * Options for running tests
 */
interface TestRunOptions {
  /** Path to project root */
  projectPath: string;
  
  /** Glob pattern for test files */
  testPattern?: string;
  
  /** Specific tests to run */
  specificTests?: string[];
  
  /** Timeout in milliseconds */
  timeout?: number;
}

/**
 * Detected test framework
 */
type TestFramework = 'vitest' | 'jest' | 'mocha' | 'unknown';

/**
 * Detected project type
 */
type ProjectType = 'typescript' | 'javascript' | 'unknown';
```

---

## Configuration Types

```typescript
/**
 * CLI configuration (stored in .pear/config.json)
 */
interface PearConfig {
  /** Version of the config schema */
  version: number;
  
  /** Default feature path pattern */
  defaultFeaturePath?: string;
  
  /** Test command override */
  testCommand?: string;
}
```

---

## Type Exports

All types should be exported from a central location:

```typescript
// src/types/index.ts

// Core types
export type { Phase, PhaseStatus, UnitStatus };
export type { Session, SessionSummary };
export type { WorkflowState };

// Phase states
export type { PlanningState };
export type { InterfaceState, DependencyAnalysis, DependencyNode };
export type { ImplementationState, ImplementationUnit, TestImplementation };
export type { TestingState, TestRunResult, TestResult, TestError, TestFailure, ManualTest };

// Messages
export type { Message };

// Actions and results
export type { Action };
export type { PhaseResult, Artifact, ApplyResult };
export type { GoBackResult, InvalidatedItem };
export type { CompletionResult };

// Parser types
export type { ParsedCodeBlock, ParsedSection, ValidationResult };

// Context types
export type { ContextWindow, ProjectContext, ImplementationContext, FileContent };

// LLM types
export type { ChatOptions };

// Test runner types
export type { TestRunOptions, TestFramework, ProjectType };

// Configuration
export type { PearConfig };
```

---

## Related Documents

- [02-components.md](./02-components.md) — Component interfaces using these types
- [07-phase-transitions.md](./07-phase-transitions.md) — State transitions
- [09-file-structure.md](./09-file-structure.md) — Where types file lives

