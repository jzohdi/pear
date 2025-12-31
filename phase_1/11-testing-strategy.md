# Testing Strategy

**Document**: `11-testing-strategy.md`  
**Purpose**: Unit, integration, and manual testing approach

---

## Overview

Phase 1 testing focuses on validating that the core workflow works correctly. We use three levels of testing:

1. **Unit Tests**: Individual component behavior
2. **Integration Tests**: Component interactions
3. **Manual Tests**: End-to-end workflow validation

---

## Testing Framework

**Framework**: Vitest  
**Why**: Fast, TypeScript-native, great DX

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    include: ['tests/**/*.test.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      include: ['src/**/*.ts'],
      exclude: ['src/types/**', 'src/index.ts'],
    },
  },
});
```

---

## Unit Tests

### State Manager

```typescript
// tests/state/manager.test.ts

describe('StateManager', () => {
  let manager: StateManager;
  let tempDir: string;
  
  beforeEach(async () => {
    tempDir = await fs.mkdtemp(path.join(os.tmpdir(), 'pear-test-'));
    manager = new StateManager(tempDir);
  });
  
  afterEach(async () => {
    await fs.rm(tempDir, { recursive: true });
  });
  
  describe('createSession', () => {
    it('should create session with unique ID', () => {
      const session = manager.createSession({ featureName: 'Test' });
      
      expect(session.sessionId).toBeDefined();
      expect(session.sessionId).toMatch(/^[a-f0-9-]{36}$/); // UUID format
    });
    
    it('should initialize with planning phase', () => {
      const session = manager.createSession({ featureName: 'Test' });
      
      expect(session.currentPhase).toBe('planning');
      expect(session.phaseStatus).toBe('in_progress');
    });
    
    it('should set timestamps', () => {
      const before = new Date();
      const session = manager.createSession({ featureName: 'Test' });
      const after = new Date();
      
      expect(new Date(session.createdAt).getTime()).toBeGreaterThanOrEqual(before.getTime());
      expect(new Date(session.createdAt).getTime()).toBeLessThanOrEqual(after.getTime());
    });
  });
  
  describe('saveCheckpoint', () => {
    it('should persist session to JSON', async () => {
      const session = manager.createSession({ featureName: 'Test' });
      await manager.saveCheckpoint(session);
      
      const filePath = path.join(tempDir, 'sessions', `${session.sessionId}.json`);
      const exists = await fs.access(filePath).then(() => true).catch(() => false);
      
      expect(exists).toBe(true);
    });
    
    it('should create backup before saving', async () => {
      const session = manager.createSession({ featureName: 'Test' });
      await manager.saveCheckpoint(session);
      
      // Modify and save again
      session.featureName = 'Test Modified';
      await manager.saveCheckpoint(session);
      
      const backupPath = path.join(tempDir, 'sessions', `${session.sessionId}.backup.json`);
      const backupExists = await fs.access(backupPath).then(() => true).catch(() => false);
      
      expect(backupExists).toBe(true);
    });
  });
  
  describe('loadCheckpoint', () => {
    it('should load saved session', async () => {
      const session = manager.createSession({ featureName: 'Test' });
      session.planning.designDocContent = 'Test content';
      await manager.saveCheckpoint(session);
      
      const loaded = await manager.loadCheckpoint(session.sessionId);
      
      expect(loaded).not.toBeNull();
      expect(loaded!.featureName).toBe('Test');
      expect(loaded!.planning.designDocContent).toBe('Test content');
    });
    
    it('should return null for non-existent session', async () => {
      const loaded = await manager.loadCheckpoint('non-existent');
      
      expect(loaded).toBeNull();
    });
  });
  
  describe('restoreFromBackup', () => {
    it('should restore from backup when primary is corrupted', async () => {
      const session = manager.createSession({ featureName: 'Original' });
      await manager.saveCheckpoint(session);
      
      // Save again to create backup
      session.featureName = 'Modified';
      await manager.saveCheckpoint(session);
      
      // Corrupt primary file
      const primaryPath = path.join(tempDir, 'sessions', `${session.sessionId}.json`);
      await fs.writeFile(primaryPath, 'not valid json');
      
      // Restore
      const restored = await manager.restoreFromBackup(session.sessionId);
      
      expect(restored).not.toBeNull();
      expect(restored!.featureName).toBe('Original'); // From backup
    });
  });
  
  describe('listIncompleteSessions', () => {
    it('should list sessions not in done folder', async () => {
      const session1 = manager.createSession({ featureName: 'Active' });
      await manager.saveCheckpoint(session1);
      
      const list = await manager.listIncompleteSessions();
      
      expect(list).toHaveLength(1);
      expect(list[0].featureName).toBe('Active');
    });
  });
});
```

### Response Parser

```typescript
// tests/llm/parser.test.ts

describe('ResponseParser', () => {
  describe('parseCodeBlocks', () => {
    it('should extract code block with language', () => {
      const response = `
Here's the code:

\`\`\`typescript
const x = 1;
\`\`\`
      `;
      
      const blocks = parseCodeBlocks(response);
      
      expect(blocks).toHaveLength(1);
      expect(blocks[0].language).toBe('typescript');
      expect(blocks[0].content).toBe('const x = 1;');
    });
    
    it('should extract file path from comment', () => {
      const response = `
\`\`\`typescript
// src/utils/helper.ts
export function helper() {}
\`\`\`
      `;
      
      const blocks = parseCodeBlocks(response);
      
      expect(blocks[0].filePath).toBe('src/utils/helper.ts');
      expect(blocks[0].content).toBe('export function helper() {}');
    });
    
    it('should handle code block without file path', () => {
      const response = `
\`\`\`typescript
const x = 1;
\`\`\`
      `;
      
      const blocks = parseCodeBlocks(response);
      
      expect(blocks[0].filePath).toBeNull();
    });
    
    it('should handle multiple code blocks', () => {
      const response = `
\`\`\`typescript
// src/a.ts
const a = 1;
\`\`\`

Some text

\`\`\`typescript
// src/b.ts
const b = 2;
\`\`\`
      `;
      
      const blocks = parseCodeBlocks(response);
      
      expect(blocks).toHaveLength(2);
    });
    
    it('should handle empty code block', () => {
      const response = `
\`\`\`typescript
\`\`\`
      `;
      
      const blocks = parseCodeBlocks(response);
      
      expect(blocks[0].content).toBe('');
    });
  });
  
  describe('parseArtifacts', () => {
    it('should create artifact from code block with path', () => {
      const response = `
\`\`\`typescript
// src/test.ts
export const x = 1;
\`\`\`
      `;
      
      const artifacts = parseArtifacts(response);
      
      expect(artifacts).toHaveLength(1);
      expect(artifacts[0].path).toBe('src/test.ts');
      expect(artifacts[0].content).toBe('export const x = 1;');
      expect(artifacts[0].action).toBe('create');
    });
    
    it('should combine multiple blocks for same file', () => {
      const response = `
\`\`\`typescript
// src/test.ts
// Part 1
const a = 1;
\`\`\`

\`\`\`typescript
// src/test.ts
// Part 2
const b = 2;
\`\`\`
      `;
      
      const artifacts = parseArtifacts(response);
      
      expect(artifacts).toHaveLength(1);
      expect(artifacts[0].content).toContain('Part 1');
      expect(artifacts[0].content).toContain('Part 2');
    });
    
    it('should ignore code blocks without file path', () => {
      const response = `
\`\`\`typescript
const x = 1;
\`\`\`
      `;
      
      const artifacts = parseArtifacts(response);
      
      expect(artifacts).toHaveLength(0);
    });
  });
  
  describe('parseSections', () => {
    it('should extract markdown sections', () => {
      const response = `
# Title

Some intro

## Section 1

Content 1

## Section 2

Content 2
      `;
      
      const sections = parseSections(response);
      
      expect(sections).toHaveLength(3);
      expect(sections[0].heading).toBe('Title');
      expect(sections[1].heading).toBe('Section 1');
      expect(sections[1].content).toContain('Content 1');
    });
  });
});
```

### Context Manager

```typescript
// tests/llm/context.test.ts

describe('ContextManager', () => {
  describe('estimateTokens', () => {
    it('should estimate tokens from character count', () => {
      const text = 'Hello world'; // 11 chars
      const tokens = estimateTokens(text);
      
      expect(tokens).toBe(3); // ceil(11/4)
    });
  });
  
  describe('buildContext', () => {
    it('should always include design document', () => {
      const state = createMockState({
        planning: {
          designDocContent: 'Design doc content',
          approved: true,
        },
      });
      
      const context = buildContext(state, 'implementation');
      
      expect(context.designDocument).toBe('Design doc content');
    });
    
    it('should include types for implementation phase', () => {
      const state = createMockState({
        interface: {
          typesContent: 'export interface Test {}',
          approved: true,
        },
      });
      
      const context = buildContext(state, 'implementation');
      
      expect(context.typeDefinitions).toBe('export interface Test {}');
    });
    
    it('should limit message history to token budget', () => {
      const state = createMockState();
      
      // Add many messages
      for (let i = 0; i < 100; i++) {
        state.messages.push({
          role: 'user',
          content: 'A'.repeat(1000), // ~250 tokens each
          timestamp: new Date(),
          phase: 'planning',
        });
      }
      
      const context = buildContext(state, 'planning');
      
      // Should not include all messages
      expect(context.recentMessages.length).toBeLessThan(100);
      expect(context.estimatedTokens).toBeLessThan(100000);
    });
  });
  
  describe('buildMessageWindow', () => {
    it('should always include first message', () => {
      const messages = [
        { role: 'user', content: 'First', timestamp: new Date(), phase: 'planning' },
        { role: 'assistant', content: 'Second', timestamp: new Date(), phase: 'planning' },
        { role: 'user', content: 'Third', timestamp: new Date(), phase: 'planning' },
      ];
      
      const window = buildMessageWindow(messages, 100);
      
      expect(window[0].content).toBe('First');
    });
    
    it('should include recent messages', () => {
      const messages = [];
      for (let i = 0; i < 20; i++) {
        messages.push({
          role: 'user' as const,
          content: `Message ${i}`,
          timestamp: new Date(),
          phase: 'planning' as const,
        });
      }
      
      const window = buildMessageWindow(messages, 50); // Limited tokens
      
      // Should include last messages
      expect(window.some(m => m.content === 'Message 19')).toBe(true);
    });
  });
});
```

### Workflow Controller

```typescript
// tests/workflow/controller.test.ts

describe('WorkflowController', () => {
  let controller: WorkflowController;
  let mockLLMClient: MockLLMClient;
  
  beforeEach(() => {
    mockLLMClient = new MockLLMClient();
    controller = new WorkflowController({
      llmClient: mockLLMClient,
      // ... other mocks
    });
  });
  
  describe('startNewSession', () => {
    it('should create session in planning phase', async () => {
      const session = await controller.startNewSession(
        'Add user auth',
        'src/features/auth'
      );
      
      expect(session.state.currentPhase).toBe('planning');
      expect(session.state.featureName).toBe('Add user auth');
    });
  });
  
  describe('approveCurrentStep', () => {
    it('should transition from planning to interface on approval', async () => {
      await controller.startNewSession('Test', 'src/test');
      controller.getCurrentState().planning.designDocContent = 'Design';
      controller.getCurrentState().planning.designDocPath = 'src/test/DESIGN.md';
      
      await controller.approveCurrentStep();
      
      expect(controller.getCurrentPhase()).toBe('interface');
    });
    
    it('should not skip phases', async () => {
      await controller.startNewSession('Test', 'src/test');
      
      // Try to approve without design doc
      await expect(controller.approveCurrentStep()).rejects.toThrow();
    });
  });
  
  describe('goBackToPhase', () => {
    it('should allow going back to planning from interface', async () => {
      // Setup: get to interface phase
      await controller.startNewSession('Test', 'src/test');
      controller.getCurrentState().planning.designDocContent = 'Design';
      await controller.approveCurrentStep();
      
      expect(controller.getCurrentPhase()).toBe('interface');
      
      // Go back
      const result = await controller.goBackToPhase('planning');
      
      expect(result.targetPhase).toBe('planning');
      expect(controller.getCurrentPhase()).toBe('planning');
    });
    
    it('should preserve implementation when going to interface', async () => {
      // Setup: get to implementation with some approved units
      // ... setup code
      
      const result = await controller.goBackToPhase('interface');
      
      expect(result.preservedState.implementation).toBeDefined();
    });
  });
});
```

---

## Integration Tests

```typescript
// tests/integration/full-workflow.test.ts

describe('Full Workflow Integration', () => {
  let controller: WorkflowController;
  let tempDir: string;
  
  beforeEach(async () => {
    tempDir = await fs.mkdtemp(path.join(os.tmpdir(), 'pear-integration-'));
    
    // Create controller with real components, mocked LLM
    controller = new WorkflowController({
      llmClient: new MockLLMClient(),
      stateManager: new StateManager(path.join(tempDir, '.pear')),
      fileManager: new FileManager(tempDir),
      testRunner: new MockTestRunner(),
    });
  });
  
  afterEach(async () => {
    await fs.rm(tempDir, { recursive: true });
  });
  
  it('should complete all phases for a simple feature', async () => {
    // Start session
    await controller.startNewSession('Add helper', 'src/features/helper');
    
    // Planning
    mockLLMResponses([
      '# Feature: Helper\n\n## Problem\nNeed helpers...',
    ]);
    await controller.runCurrentPhase();
    await controller.approveCurrentStep();
    
    expect(controller.getCurrentPhase()).toBe('interface');
    
    // Interface
    mockLLMResponses([
      '```typescript\n// src/features/helper/types.ts\nexport interface Config {}\n```',
      '```typescript\n// src/features/helper/__tests__/helper.test.ts\ndescribe(...)\n```',
      '## Implementation Order\n1. helper()\n',
    ]);
    await controller.runCurrentPhase(); // types
    await controller.approveCurrentStep();
    await controller.runCurrentPhase(); // tests
    await controller.approveCurrentStep();
    await controller.runCurrentPhase(); // order
    await controller.approveCurrentStep();
    
    expect(controller.getCurrentPhase()).toBe('implementation');
    
    // ... continue through implementation and testing
  });
  
  it('should handle crash recovery', async () => {
    // Start and get partway through
    const session = await controller.startNewSession('Test', 'src/test');
    const sessionId = session.sessionId;
    
    // Simulate crash (create new controller instance)
    const newController = new WorkflowController({
      // ... same config
    });
    
    // Resume
    const resumed = await newController.resumeSession(sessionId);
    
    expect(resumed.state.featureName).toBe('Test');
  });
});
```

---

## Mock Utilities

```typescript
// tests/setup.ts

export class MockLLMClient implements LLMClient {
  private responses: string[] = [];
  private responseIndex = 0;
  
  setResponses(responses: string[]) {
    this.responses = responses;
    this.responseIndex = 0;
  }
  
  async *chat(options: ChatOptions): AsyncIterable<string> {
    const response = this.responses[this.responseIndex++] || 'Mock response';
    yield response;
  }
}

export class MockTestRunner implements TestRunner {
  private results: TestRunResult = {
    passed: 5,
    failed: 0,
    skipped: 0,
    duration: 100,
    results: [],
    failures: [],
    timestamp: new Date(),
  };
  
  setResults(results: Partial<TestRunResult>) {
    this.results = { ...this.results, ...results };
  }
  
  async runTests(): Promise<TestRunResult> {
    return this.results;
  }
}

export function createMockState(overrides?: Partial<WorkflowState>): WorkflowState {
  return {
    sessionId: 'test-session',
    featureName: 'Test Feature',
    featurePath: 'src/features/test',
    createdAt: new Date(),
    lastActivityAt: new Date(),
    currentPhase: 'planning',
    phaseStatus: 'in_progress',
    planning: {
      designDocContent: null,
      designDocPath: null,
      approved: false,
    },
    interface: {
      typesContent: null,
      typesPath: null,
      testsContent: null,
      testsPath: null,
      approved: false,
      dependencyAnalysis: null,
    },
    implementation: {
      units: [],
      currentUnitIndex: 0,
      dependencyOrder: [],
      testImplementations: [],
      currentTestIndex: 0,
      stage: 'production',
    },
    testing: {
      testRuns: [],
      lastTestRun: null,
      debugIterations: 0,
      manualTests: [],
      manualTestsComplete: false,
      testEvidencePath: null,
      testEvidenceGenerated: false,
    },
    messages: [],
    ...overrides,
  };
}
```

---

## Manual Testing Checklist

### Day-by-Day Manual Tests

**Day 2 (State Manager)**:
- [ ] Create session, close CLI, resume session
- [ ] Kill CLI mid-session, verify recovery works

**Day 3 (LLM Client)**:
- [ ] Verify streaming displays in real-time
- [ ] Test with invalid API key (error handling)

**Day 6 (Planning Phase)**:
- [ ] Complete planning start to finish
- [ ] Request changes, verify re-generation
- [ ] Verify DESIGN.md is created

**Day 10 (Test Runner)**:
- [ ] Run tests on a project with Vitest
- [ ] Verify failure details are correct

**Day 14 (Real Feature)**:
- [ ] Build entire feature with Pear
- [ ] Document any issues

---

## Coverage Goals

| Component | Target Coverage |
|-----------|-----------------|
| State Manager | 90% |
| Response Parser | 95% |
| Context Manager | 85% |
| Workflow Controller | 80% |
| File Manager | 80% |
| Overall | 80% |

---

## Related Documents

- [10-implementation-plan.md](./10-implementation-plan.md) — When to write tests
- [12-error-handling.md](./12-error-handling.md) — Error scenarios to test
- [02-components.md](./02-components.md) — What to test

