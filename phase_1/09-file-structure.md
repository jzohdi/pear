# File Structure

**Document**: `09-file-structure.md`  
**Purpose**: Project organization for the CLI application and generated files

---

## CLI Application Structure

```
pear-cli/
├── package.json              # Dependencies and scripts
├── tsconfig.json             # TypeScript configuration
├── vitest.config.ts          # Test configuration
├── .env.example              # Environment variable template
├── .gitignore                # Git ignore rules
│
├── src/
│   ├── index.ts              # Entry point (CLI commands)
│   │
│   ├── types/
│   │   └── index.ts          # All type definitions (from 03-data-models.md)
│   │
│   ├── cli/
│   │   ├── index.ts          # CLI handler main class
│   │   ├── display.ts        # Output formatting (code, markdown)
│   │   ├── prompts.ts        # User input prompts (inquirer)
│   │   └── streaming.ts      # Stream response display
│   │
│   ├── workflow/
│   │   ├── controller.ts     # Main workflow orchestration
│   │   ├── transitions.ts    # Phase transition logic
│   │   └── phases/
│   │       ├── planning.ts   # Planning phase handler
│   │       ├── interface.ts  # Interface phase handler
│   │       ├── implementation.ts  # Implementation phase handler
│   │       ├── testing.ts    # Testing phase handler
│   │       └── completion.ts # Completion handler
│   │
│   ├── state/
│   │   ├── manager.ts        # State management
│   │   ├── storage.ts        # JSON persistence
│   │   └── validation.ts     # State validation
│   │
│   ├── llm/
│   │   ├── client.ts         # Anthropic API wrapper
│   │   ├── prompts.ts        # System prompts for each phase
│   │   ├── parser.ts         # Response parsing (code blocks, artifacts)
│   │   └── context.ts        # Context window management
│   │
│   ├── testing/
│   │   ├── runner.ts         # Test execution
│   │   └── parsers/
│   │       └── vitest.ts     # Vitest JSON output parser
│   │
│   ├── files/
│   │   ├── manager.ts        # File read/write operations
│   │   └── artifacts.ts      # Artifact application
│   │
│   └── utils/
│       ├── errors.ts         # Error types and handling
│       ├── logger.ts         # Logging utilities
│       └── tokens.ts         # Token estimation
│
├── tests/
│   ├── setup.ts              # Test setup and mocks
│   │
│   ├── workflow/
│   │   ├── controller.test.ts
│   │   └── transitions.test.ts
│   │
│   ├── state/
│   │   ├── manager.test.ts
│   │   └── storage.test.ts
│   │
│   ├── llm/
│   │   ├── parser.test.ts
│   │   └── context.test.ts
│   │
│   └── testing/
│       └── runner.test.ts
│
└── .pear/                    # Created at runtime (for CLI development)
    ├── config.json
    └── sessions/
```

---

## Module Descriptions

### `src/index.ts`

Entry point that sets up CLI commands:

```typescript
#!/usr/bin/env node

import { Command } from 'commander';
import { CLIHandler } from './cli';
import { WorkflowController } from './workflow/controller';

const program = new Command();

program
  .name('pear')
  .description('AI pair-programming CLI')
  .version('0.1.0');

program
  .command('new')
  .description('Start a new feature session')
  .action(async () => {
    const cli = new CLIHandler();
    const controller = new WorkflowController();
    // ... implementation
  });

program
  .command('resume')
  .description('Resume an incomplete session')
  .action(async () => {
    // ... implementation
  });

program
  .command('list')
  .description('List all sessions')
  .action(async () => {
    // ... implementation
  });

program.parse();
```

### `src/types/index.ts`

Central type definitions (see [03-data-models.md](./03-data-models.md)):

```typescript
// Re-export all types
export * from './workflow';
export * from './state';
export * from './llm';
export * from './testing';
```

### `src/workflow/controller.ts`

Main orchestration logic:

```typescript
export class WorkflowController {
  private state: WorkflowState | null = null;
  private stateManager: StateManager;
  private llmClient: LLMClient;
  private fileManager: FileManager;
  private testRunner: TestRunner;
  
  async startNewSession(
    description: string, 
    path: string
  ): Promise<Session> {
    // Create initial state
    // Start planning phase
  }
  
  async runCurrentPhase(): Promise<PhaseResult> {
    // Delegate to phase handler
  }
  
  // ... other methods
}
```

### `src/llm/prompts.ts`

System prompts (see [04-llm-prompts.md](./04-llm-prompts.md)):

```typescript
export const PROMPTS = {
  planning: PLANNING_SYSTEM_PROMPT,
  interface: INTERFACE_SYSTEM_PROMPT,
  implementation: IMPLEMENTATION_SYSTEM_PROMPT,
  testing: TESTING_SYSTEM_PROMPT,
};

export const TEMPERATURES: Record<Phase, number> = {
  planning: 0.7,
  interface: 0.3,
  implementation: 0.2,
  testing: 0.3,
  complete: 0.3,
};
```

---

## Generated Files in Target Project

When using the CLI on a project, these files are created:

```
target-project/
├── .pear/
│   ├── config.json           # Project configuration
│   └── sessions/
│       ├── {session-id}.json         # Active session state
│       ├── {session-id}.backup.json  # Previous checkpoint
│       └── done/                     # Archived sessions
│           └── {old-session}.json
│
└── src/features/{feature}/
    ├── DESIGN.md             # Phase 1: Design document
    ├── types.ts              # Phase 2: Type definitions
    ├── index.ts              # Phase 3: Main implementation
    ├── {helpers}.ts          # Phase 3: Additional files as needed
    ├── TEST_EVIDENCE.md      # Phase 4: Test evidence
    └── __tests__/
        └── {feature}.test.ts # Phase 2-3: Tests
```

---

## Configuration Files

### `.pear/config.json`

```json
{
  "version": 1,
  "defaultFeaturePath": "src/features",
  "testCommand": "npx vitest run"
}
```

### `.pear/sessions/{session-id}.json`

```json
{
  "sessionId": "abc123",
  "featureName": "User Authentication",
  "featurePath": "src/features/auth",
  "createdAt": "2025-01-15T10:00:00.000Z",
  "lastActivityAt": "2025-01-15T12:30:00.000Z",
  "currentPhase": "implementation",
  "phaseStatus": "in_progress",
  "planning": {
    "designDocContent": "# Feature: User Authentication...",
    "designDocPath": "src/features/auth/DESIGN.md",
    "approved": true
  },
  "interface": {
    "typesContent": "export interface AuthConfig {...}",
    "typesPath": "src/features/auth/types.ts",
    "testsContent": "describe('AuthService', () => {...})",
    "testsPath": "src/features/auth/__tests__/auth.test.ts",
    "approved": true,
    "dependencyAnalysis": {
      "graph": [...],
      "suggestedOrder": ["getOAuthUrl", "createSession", ...],
      "orderApproved": true
    }
  },
  "implementation": {
    "units": [
      {
        "id": "unit-1",
        "name": "getOAuthUrl",
        "description": "Generate OAuth redirect URL",
        "filePath": "src/features/auth/oauth.ts",
        "status": "approved",
        "implementation": "export function getOAuthUrl(...) {...}",
        "dependencies": []
      }
    ],
    "currentUnitIndex": 3,
    "dependencyOrder": ["unit-1", "unit-2", "unit-3", "unit-4", "unit-5"],
    "testImplementations": [...],
    "currentTestIndex": 0,
    "stage": "production"
  },
  "testing": {
    "testRuns": [],
    "lastTestRun": null,
    "debugIterations": 0,
    "manualTests": [],
    "manualTestsComplete": false,
    "testEvidencePath": null,
    "testEvidenceGenerated": false
  },
  "messages": [
    {
      "role": "user",
      "content": "Add Google OAuth authentication...",
      "timestamp": "2025-01-15T10:00:00.000Z",
      "phase": "planning"
    }
  ]
}
```

---

## Package Configuration

### `package.json`

```json
{
  "name": "pear-cli",
  "version": "0.1.0",
  "description": "AI pair-programming CLI",
  "main": "dist/index.js",
  "bin": {
    "pear": "dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "dev": "tsx src/index.ts",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint src/",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.30.0",
    "chalk": "^5.3.0",
    "commander": "^12.0.0",
    "inquirer": "^9.2.0",
    "marked": "^12.0.0",
    "marked-terminal": "^7.0.0",
    "ora": "^8.0.0",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "@types/inquirer": "^9.0.0",
    "@types/node": "^20.0.0",
    "@types/uuid": "^9.0.0",
    "typescript": "^5.4.0",
    "vitest": "^1.4.0",
    "tsx": "^4.7.0",
    "@typescript-eslint/eslint-plugin": "^7.0.0",
    "@typescript-eslint/parser": "^7.0.0",
    "eslint": "^8.57.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

### `vitest.config.ts`

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    include: ['tests/**/*.test.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
    },
  },
});
```

### `.env.example`

```bash
# Anthropic API key (required)
ANTHROPIC_API_KEY=sk-ant-...

# Optional: custom model
ANTHROPIC_MODEL=claude-sonnet-4-20250514

# Optional: enable debug logging
DEBUG=pear:*
```

### `.gitignore`

```gitignore
# Dependencies
node_modules/

# Build output
dist/

# Environment
.env
.env.local

# IDE
.vscode/
.idea/

# OS
.DS_Store

# Test coverage
coverage/

# Pear runtime (local sessions)
.pear/
```

---

## File Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Component files | camelCase | `stateManager.ts` |
| Type files | camelCase | `workflowTypes.ts` |
| Test files | same name + `.test` | `manager.test.ts` |
| Config files | kebab-case | `vitest.config.ts` |
| Generated feature files | kebab-case | `auth-service.ts` |

---

## Import Organization

```typescript
// 1. Node built-ins
import fs from 'fs/promises';
import path from 'path';

// 2. External packages
import Anthropic from '@anthropic-ai/sdk';
import chalk from 'chalk';

// 3. Internal types
import type { WorkflowState, Phase } from '../types';

// 4. Internal modules
import { StateManager } from '../state/manager';
import { LLMClient } from '../llm/client';
```

---

## Related Documents

- [02-components.md](./02-components.md) — What each module does
- [03-data-models.md](./03-data-models.md) — Type definitions
- [10-implementation-plan.md](./10-implementation-plan.md) — Build order

