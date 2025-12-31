# Error Handling

**Document**: `12-error-handling.md`  
**Purpose**: Error types, recovery strategies, and user experience

---

## Overview

Robust error handling is critical for a CLI tool that manages long-running sessions. Users must never lose work due to errors, and recovery paths must be clear.

---

## Error Taxonomy

### Error Base Class

```typescript
// src/utils/errors.ts

/**
 * Base class for all Pear errors
 */
export class PearError extends Error {
  constructor(
    message: string,
    public readonly code: ErrorCode,
    public readonly recoverable: boolean = true,
    public readonly context?: Record<string, unknown>
  ) {
    super(message);
    this.name = 'PearError';
    
    // Capture stack trace
    Error.captureStackTrace(this, this.constructor);
  }
}

type ErrorCode = 
  // Session errors
  | 'SESSION_NOT_FOUND'
  | 'SESSION_CORRUPTED'
  | 'SESSION_LOCKED'
  
  // Workflow errors
  | 'PHASE_TRANSITION_INVALID'
  | 'APPROVAL_REQUIRED'
  | 'STATE_INVALID'
  
  // LLM errors
  | 'LLM_CONNECTION_FAILED'
  | 'LLM_RATE_LIMITED'
  | 'LLM_RESPONSE_INVALID'
  | 'LLM_CONTEXT_TOO_LONG'
  
  // File errors
  | 'FILE_READ_FAILED'
  | 'FILE_WRITE_FAILED'
  | 'FILE_NOT_FOUND'
  | 'ARTIFACT_CONFLICT'
  
  // Test errors
  | 'TEST_RUNNER_FAILED'
  | 'TEST_PARSE_FAILED'
  
  // Other
  | 'PARSE_ERROR'
  | 'CHECKPOINT_FAILED'
  | 'UNKNOWN';
```

### Specific Error Classes

```typescript
/**
 * Session-related errors
 */
export class SessionError extends PearError {
  constructor(
    message: string, 
    code: 'SESSION_NOT_FOUND' | 'SESSION_CORRUPTED' | 'SESSION_LOCKED',
    public readonly sessionId?: string
  ) {
    super(message, code, code !== 'SESSION_CORRUPTED');
    this.name = 'SessionError';
  }
}

/**
 * Workflow/phase transition errors
 */
export class WorkflowError extends PearError {
  constructor(
    message: string, 
    code: 'PHASE_TRANSITION_INVALID' | 'APPROVAL_REQUIRED' | 'STATE_INVALID',
    public readonly currentPhase?: Phase,
    public readonly targetPhase?: Phase
  ) {
    super(message, code, false, { currentPhase, targetPhase });
    this.name = 'WorkflowError';
  }
}

/**
 * LLM/API errors
 */
export class LLMError extends PearError {
  constructor(
    message: string, 
    code: 'LLM_CONNECTION_FAILED' | 'LLM_RATE_LIMITED' | 'LLM_RESPONSE_INVALID' | 'LLM_CONTEXT_TOO_LONG',
    public readonly retryable: boolean = true,
    public readonly retryAfter?: number // seconds
  ) {
    super(message, code, true, { retryable, retryAfter });
    this.name = 'LLMError';
  }
}

/**
 * File system errors
 */
export class FileError extends PearError {
  constructor(
    message: string,
    code: 'FILE_READ_FAILED' | 'FILE_WRITE_FAILED' | 'FILE_NOT_FOUND' | 'ARTIFACT_CONFLICT',
    public readonly filePath: string
  ) {
    super(message, code, true, { filePath });
    this.name = 'FileError';
  }
}

/**
 * Test runner errors
 */
export class TestError extends PearError {
  constructor(
    message: string,
    code: 'TEST_RUNNER_FAILED' | 'TEST_PARSE_FAILED',
    public readonly rawOutput?: string
  ) {
    super(message, code, true, { rawOutput: rawOutput?.substring(0, 500) });
    this.name = 'TestError';
  }
}
```

---

## Recovery Strategies

### Error Recovery Matrix

| Error Code | Recoverable | Strategy | User Action |
|------------|-------------|----------|-------------|
| `SESSION_NOT_FOUND` | Yes | List available sessions | Select different session |
| `SESSION_CORRUPTED` | Maybe | Restore from backup | Confirm restore |
| `PHASE_TRANSITION_INVALID` | No | Show valid transitions | Choose valid action |
| `LLM_CONNECTION_FAILED` | Yes | Retry with backoff | Wait or quit |
| `LLM_RATE_LIMITED` | Yes | Wait and retry | Wait (automatic) |
| `LLM_RESPONSE_INVALID` | Yes | Retry same request | Automatic retry |
| `FILE_WRITE_FAILED` | Yes | Show error, offer retry | Fix permissions |
| `ARTIFACT_CONFLICT` | Yes | Show diff | Choose overwrite/skip |
| `TEST_RUNNER_FAILED` | Yes | Show raw output | Manual investigation |
| `CHECKPOINT_FAILED` | Yes | Warn, continue | Acknowledge |

### Retry Logic

```typescript
// src/utils/retry.ts

interface RetryOptions {
  maxRetries: number;
  initialDelayMs: number;
  maxDelayMs: number;
  retryOn: ErrorCode[];
}

const DEFAULT_RETRY_OPTIONS: RetryOptions = {
  maxRetries: 3,
  initialDelayMs: 1000,
  maxDelayMs: 30000,
  retryOn: ['LLM_CONNECTION_FAILED', 'LLM_RATE_LIMITED'],
};

async function withRetry<T>(
  operation: () => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const opts = { ...DEFAULT_RETRY_OPTIONS, ...options };
  let lastError: PearError;
  
  for (let attempt = 0; attempt < opts.maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (!(error instanceof PearError)) {
        throw error;
      }
      
      if (!opts.retryOn.includes(error.code)) {
        throw error;
      }
      
      lastError = error;
      
      // Calculate delay with exponential backoff
      const delay = Math.min(
        opts.initialDelayMs * Math.pow(2, attempt),
        opts.maxDelayMs
      );
      
      // Special handling for rate limits
      if (error.code === 'LLM_RATE_LIMITED' && error instanceof LLMError) {
        const retryAfter = error.retryAfter || delay / 1000;
        console.log(chalk.yellow(`Rate limited. Waiting ${retryAfter}s...`));
        await sleep(retryAfter * 1000);
      } else {
        console.log(chalk.yellow(`Retrying in ${delay}ms... (${attempt + 1}/${opts.maxRetries})`));
        await sleep(delay);
      }
    }
  }
  
  throw lastError!;
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

---

## Error Display

### CLI Error Display

```typescript
// src/cli/display.ts

function displayError(error: PearError): void {
  console.log();
  console.log(chalk.red('━'.repeat(60)));
  console.log(chalk.red.bold(`❌ ${error.name}: ${error.code}`));
  console.log(chalk.red('━'.repeat(60)));
  console.log();
  console.log(chalk.red(error.message));
  
  // Show context if available
  if (error.context && Object.keys(error.context).length > 0) {
    console.log();
    console.log(chalk.gray('Details:'));
    for (const [key, value] of Object.entries(error.context)) {
      if (value !== undefined) {
        console.log(chalk.gray(`  ${key}: ${JSON.stringify(value)}`));
      }
    }
  }
  
  // Show recovery options
  console.log();
  if (error.recoverable) {
    console.log(chalk.yellow('💡 This error may be recoverable:'));
    console.log(chalk.yellow(getRecoveryHint(error)));
  } else {
    console.log(chalk.red('⚠️  This error requires manual intervention.'));
  }
  
  console.log();
}

function getRecoveryHint(error: PearError): string {
  switch (error.code) {
    case 'SESSION_NOT_FOUND':
      return '  Use "pear list" to see available sessions';
    case 'SESSION_CORRUPTED':
      return '  Run "pear resume" to attempt recovery from backup';
    case 'LLM_CONNECTION_FAILED':
      return '  Check your internet connection and ANTHROPIC_API_KEY';
    case 'LLM_RATE_LIMITED':
      return '  Wait a moment and try again';
    case 'FILE_WRITE_FAILED':
      return '  Check file permissions and disk space';
    case 'ARTIFACT_CONFLICT':
      return '  Choose to overwrite or skip the conflicting file';
    case 'TEST_RUNNER_FAILED':
      return '  Check that vitest is installed and configured';
    default:
      return '  Try the operation again or use "pear resume"';
  }
}
```

### Error Logging

```typescript
// src/utils/logger.ts

const DEBUG = process.env.DEBUG?.includes('pear');

function logError(error: Error, context?: Record<string, unknown>): void {
  const timestamp = new Date().toISOString();
  const logEntry = {
    timestamp,
    name: error.name,
    message: error.message,
    code: error instanceof PearError ? error.code : 'UNKNOWN',
    context: {
      ...context,
      ...(error instanceof PearError ? error.context : {}),
    },
    stack: error.stack,
  };
  
  if (DEBUG) {
    console.error(chalk.gray('─'.repeat(60)));
    console.error(chalk.gray(`[DEBUG] ${timestamp}`));
    console.error(chalk.gray(JSON.stringify(logEntry, null, 2)));
    console.error(chalk.gray('─'.repeat(60)));
  }
  
  // Could also write to file:
  // await appendFile('.pear/error.log', JSON.stringify(logEntry) + '\n');
}
```

---

## Handling Specific Scenarios

### Session Corruption

```typescript
async function handleCorruptedSession(
  sessionId: string,
  manager: StateManager,
  cli: CLIHandler
): Promise<WorkflowState | null> {
  cli.displayError(new SessionError(
    `Session ${sessionId} appears to be corrupted`,
    'SESSION_CORRUPTED',
    sessionId
  ));
  
  // Try backup
  const backup = await manager.restoreFromBackup(sessionId);
  
  if (backup) {
    const confirm = await cli.promptForConfirmation(
      `Found backup from ${backup.lastActivityAt}. Restore?`
    );
    
    if (confirm) {
      await manager.saveCheckpoint(backup);
      console.log(chalk.green('✓ Session restored from backup'));
      return backup;
    }
  }
  
  // No backup or user declined
  const abandon = await cli.promptForConfirmation(
    'Unable to recover. Abandon this session?'
  );
  
  if (abandon) {
    // Archive as failed
    await manager.archiveSession(sessionId);
    console.log(chalk.yellow('Session archived as unrecoverable'));
  }
  
  return null;
}
```

### LLM Response Invalid

```typescript
async function handleInvalidLLMResponse(
  response: string,
  phase: Phase,
  cli: CLIHandler
): Promise<'retry' | 'manual' | 'skip'> {
  console.log(chalk.yellow('\n⚠️  The LLM response could not be parsed correctly.'));
  console.log(chalk.gray('This sometimes happens with complex outputs.\n'));
  
  console.log(chalk.gray('─── Raw Response ───'));
  console.log(chalk.gray(response.substring(0, 500)));
  if (response.length > 500) {
    console.log(chalk.gray('... (truncated)'));
  }
  console.log(chalk.gray('───────────────────\n'));
  
  const action = await cli.promptForAction([
    { type: 'retry' as const },
    { type: 'manual' as const, label: 'Enter content manually' },
    { type: 'skip' as const, label: 'Skip this step' },
  ]);
  
  return action.type as 'retry' | 'manual' | 'skip';
}
```

### Artifact Conflict

```typescript
async function handleArtifactConflict(
  artifact: Artifact,
  existingContent: string,
  cli: CLIHandler
): Promise<'overwrite' | 'skip' | 'diff'> {
  console.log(chalk.yellow(`\n⚠️  File already exists: ${artifact.path}`));
  
  // Show brief diff
  const diffPreview = generateDiffPreview(existingContent, artifact.content);
  console.log(chalk.gray('\n─── Difference Preview ───'));
  console.log(diffPreview);
  console.log(chalk.gray('──────────────────────────\n'));
  
  const action = await cli.promptForAction([
    { type: 'overwrite' as const, label: 'Overwrite with new content' },
    { type: 'diff' as const, label: 'Show full diff' },
    { type: 'skip' as const, label: 'Skip this file' },
  ]);
  
  if (action.type === 'diff') {
    console.log(generateFullDiff(existingContent, artifact.content));
    return handleArtifactConflict(artifact, existingContent, cli);
  }
  
  return action.type as 'overwrite' | 'skip';
}
```

### Test Runner Failure

```typescript
async function handleTestRunnerFailure(
  error: TestError,
  cli: CLIHandler
): Promise<'retry' | 'skip' | 'manual'> {
  console.log(chalk.red('\n❌ Test runner failed to execute'));
  console.log(chalk.gray(`Error: ${error.message}\n`));
  
  if (error.rawOutput) {
    console.log(chalk.gray('─── Raw Output ───'));
    console.log(chalk.gray(error.rawOutput.substring(0, 1000)));
    console.log(chalk.gray('──────────────────\n'));
  }
  
  console.log(chalk.yellow('Common causes:'));
  console.log(chalk.yellow('  • vitest not installed (run: npm install -D vitest)'));
  console.log(chalk.yellow('  • Missing vitest.config.ts'));
  console.log(chalk.yellow('  • TypeScript compilation errors in test files'));
  console.log();
  
  const action = await cli.promptForAction([
    { type: 'retry' as const },
    { type: 'manual' as const, label: 'Run tests manually and continue' },
    { type: 'skip' as const, label: 'Skip automated tests' },
  ]);
  
  return action.type as 'retry' | 'skip' | 'manual';
}
```

---

## Checkpoint Protection

### Safe State Updates

```typescript
async function safeUpdateState(
  manager: StateManager,
  sessionId: string,
  updates: Partial<WorkflowState>
): Promise<WorkflowState> {
  try {
    const state = manager.updateSession(sessionId, updates);
    await manager.saveCheckpoint(state);
    return state;
  } catch (error) {
    // Checkpoint failed, warn user
    console.log(chalk.yellow('\n⚠️  Warning: Could not save checkpoint'));
    console.log(chalk.yellow('Your progress may not be saved if the CLI crashes.\n'));
    
    // Still return the updated state (in memory)
    return manager.getSession(sessionId)!;
  }
}
```

### Crash Recovery on Startup

```typescript
async function checkForCrashedSessions(
  manager: StateManager,
  cli: CLIHandler
): Promise<void> {
  const sessions = await manager.listIncompleteSessions();
  
  const crashed = sessions.filter(s => s.status === 'interrupted');
  
  if (crashed.length === 0) return;
  
  console.log(chalk.yellow(`\n⚠️  Found ${crashed.length} interrupted session(s):\n`));
  
  for (const session of crashed) {
    console.log(chalk.yellow(`  • ${session.featureName}`));
    console.log(chalk.gray(`    Last active: ${session.lastActivity}`));
    console.log(chalk.gray(`    Phase: ${session.currentPhase}`));
    console.log();
  }
  
  const resume = await cli.promptForConfirmation(
    'Would you like to resume an interrupted session?'
  );
  
  if (resume) {
    // Launch session selection
  }
}
```

---

## Graceful Shutdown

```typescript
// src/index.ts

function setupGracefulShutdown(controller: WorkflowController): void {
  const shutdown = async (signal: string) => {
    console.log(chalk.yellow(`\n\nReceived ${signal}. Saving progress...`));
    
    try {
      const state = controller.getCurrentState();
      if (state) {
        await controller.saveCheckpoint();
        console.log(chalk.green('✓ Progress saved'));
        console.log(chalk.gray(`  Resume with: pear resume ${state.sessionId}`));
      }
    } catch (error) {
      console.log(chalk.red('⚠️  Could not save progress'));
    }
    
    process.exit(0);
  };
  
  process.on('SIGINT', () => shutdown('SIGINT'));
  process.on('SIGTERM', () => shutdown('SIGTERM'));
  
  // Handle uncaught errors
  process.on('uncaughtException', async (error) => {
    console.error(chalk.red('\n\n💥 Unexpected error:'));
    console.error(error);
    
    try {
      await controller.saveCheckpoint();
      console.log(chalk.yellow('\n✓ Emergency checkpoint saved'));
    } catch {
      console.log(chalk.red('Could not save emergency checkpoint'));
    }
    
    process.exit(1);
  });
}
```

---

## Testing Error Handling

```typescript
// tests/utils/errors.test.ts

describe('Error Handling', () => {
  describe('withRetry', () => {
    it('should retry on retryable errors', async () => {
      let attempts = 0;
      
      const operation = async () => {
        attempts++;
        if (attempts < 3) {
          throw new LLMError('Connection failed', 'LLM_CONNECTION_FAILED');
        }
        return 'success';
      };
      
      const result = await withRetry(operation);
      
      expect(result).toBe('success');
      expect(attempts).toBe(3);
    });
    
    it('should not retry non-retryable errors', async () => {
      const operation = async () => {
        throw new WorkflowError('Invalid transition', 'PHASE_TRANSITION_INVALID');
      };
      
      await expect(withRetry(operation)).rejects.toThrow('Invalid transition');
    });
  });
});
```

---

## Related Documents

- [02-components.md](./02-components.md) — Where error handling is implemented
- [08-user-interface.md](./08-user-interface.md) — Error display UI
- [11-testing-strategy.md](./11-testing-strategy.md) — Testing error scenarios

