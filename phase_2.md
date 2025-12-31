# Phase 2: Core Engine Library

**Duration**: 2-3 weeks  
**Goal**: Extract and refine the CLI prototype into a clean, well-tested, reusable library that can power the VS Code fork.

**Prerequisites**: Phase 1 complete with validated workflow

---

## Table of Contents

1. [Objectives](#objectives)
2. [Success Criteria](#success-criteria)
3. [Scope](#scope)
4. [Architecture](#architecture)
5. [Module Design](#module-design)
6. [API Design](#api-design)
7. [Data Models](#data-models)
8. [LLM Abstraction](#llm-abstraction)
9. [Storage Abstraction](#storage-abstraction)
10. [Testing Strategy](#testing-strategy)
11. [Implementation Plan](#implementation-plan)
12. [Migration from Phase 1](#migration-from-phase-1)
13. [Exit Criteria](#exit-criteria)

---

## Objectives

### Primary Objectives

1. **Clean separation of concerns**: Each module has a single responsibility
2. **Testable in isolation**: Every component can be unit tested without dependencies
3. **UI-agnostic**: The library knows nothing about CLI or VS Code
4. **Extensible**: Support for multiple LLM providers, languages, and storage backends
5. **Type-safe**: Full TypeScript types with strict mode

### Secondary Objectives

1. Improve prompt quality based on Phase 1 learnings
2. Add comprehensive error handling
3. Implement proper logging
4. Prepare for folder locking (Phase 3 will activate it)
5. Document public APIs

---

## Success Criteria

| Criteria | Measurement |
|----------|-------------|
| 100% of Phase 1 functionality preserved | All workflows still work |
| >90% test coverage on core modules | Coverage report |
| Zero UI dependencies in core library | No CLI/DOM imports |
| Can swap LLM provider | Works with mock provider |
| Can swap storage backend | Works with in-memory storage |
| <100ms for non-LLM operations | Performance benchmarks |

---

## Scope

### In Scope

- Workflow state machine (extracted and improved)
- Context manager (new, more sophisticated)
- LLM orchestrator with provider abstraction
- Session manager with recovery
- Lock manager (API ready, single-machine for now)
- Artifact store
- Project analyzer (for onboarding)
- Full TypeScript type definitions
- Comprehensive test suite

### Out of Scope (Deferred)

- VS Code integration (Phase 3)
- Distributed locking (Phase 4)
- Multiple LLM provider implementations beyond Claude (Phase 4)
- Advanced project analysis heuristics

---

## Architecture

### Package Structure

```
@pear/core/
├── src/
│   ├── index.ts                 # Public exports
│   │
│   ├── workflow/                # Workflow orchestration
│   │   ├── index.ts
│   │   ├── state-machine.ts
│   │   ├── phases/
│   │   │   ├── planning.ts
│   │   │   ├── interface.ts
│   │   │   ├── implementation.ts
│   │   │   └── testing.ts
│   │   └── transitions.ts
│   │
│   ├── context/                 # Context management
│   │   ├── index.ts
│   │   ├── manager.ts
│   │   └── builder.ts
│   │
│   ├── llm/                     # LLM abstraction
│   │   ├── index.ts
│   │   ├── orchestrator.ts
│   │   ├── providers/
│   │   │   ├── base.ts          # Abstract provider
│   │   │   ├── anthropic.ts
│   │   │   └── mock.ts          # For testing
│   │   └── prompts/
│   │       ├── index.ts
│   │       ├── planning.ts
│   │       ├── interface.ts
│   │       ├── implementation.ts
│   │       └── testing.ts
│   │
│   ├── session/                 # Session management
│   │   ├── index.ts
│   │   ├── manager.ts
│   │   ├── recovery.ts
│   │   └── storage/
│   │       ├── base.ts          # Abstract storage
│   │       ├── file.ts          # File-based (YAML)
│   │       └── memory.ts        # In-memory (testing)
│   │
│   ├── lock/                    # Folder locking
│   │   ├── index.ts
│   │   ├── manager.ts
│   │   └── storage/
│   │       ├── base.ts
│   │       └── file.ts
│   │
│   ├── project/                 # Project analysis
│   │   ├── index.ts
│   │   ├── analyzer.ts
│   │   └── detectors/
│   │       ├── language.ts
│   │       ├── test-framework.ts
│   │       └── structure.ts
│   │
│   ├── artifacts/               # File artifact management
│   │   ├── index.ts
│   │   ├── store.ts
│   │   └── templates/
│   │       ├── design-doc.ts
│   │       └── test-evidence.ts
│   │
│   └── types/                   # Shared type definitions
│       ├── index.ts
│       ├── workflow.ts
│       ├── session.ts
│       ├── llm.ts
│       └── project.ts
│
├── tests/
│   ├── workflow/
│   ├── context/
│   ├── llm/
│   ├── session/
│   ├── lock/
│   ├── project/
│   └── integration/
│
├── package.json
├── tsconfig.json
└── vitest.config.ts
```

### Dependency Graph

```
                          ┌─────────────┐
                          │   types/    │
                          │  (shared)   │
                          └──────┬──────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
        ▼                        ▼                        ▼
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│   session/  │          │    llm/     │          │   project/  │
│   storage   │          │  providers  │          │  analyzer   │
└──────┬──────┘          └──────┬──────┘          └──────┬──────┘
       │                        │                        │
       ▼                        ▼                        │
┌─────────────┐          ┌─────────────┐                 │
│   session/  │          │    llm/     │                 │
│   manager   │          │ orchestrator│                 │
└──────┬──────┘          └──────┬──────┘                 │
       │                        │                        │
       │    ┌─────────────┐     │                        │
       │    │   context/  │◀────┘                        │
       │    │   manager   │                              │
       │    └──────┬──────┘                              │
       │           │                                     │
       ▼           ▼                                     ▼
┌─────────────────────────────────────────────────────────────┐
│                      workflow/                              │
│                    state-machine                            │
└─────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│                     Public API                              │
│              (PearEngine class)                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Module Design

### 1. Workflow Module (`src/workflow/`)

**Responsibility**: Manage the four-phase workflow state machine.

```typescript
// src/workflow/state-machine.ts

export class WorkflowStateMachine {
  private state: WorkflowState;
  private readonly transitions: TransitionMap;
  
  constructor(initialState: WorkflowState) {
    this.state = initialState;
    this.transitions = buildTransitionMap();
  }
  
  // State access
  getState(): Readonly<WorkflowState> {
    return Object.freeze({ ...this.state });
  }
  
  getCurrentPhase(): Phase {
    return this.state.currentPhase;
  }
  
  // Transitions
  canTransition(action: WorkflowAction): boolean {
    const transition = this.transitions.get(this.state.currentPhase, action);
    return transition !== undefined && transition.guard(this.state);
  }
  
  transition(action: WorkflowAction): WorkflowState {
    if (!this.canTransition(action)) {
      throw new InvalidTransitionError(this.state.currentPhase, action);
    }
    
    const transition = this.transitions.get(this.state.currentPhase, action)!;
    this.state = transition.apply(this.state);
    return this.getState();
  }
  
  // Phase-specific updates
  updatePlanningState(updates: Partial<PlanningState>): void;
  updateInterfaceState(updates: Partial<InterfaceState>): void;
  updateImplementationState(updates: Partial<ImplementationState>): void;
  updateTestingState(updates: Partial<TestingState>): void;
}

// Transitions are explicit and testable
interface Transition {
  from: Phase;
  action: WorkflowAction;
  to: Phase;
  guard: (state: WorkflowState) => boolean;
  apply: (state: WorkflowState) => WorkflowState;
}

type WorkflowAction =
  | 'approve_planning'
  | 'approve_interface'
  | 'approve_unit'
  | 'approve_all_units'
  | 'start_testing'
  | 'tests_passed'
  | 'complete'
  | 'go_back_to_planning'
  | 'go_back_to_interface';
```

### 2. Context Module (`src/context/`)

**Responsibility**: Build and manage context windows for LLM calls.

```typescript
// src/context/manager.ts

export class ContextManager {
  private readonly maxTokens: number;
  private readonly tokenCounter: TokenCounter;
  
  constructor(options: ContextManagerOptions) {
    this.maxTokens = options.maxTokens ?? 100000;
    this.tokenCounter = options.tokenCounter ?? new TikTokenCounter();
  }
  
  buildContext(state: WorkflowState, files: FileMap): ContextWindow {
    const builder = new ContextBuilder(this.maxTokens, this.tokenCounter);
    
    // Always include (highest priority)
    builder.addRequired('designDocument', state.planning.designDocContent);
    builder.addRequired('currentPhase', state.currentPhase);
    
    // Phase-specific content
    switch (state.currentPhase) {
      case 'interface':
        builder.addRequired('typeDefinitions', state.interface.typesContent);
        break;
        
      case 'implementation':
        builder.addRequired('typeDefinitions', state.interface.typesContent);
        builder.addRequired('testCases', state.interface.testsContent);
        builder.addOptional('currentUnit', this.getCurrentUnit(state));
        builder.addOptional('previousUnits', this.getPreviousUnits(state, 3));
        break;
        
      case 'testing':
        builder.addRequired('implementation', this.getImplementation(state, files));
        builder.addOptional('testResults', state.testing.lastTestRun);
        break;
    }
    
    // Conversation history (sliding window)
    builder.addConversation(state.messages, { maxMessages: 20 });
    
    return builder.build();
  }
  
  // Summarize older context if needed
  summarizePhase(phase: Phase, state: WorkflowState): string;
}

// src/context/builder.ts
export class ContextBuilder {
  private sections: ContextSection[] = [];
  private usedTokens = 0;
  
  addRequired(key: string, content: string | null): this;
  addOptional(key: string, content: string | null): this;
  addConversation(messages: Message[], options: ConversationOptions): this;
  
  build(): ContextWindow {
    // Prioritize required, then fill with optional
    // Truncate conversation if needed
  }
}
```

### 3. LLM Module (`src/llm/`)

**Responsibility**: Abstract LLM interactions, manage providers, construct prompts.

```typescript
// src/llm/providers/base.ts

export interface LLMProvider {
  readonly name: string;
  
  // Core method - streaming response
  chat(request: ChatRequest): AsyncIterable<ChatChunk>;
  
  // Optional capabilities
  supportsStreaming(): boolean;
  supportsToolUse(): boolean;
  getMaxContextTokens(): number;
}

export interface ChatRequest {
  systemPrompt: string;
  messages: ChatMessage[];
  temperature?: number;
  maxTokens?: number;
}

export interface ChatChunk {
  type: 'text' | 'tool_use' | 'done';
  content?: string;
  toolCall?: ToolCall;
}

// src/llm/orchestrator.ts

export class LLMOrchestrator {
  private readonly provider: LLMProvider;
  private readonly promptRegistry: PromptRegistry;
  
  constructor(provider: LLMProvider, prompts?: PromptRegistry) {
    this.provider = provider;
    this.promptRegistry = prompts ?? defaultPromptRegistry;
  }
  
  // Phase-specific methods that construct appropriate prompts
  async *planFeature(
    description: string, 
    context: ContextWindow
  ): AsyncIterable<string> {
    const prompt = this.promptRegistry.get('planning');
    const request = this.buildRequest(prompt, context, [
      { role: 'user', content: description }
    ]);
    
    for await (const chunk of this.provider.chat(request)) {
      if (chunk.type === 'text' && chunk.content) {
        yield chunk.content;
      }
    }
  }
  
  async *designInterface(context: ContextWindow): AsyncIterable<string>;
  async *implementUnit(unit: ImplementationUnit, context: ContextWindow): AsyncIterable<string>;
  async *debugFailure(failure: TestFailure, context: ContextWindow): AsyncIterable<string>;
  
  // Response parsing
  parseDesignDocument(response: string): ParsedDesignDocument;
  parseTypeDefinitions(response: string): ParsedTypes;
  parseImplementation(response: string): ParsedImplementation;
}

// src/llm/providers/anthropic.ts

export class AnthropicProvider implements LLMProvider {
  private readonly client: Anthropic;
  private readonly model: string;
  
  constructor(options: AnthropicOptions) {
    this.client = new Anthropic({ apiKey: options.apiKey });
    this.model = options.model ?? 'claude-sonnet-4-20250514';
  }
  
  async *chat(request: ChatRequest): AsyncIterable<ChatChunk> {
    const stream = await this.client.messages.stream({
      model: this.model,
      system: request.systemPrompt,
      messages: this.convertMessages(request.messages),
      max_tokens: request.maxTokens ?? 4096,
      temperature: request.temperature ?? 0.3,
    });
    
    for await (const event of stream) {
      if (event.type === 'content_block_delta') {
        yield { type: 'text', content: event.delta.text };
      }
    }
    
    yield { type: 'done' };
  }
}

// src/llm/providers/mock.ts

export class MockProvider implements LLMProvider {
  private responses: Map<string, string> = new Map();
  
  // For testing: set canned responses
  setResponse(promptContains: string, response: string): void {
    this.responses.set(promptContains, response);
  }
  
  async *chat(request: ChatRequest): AsyncIterable<ChatChunk> {
    // Find matching response based on message content
    const response = this.findMatchingResponse(request);
    
    // Simulate streaming by yielding chunks
    for (const word of response.split(' ')) {
      yield { type: 'text', content: word + ' ' };
      await sleep(10); // Simulate delay
    }
    
    yield { type: 'done' };
  }
}
```

### 4. Session Module (`src/session/`)

**Responsibility**: Manage session lifecycle, persistence, and recovery.

```typescript
// src/session/manager.ts

export class SessionManager {
  private readonly storage: SessionStorage;
  private activeSessions: Map<string, WorkflowState> = new Map();
  
  constructor(storage: SessionStorage) {
    this.storage = storage;
  }
  
  // Lifecycle
  async createSession(options: CreateSessionOptions): Promise<Session> {
    const sessionId = generateSessionId();
    const state = createInitialState(sessionId, options);
    
    await this.storage.save(sessionId, state);
    this.activeSessions.set(sessionId, state);
    
    return new Session(sessionId, state, this);
  }
  
  async loadSession(sessionId: string): Promise<Session | null> {
    const state = await this.storage.load(sessionId);
    if (!state) return null;
    
    this.activeSessions.set(sessionId, state);
    return new Session(sessionId, state, this);
  }
  
  // Recovery
  async listIncompleteSessions(): Promise<SessionInfo[]> {
    return this.storage.listIncomplete();
  }
  
  async detectInterruptedSessions(): Promise<SessionInfo[]> {
    const incomplete = await this.listIncompleteSessions();
    return incomplete.filter(s => s.state === 'interrupted');
  }
  
  // Checkpointing
  async checkpoint(sessionId: string, state: WorkflowState): Promise<void> {
    state.lastActivityAt = new Date();
    await this.storage.save(sessionId, state);
    this.activeSessions.set(sessionId, state);
  }
  
  // Completion
  async completeSession(sessionId: string): Promise<void> {
    const state = this.activeSessions.get(sessionId);
    if (!state) throw new SessionNotFoundError(sessionId);
    
    state.currentPhase = 'complete';
    state.complete = {
      completedAt: new Date(),
      lockReleased: false,
      archived: false,
    };
    
    await this.storage.archive(sessionId, state);
    this.activeSessions.delete(sessionId);
  }
}

// src/session/storage/base.ts

export interface SessionStorage {
  save(sessionId: string, state: WorkflowState): Promise<void>;
  load(sessionId: string): Promise<WorkflowState | null>;
  delete(sessionId: string): Promise<void>;
  archive(sessionId: string, state: WorkflowState): Promise<void>;
  listIncomplete(): Promise<SessionInfo[]>;
  listArchived(): Promise<SessionInfo[]>;
}

// src/session/storage/file.ts

export class FileSessionStorage implements SessionStorage {
  private readonly basePath: string;
  
  constructor(basePath: string) {
    this.basePath = basePath;
  }
  
  async save(sessionId: string, state: WorkflowState): Promise<void> {
    const path = join(this.basePath, 'sessions', `${sessionId}.yaml`);
    await writeFile(path, yaml.stringify(state));
  }
  
  async load(sessionId: string): Promise<WorkflowState | null> {
    const path = join(this.basePath, 'sessions', `${sessionId}.yaml`);
    try {
      const content = await readFile(path, 'utf-8');
      return yaml.parse(content) as WorkflowState;
    } catch {
      return null;
    }
  }
  
  // ... other methods
}
```

### 5. Lock Module (`src/lock/`)

**Responsibility**: Manage folder-level write locks for parallel sessions.

```typescript
// src/lock/manager.ts

export class LockManager {
  private readonly storage: LockStorage;
  
  constructor(storage: LockStorage) {
    this.storage = storage;
  }
  
  // Acquire write lock on a folder
  async acquireWriteLock(
    path: string, 
    sessionId: string, 
    metadata: LockMetadata
  ): Promise<LockResult> {
    const existing = await this.storage.getLock(path);
    
    if (existing && existing.sessionId !== sessionId) {
      return {
        success: false,
        reason: 'locked_by_other',
        holder: existing,
      };
    }
    
    const lock: FolderLock = {
      path,
      sessionId,
      type: 'write',
      acquiredAt: new Date(),
      ...metadata,
    };
    
    await this.storage.setLock(path, lock);
    return { success: true, lock };
  }
  
  // Release lock
  async releaseLock(path: string, sessionId: string): Promise<void> {
    const existing = await this.storage.getLock(path);
    
    if (existing && existing.sessionId === sessionId) {
      await this.storage.deleteLock(path);
    }
  }
  
  // Check if path is readable (always true - reads don't need locks)
  canRead(path: string): boolean {
    return true;
  }
  
  // Check if session can write to path
  async canWrite(path: string, sessionId: string): Promise<boolean> {
    const lock = await this.storage.getLock(path);
    return !lock || lock.sessionId === sessionId;
  }
  
  // List all locks
  async listLocks(): Promise<FolderLock[]> {
    return this.storage.listLocks();
  }
  
  // Release all locks for a session
  async releaseAllForSession(sessionId: string): Promise<void> {
    const locks = await this.listLocks();
    for (const lock of locks) {
      if (lock.sessionId === sessionId) {
        await this.storage.deleteLock(lock.path);
      }
    }
  }
  
  // Cleanup stale locks (older than timeout)
  async cleanupStaleLocks(timeoutMs: number): Promise<FolderLock[]> {
    const locks = await this.listLocks();
    const stale: FolderLock[] = [];
    const now = Date.now();
    
    for (const lock of locks) {
      if (now - lock.acquiredAt.getTime() > timeoutMs) {
        await this.storage.deleteLock(lock.path);
        stale.push(lock);
      }
    }
    
    return stale;
  }
}
```

### 6. Project Module (`src/project/`)

**Responsibility**: Analyze project structure for onboarding.

```typescript
// src/project/analyzer.ts

export class ProjectAnalyzer {
  private readonly detectors: Detector[];
  
  constructor(detectors?: Detector[]) {
    this.detectors = detectors ?? defaultDetectors;
  }
  
  async analyze(rootPath: string): Promise<ProjectMap> {
    const results = await Promise.all(
      this.detectors.map(d => d.detect(rootPath))
    );
    
    return this.mergeResults(results);
  }
  
  private mergeResults(results: DetectorResult[]): ProjectMap {
    return {
      version: 1,
      analyzedAt: new Date(),
      structure: this.detectStructure(results),
      languages: this.detectLanguages(results),
      testing: this.detectTesting(results),
      existingModules: this.detectModules(results),
    };
  }
}

// src/project/detectors/language.ts

export class LanguageDetector implements Detector {
  async detect(rootPath: string): Promise<DetectorResult> {
    const indicators = {
      typescript: ['tsconfig.json', 'package.json'],
      python: ['pyproject.toml', 'setup.py', 'requirements.txt'],
      go: ['go.mod'],
      rust: ['Cargo.toml'],
    };
    
    const detected: string[] = [];
    
    for (const [lang, files] of Object.entries(indicators)) {
      for (const file of files) {
        if (await fileExists(join(rootPath, file))) {
          detected.push(lang);
          break;
        }
      }
    }
    
    return { type: 'language', detected };
  }
}
```

---

## API Design

### Public API (`PearEngine`)

The main entry point for consumers of the library:

```typescript
// src/index.ts

export class PearEngine {
  private readonly sessionManager: SessionManager;
  private readonly lockManager: LockManager;
  private readonly llmOrchestrator: LLMOrchestrator;
  private readonly contextManager: ContextManager;
  private readonly projectAnalyzer: ProjectAnalyzer;
  
  constructor(options: PearEngineOptions) {
    // Wire up dependencies
    const storage = options.sessionStorage ?? new FileSessionStorage(options.pearDir);
    const lockStorage = options.lockStorage ?? new FileLockStorage(options.pearDir);
    const provider = options.llmProvider ?? new AnthropicProvider(options.anthropic);
    
    this.sessionManager = new SessionManager(storage);
    this.lockManager = new LockManager(lockStorage);
    this.llmOrchestrator = new LLMOrchestrator(provider);
    this.contextManager = new ContextManager(options.context);
    this.projectAnalyzer = new ProjectAnalyzer();
  }
  
  // === Session Lifecycle ===
  
  async createSession(options: CreateSessionOptions): Promise<PearSession> {
    // Acquire lock first
    const lockResult = await this.lockManager.acquireWriteLock(
      options.featurePath,
      options.sessionId ?? generateId(),
      { feature: options.featureName, user: options.user }
    );
    
    if (!lockResult.success) {
      throw new FolderLockedError(options.featurePath, lockResult.holder);
    }
    
    const session = await this.sessionManager.createSession(options);
    return new PearSession(session, this);
  }
  
  async resumeSession(sessionId: string): Promise<PearSession | null> {
    const session = await this.sessionManager.loadSession(sessionId);
    if (!session) return null;
    return new PearSession(session, this);
  }
  
  async listIncompleteSessions(): Promise<SessionInfo[]> {
    return this.sessionManager.listIncompleteSessions();
  }
  
  // === Project Analysis ===
  
  async analyzeProject(rootPath: string): Promise<ProjectMap> {
    return this.projectAnalyzer.analyze(rootPath);
  }
  
  // === Lock Management ===
  
  async getLocks(): Promise<FolderLock[]> {
    return this.lockManager.listLocks();
  }
  
  async releaseLock(path: string, sessionId: string): Promise<void> {
    return this.lockManager.releaseLock(path, sessionId);
  }
}

// Session wrapper with workflow methods
export class PearSession {
  private readonly session: Session;
  private readonly engine: PearEngine;
  private readonly stateMachine: WorkflowStateMachine;
  
  constructor(session: Session, engine: PearEngine) {
    this.session = session;
    this.engine = engine;
    this.stateMachine = new WorkflowStateMachine(session.state);
  }
  
  // === State Access ===
  
  getState(): Readonly<WorkflowState> {
    return this.stateMachine.getState();
  }
  
  getCurrentPhase(): Phase {
    return this.stateMachine.getCurrentPhase();
  }
  
  getAvailableActions(): WorkflowAction[] {
    // Return actions valid for current state
  }
  
  // === Workflow Actions (return AsyncIterable for streaming) ===
  
  async *runPlanning(description: string): AsyncIterable<WorkflowEvent> {
    yield { type: 'phase_started', phase: 'planning' };
    
    const context = this.engine.contextManager.buildContext(
      this.stateMachine.getState(),
      await this.loadProjectFiles()
    );
    
    for await (const chunk of this.engine.llmOrchestrator.planFeature(description, context)) {
      yield { type: 'llm_chunk', content: chunk };
    }
    
    yield { type: 'awaiting_approval', phase: 'planning' };
  }
  
  async *runInterface(): AsyncIterable<WorkflowEvent>;
  async *runImplementation(unitId: string): AsyncIterable<WorkflowEvent>;
  async *runTesting(): AsyncIterable<WorkflowEvent>;
  async *debugFailure(failure: TestFailure): AsyncIterable<WorkflowEvent>;
  
  // === Approvals ===
  
  async approvePlanning(designDoc: string): Promise<void> {
    this.stateMachine.updatePlanningState({
      designDocContent: designDoc,
      approved: true,
    });
    this.stateMachine.transition('approve_planning');
    await this.checkpoint();
  }
  
  async approveInterface(types: string, tests: string): Promise<void>;
  async approveUnit(unitId: string, implementation: string): Promise<void>;
  async approveTesting(): Promise<void>;
  
  // === Navigation ===
  
  async goBackToPlanning(): Promise<void> {
    this.stateMachine.transition('go_back_to_planning');
    await this.checkpoint();
  }
  
  async goBackToInterface(): Promise<void> {
    this.stateMachine.transition('go_back_to_interface');
    await this.checkpoint();
  }
  
  // === Lifecycle ===
  
  async complete(): Promise<void> {
    this.stateMachine.transition('complete');
    await this.engine.lockManager.releaseLock(
      this.session.state.featurePath,
      this.session.sessionId
    );
    await this.engine.sessionManager.completeSession(this.session.sessionId);
  }
  
  async cancel(): Promise<void> {
    await this.engine.lockManager.releaseAllForSession(this.session.sessionId);
    // Keep files, just release locks and mark as cancelled
  }
  
  // === Internal ===
  
  private async checkpoint(): Promise<void> {
    await this.engine.sessionManager.checkpoint(
      this.session.sessionId,
      this.stateMachine.getState()
    );
  }
}

// Events emitted during workflow execution
export type WorkflowEvent =
  | { type: 'phase_started'; phase: Phase }
  | { type: 'llm_chunk'; content: string }
  | { type: 'llm_complete' }
  | { type: 'artifact_created'; path: string }
  | { type: 'awaiting_approval'; phase: Phase }
  | { type: 'error'; error: Error };
```

### Configuration Options

```typescript
export interface PearEngineOptions {
  // Required
  pearDir: string;              // Path to .pear directory
  
  // LLM configuration
  anthropic?: {
    apiKey: string;
    model?: string;
  };
  
  // Optional overrides
  llmProvider?: LLMProvider;    // Custom provider
  sessionStorage?: SessionStorage;
  lockStorage?: LockStorage;
  
  // Context settings
  context?: {
    maxTokens?: number;
  };
}

export interface CreateSessionOptions {
  featureName: string;
  featurePath: string;
  user?: string;
  sessionId?: string;          // Auto-generated if not provided
}
```

---

## Data Models

Full type definitions extracted to `src/types/`:

```typescript
// src/types/workflow.ts

export type Phase = 'planning' | 'interface' | 'implementation' | 'testing' | 'complete';
export type PhaseStatus = 'pending' | 'in_progress' | 'awaiting_approval' | 'approved';

export interface WorkflowState {
  sessionId: string;
  featureName: string;
  featurePath: string;
  createdAt: Date;
  lastActivityAt: Date;
  currentPhase: Phase;
  phaseStatus: PhaseStatus;
  
  planning: PlanningState;
  interface: InterfaceState;
  implementation: ImplementationState;
  testing: TestingState;
  complete: CompleteState;
  
  messages: Message[];
}

export interface PlanningState {
  designDocContent: string | null;
  designDocPath: string | null;
  approved: boolean;
}

export interface InterfaceState {
  typesContent: string | null;
  typesPath: string | null;
  testsContent: string | null;
  testsPath: string | null;
  approved: boolean;
}

export interface ImplementationState {
  units: ImplementationUnit[];
  currentUnitIndex: number;
  dependencyOrder: string[];
}

export interface ImplementationUnit {
  id: string;
  name: string;
  description: string;
  filePath: string;
  status: UnitStatus;
  implementation: string | null;
  dependencies: string[];
}

export type UnitStatus = 'pending' | 'in_progress' | 'awaiting_review' | 'approved';

export interface TestingState {
  lastTestRun: TestRunResult | null;
  debugIterations: number;
  manualTestsComplete: boolean;
  evidenceDocPath: string | null;
}

export interface TestRunResult {
  passed: number;
  failed: number;
  failures: TestFailure[];
  timestamp: Date;
}

export interface TestFailure {
  testName: string;
  expected: string;
  received: string;
  errorMessage: string;
  filePath?: string;
  lineNumber?: number;
}

export interface CompleteState {
  completedAt: Date | null;
  lockReleased: boolean;
  archived: boolean;
}

export interface Message {
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
  phase: Phase;
}
```

---

## Testing Strategy

### Unit Tests

Each module has isolated unit tests:

```typescript
// tests/workflow/state-machine.test.ts
describe('WorkflowStateMachine', () => {
  describe('transitions', () => {
    it('should start in planning phase', () => {
      const sm = new WorkflowStateMachine(createInitialState());
      expect(sm.getCurrentPhase()).toBe('planning');
    });
    
    it('should transition from planning to interface on approval', () => {
      const sm = new WorkflowStateMachine(createInitialState());
      sm.updatePlanningState({ approved: true, designDocContent: '...' });
      sm.transition('approve_planning');
      expect(sm.getCurrentPhase()).toBe('interface');
    });
    
    it('should not allow skipping to implementation', () => {
      const sm = new WorkflowStateMachine(createInitialState());
      expect(() => sm.transition('approve_interface')).toThrow(InvalidTransitionError);
    });
    
    it('should allow going back to planning from interface', () => {
      const sm = createStateMachineAtPhase('interface');
      sm.transition('go_back_to_planning');
      expect(sm.getCurrentPhase()).toBe('planning');
    });
  });
});

// tests/llm/orchestrator.test.ts
describe('LLMOrchestrator', () => {
  let orchestrator: LLMOrchestrator;
  let mockProvider: MockProvider;
  
  beforeEach(() => {
    mockProvider = new MockProvider();
    orchestrator = new LLMOrchestrator(mockProvider);
  });
  
  it('should stream planning response', async () => {
    mockProvider.setResponse('design', '# Feature: Test\n\n## Problem...');
    
    const chunks: string[] = [];
    for await (const chunk of orchestrator.planFeature('Add auth', mockContext)) {
      chunks.push(chunk);
    }
    
    expect(chunks.join('')).toContain('# Feature');
  });
});

// tests/session/manager.test.ts
describe('SessionManager', () => {
  let manager: SessionManager;
  let storage: MemorySessionStorage;
  
  beforeEach(() => {
    storage = new MemorySessionStorage();
    manager = new SessionManager(storage);
  });
  
  it('should create and persist session', async () => {
    const session = await manager.createSession({
      featureName: 'Auth',
      featurePath: 'src/features/auth',
    });
    
    expect(session.sessionId).toBeDefined();
    
    const loaded = await manager.loadSession(session.sessionId);
    expect(loaded).not.toBeNull();
    expect(loaded!.state.featureName).toBe('Auth');
  });
});
```

### Integration Tests

```typescript
// tests/integration/full-workflow.test.ts
describe('Full Workflow Integration', () => {
  let engine: PearEngine;
  let tempDir: string;
  
  beforeEach(async () => {
    tempDir = await createTempDir();
    engine = new PearEngine({
      pearDir: join(tempDir, '.pear'),
      llmProvider: new MockProvider(), // Canned responses
    });
  });
  
  afterEach(async () => {
    await cleanupTempDir(tempDir);
  });
  
  it('should complete all phases', async () => {
    // Create session
    const session = await engine.createSession({
      featureName: 'Test Feature',
      featurePath: join(tempDir, 'src/features/test'),
    });
    
    // Planning
    const planningEvents = await collectEvents(session.runPlanning('Add test feature'));
    expect(planningEvents).toContainEqual({ type: 'awaiting_approval', phase: 'planning' });
    await session.approvePlanning('# Design doc content');
    
    // Interface
    const interfaceEvents = await collectEvents(session.runInterface());
    expect(interfaceEvents).toContainEqual({ type: 'awaiting_approval', phase: 'interface' });
    await session.approveInterface('types content', 'tests content');
    
    // Implementation (mock has one unit)
    const implEvents = await collectEvents(session.runImplementation('unit-1'));
    expect(implEvents).toContainEqual({ type: 'awaiting_approval', phase: 'implementation' });
    await session.approveUnit('unit-1', 'implementation content');
    
    // Testing
    const testEvents = await collectEvents(session.runTesting());
    await session.approveTesting();
    
    // Complete
    await session.complete();
    
    expect(session.getCurrentPhase()).toBe('complete');
  });
});
```

### Test Coverage Requirements

| Module | Target Coverage |
|--------|----------------|
| workflow/ | 95% |
| context/ | 90% |
| llm/ | 85% (mock provider) |
| session/ | 95% |
| lock/ | 95% |
| project/ | 80% |

---

## Implementation Plan

### Week 1: Core Infrastructure

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | Project setup, types, error handling | TypeScript project with strict mode |
| 2 | WorkflowStateMachine | Tested state machine |
| 3 | SessionManager + FileStorage | Sessions persist correctly |
| 4 | LLMProvider abstraction + AnthropicProvider | Can call Claude |
| 5 | MockProvider + LLMOrchestrator | Testable orchestration |

### Week 2: Context and Workflow

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | ContextManager + ContextBuilder | Context windows work |
| 2 | Prompts for all phases | Refined prompts from Phase 1 |
| 3 | PearEngine public API | Clean public interface |
| 4 | PearSession workflow methods | Streaming events work |
| 5 | Integration tests | Full workflow passes |

### Week 3: Polish and Extensions

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | LockManager | Locking works locally |
| 2 | ProjectAnalyzer | Auto-detect project type |
| 3 | Error handling improvements | Helpful error messages |
| 4 | Documentation | README, API docs |
| 5 | CLI update to use library | CLI uses @pear/core |

---

## Migration from Phase 1

### Code to Extract

| Phase 1 Location | Phase 2 Location | Changes Needed |
|------------------|------------------|----------------|
| `src/state/` | `src/session/` | Add storage abstraction |
| `src/workflow/` | `src/workflow/` | Clean up, add transitions |
| `src/llm/` | `src/llm/` | Add provider abstraction |
| `src/llm/prompts.ts` | `src/llm/prompts/` | Split by phase |
| N/A | `src/context/` | New module |
| N/A | `src/lock/` | New module |
| N/A | `src/project/` | New module |

### CLI Update

After Phase 2, update the CLI to use `@pear/core`:

```typescript
// pear-cli/src/index.ts (updated)

import { PearEngine, PearSession } from '@pear/core';

const engine = new PearEngine({
  pearDir: '.pear',
  anthropic: { apiKey: process.env.ANTHROPIC_API_KEY },
});

// CLI now just handles display and prompts
// All workflow logic is in the library
```

---

## Exit Criteria

Phase 2 is complete when:

1. **Library published**: `@pear/core` package ready for use
2. **Tests passing**: >90% coverage, all integration tests pass
3. **CLI migrated**: pear-cli uses the library
4. **Documented**: API documentation complete
5. **Abstractions work**: Can swap providers/storage without code changes

### Deliverables

- [ ] `@pear/core` npm package (local for now)
- [ ] Full test suite
- [ ] API documentation
- [ ] Updated CLI using the library
- [ ] Migration guide from Phase 1

---

## Appendix: Package Dependencies

```json
{
  "name": "@pear/core",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "dependencies": {
    "@anthropic-ai/sdk": "^0.24.0",
    "yaml": "^2.4.0",
    "uuid": "^9.0.0",
    "tiktoken": "^1.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/uuid": "^9.0.0",
    "typescript": "^5.4.0",
    "vitest": "^1.4.0",
    "tsx": "^4.7.0"
  },
  "peerDependencies": {}
}
```

