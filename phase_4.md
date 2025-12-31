# Phase 4: Advanced Features & Team Scale

**Duration**: 4-8 weeks (ongoing)  
**Goal**: Expand Pear with advanced features, multiple platforms, team collaboration, and production-ready infrastructure.

**Prerequisites**: Phase 3 complete with working macOS editor

---

## Table of Contents

1. [Objectives](#objectives)
2. [Success Criteria](#success-criteria)
3. [Feature Categories](#feature-categories)
4. [Multiple LLM Providers](#multiple-llm-providers)
5. [Quick Mode](#quick-mode)
6. [Distributed Locking](#distributed-locking)
7. [Team Collaboration](#team-collaboration)
8. [Platform Expansion](#platform-expansion)
9. [Analytics & Insights](#analytics--insights)
10. [Custom Workflows](#custom-workflows)
11. [Implementation Roadmap](#implementation-roadmap)
12. [Testing Strategy](#testing-strategy)
13. [Exit Criteria](#exit-criteria)

---

## Objectives

### Primary Objectives

1. **Team scale**: Enable teams to use Pear collaboratively
2. **Platform support**: Windows and Linux distributions
3. **Provider flexibility**: Support multiple LLM providers
4. **Workflow flexibility**: Quick mode for small changes

### Secondary Objectives

1. Usage analytics for improvement insights
2. Custom workflow definitions
3. Plugin/extension system
4. Enterprise features

---

## Success Criteria

| Criteria | Measurement |
|----------|-------------|
| Multi-platform | Windows, Linux, macOS all work |
| Team usage | Multiple developers can use Pear on same repo |
| Provider choice | At least 3 LLM providers supported |
| Quick mode adoption | >30% of sessions use quick mode |
| No regressions | All Phase 3 functionality intact |

---

## Feature Categories

### Category 1: Flexibility (Weeks 1-2)
- Multiple LLM providers
- Quick mode for small changes
- Configurable workflow steps

### Category 2: Team (Weeks 3-5)
- Distributed locking via Git
- Session visibility
- Team dashboard

### Category 3: Platform (Weeks 6-7)
- Windows build
- Linux build
- Auto-update infrastructure

### Category 4: Insights (Week 8+)
- Usage analytics
- Productivity metrics
- Session history

---

## Multiple LLM Providers

### Architecture

```typescript
// src/llm/providers/index.ts

export interface LLMProvider {
  readonly name: string;
  readonly displayName: string;
  readonly capabilities: ProviderCapabilities;
  
  chat(request: ChatRequest): AsyncIterable<ChatChunk>;
  
  // Optional capabilities
  embeddings?(text: string): Promise<number[]>;
  countTokens?(text: string): Promise<number>;
}

export interface ProviderCapabilities {
  streaming: boolean;
  toolUse: boolean;
  vision: boolean;
  maxContextTokens: number;
  maxOutputTokens: number;
}

// Provider implementations
export class AnthropicProvider implements LLMProvider {
  readonly name = 'anthropic';
  readonly displayName = 'Anthropic Claude';
  readonly capabilities = {
    streaming: true,
    toolUse: true,
    vision: true,
    maxContextTokens: 200000,
    maxOutputTokens: 8192,
  };
  
  constructor(private options: AnthropicOptions) {}
  
  async *chat(request: ChatRequest): AsyncIterable<ChatChunk> {
    // Anthropic implementation
  }
}

export class OpenAIProvider implements LLMProvider {
  readonly name = 'openai';
  readonly displayName = 'OpenAI GPT-4';
  readonly capabilities = {
    streaming: true,
    toolUse: true,
    vision: true,
    maxContextTokens: 128000,
    maxOutputTokens: 4096,
  };
  
  constructor(private options: OpenAIOptions) {}
  
  async *chat(request: ChatRequest): AsyncIterable<ChatChunk> {
    // OpenAI implementation
  }
}

export class OllamaProvider implements LLMProvider {
  readonly name = 'ollama';
  readonly displayName = 'Ollama (Local)';
  readonly capabilities = {
    streaming: true,
    toolUse: false,
    vision: false,
    maxContextTokens: 8000,  // Varies by model
    maxOutputTokens: 2048,
  };
  
  constructor(private options: OllamaOptions) {}
  
  async *chat(request: ChatRequest): AsyncIterable<ChatChunk> {
    // Ollama implementation (local models)
  }
}
```

### Provider Selection UI

```
┌─────────────────────────────────────────────────────────────────────┐
│ LLM Provider Settings                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Select Provider                                                     │
│ ┌───────────────────────────────────────────────────────────────┐   │
│ │ ◉ Anthropic Claude                                            │   │
│ │   Best for code generation, large context window              │   │
│ │   API Key: sk-ant-api03-...  [Change]                         │   │
│ │                                                               │   │
│ │ ○ OpenAI GPT-4                                                │   │
│ │   Good general purpose, vision support                        │   │
│ │   API Key: Not configured  [Configure]                        │   │
│ │                                                               │   │
│ │ ○ Ollama (Local)                                              │   │
│ │   Run locally, no API costs, slower                           │   │
│ │   Status: Not installed  [Setup Guide]                        │   │
│ └───────────────────────────────────────────────────────────────┘   │
│                                                                     │
│ Model Selection                                                     │
│ ┌───────────────────────────────────────────────────────────────┐   │
│ │ Model: [claude-sonnet-4-20250514              ▼]              │   │
│ │                                                               │   │
│ │ Context: 200K tokens   Output: 8K tokens                      │   │
│ │ Capabilities: Streaming ✓  Tools ✓  Vision ✓                  │   │
│ └───────────────────────────────────────────────────────────────┘   │
│                                                                     │
│                                          [Save] [Cancel]            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Configuration

```yaml
# .pear/config.yaml

llm:
  # Primary provider
  provider: anthropic
  model: claude-sonnet-4-20250514
  
  # Provider-specific settings
  providers:
    anthropic:
      apiKey: ${ANTHROPIC_API_KEY}
      models:
        - claude-sonnet-4-20250514
        - claude-3-5-sonnet-20241022
    
    openai:
      apiKey: ${OPENAI_API_KEY}
      models:
        - gpt-4-turbo
        - gpt-4o
    
    ollama:
      baseUrl: http://localhost:11434
      models:
        - codellama:70b
        - deepseek-coder:33b

  # Per-phase provider overrides (optional)
  phaseOverrides:
    planning:
      provider: anthropic
      model: claude-sonnet-4-20250514  # Better for brainstorming
    implementation:
      provider: anthropic
      model: claude-sonnet-4-20250514  # Best for code
```

---

## Quick Mode

### Concept

Quick Mode is a streamlined workflow for small, well-defined changes that don't need the full four-phase structure.

**When to use Quick Mode:**
- Bug fixes
- Small refactors
- Adding a single function
- Documentation updates
- Config changes

**Quick Mode workflow:**
```
Describe Change → Generate → Review → Apply
      │              │          │        │
    1 step        1 step    1 step   Done
```

**Full Mode workflow:**
```
Plan → Interface → Implement → Test
  │        │           │         │
4+ steps  3+ steps  N steps   2+ steps
```

### Implementation

```typescript
// src/workflow/quickMode.ts

export class QuickModeWorkflow {
  constructor(
    private readonly llmOrchestrator: LLMOrchestrator,
    private readonly fileManager: FileManager,
  ) {}
  
  async *run(options: QuickModeOptions): AsyncIterable<QuickModeEvent> {
    yield { type: 'started', description: options.description };
    
    // 1. Analyze current state
    const context = await this.buildContext(options);
    
    // 2. Generate changes
    yield { type: 'generating' };
    
    const changes: FileChange[] = [];
    
    for await (const chunk of this.llmOrchestrator.quickChange(options.description, context)) {
      yield { type: 'chunk', content: chunk };
      
      // Parse file changes from response
      const parsed = this.parseChanges(chunk);
      if (parsed) changes.push(...parsed);
    }
    
    yield { type: 'generated', changes };
    
    // 3. Await human review
    yield { type: 'awaiting_review', changes };
    
    // 4. Apply changes (called by controller after approval)
  }
  
  async applyChanges(changes: FileChange[]): Promise<ApplyResult> {
    return this.fileManager.applyChanges(changes);
  }
}

export interface QuickModeOptions {
  description: string;
  targetFiles?: string[];  // Limit scope
  featurePath?: string;    // Context from existing feature
}

export interface FileChange {
  path: string;
  type: 'create' | 'modify' | 'delete';
  content?: string;
  diff?: string;
}
```

### Quick Mode UI

```
┌─────────────────────────────────────────────────────────────────────┐
│ 🍐 Quick Mode                                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ What would you like to change?                                      │
│ ┌───────────────────────────────────────────────────────────────┐   │
│ │ Add error handling to the validateSession function             │   │
│ └───────────────────────────────────────────────────────────────┘   │
│                                                                     │
│ Scope (optional)                                                    │
│ ┌───────────────────────────────────────────────────────────────┐   │
│ │ ☑ src/features/auth/auth.ts                                   │   │
│ │ ☐ Include related test file                                   │   │
│ └───────────────────────────────────────────────────────────────┘   │
│                                                                     │
│                            [Generate Changes] [Use Full Mode]       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

After generation:

```
┌─────────────────────────────────────────────────────────────────────┐
│ 🍐 Quick Mode - Review Changes                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Proposed Changes                                                    │
│                                                                     │
│ ┌─ src/features/auth/auth.ts ───────────────────────────────────┐   │
│ │   export async function validateSession(                      │   │
│ │     sessionId: string                                         │   │
│ │   ): Promise<UserSession | null> {                            │   │
│ │ +   if (!sessionId || typeof sessionId !== 'string') {        │   │
│ │ +     throw new InvalidSessionIdError(sessionId);             │   │
│ │ +   }                                                         │   │
│ │ +                                                             │   │
│ │     const session = await db.sessions.find(sessionId);        │   │
│ │ +   if (!session) {                                           │   │
│ │ +     return null;                                            │   │
│ │ +   }                                                         │   │
│ │ -   if (!session) return null;                                │   │
│ │     if (session.expiresAt < new Date()) return null;          │   │
│ │     return session;                                           │   │
│ │   }                                                           │   │
│ └───────────────────────────────────────────────────────────────┘   │
│                                                                     │
│ Summary: Added input validation and improved null handling          │
│                                                                     │
│               [Apply Changes] [Modify] [Cancel]                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Configuration

```yaml
# .pear/config.yaml

workflow:
  # Quick mode settings
  quickMode:
    enabled: true
    maxFilesTouched: 3     # If more files needed, suggest full mode
    requireTestUpdate: false  # Quick mode doesn't require test updates
    
  # Heuristics for suggesting quick vs full mode
  suggestions:
    suggestQuickFor:
      - "fix"
      - "typo"
      - "rename"
      - "add comment"
      - "update config"
    suggestFullFor:
      - "add feature"
      - "implement"
      - "create new"
      - "design"
```

---

## Distributed Locking

### Problem

Phase 1-3 use local file-based locks. This doesn't work for teams:

- Developer A on Machine 1 locks `src/features/auth/`
- Developer B on Machine 2 doesn't see this lock
- Both could edit the same files

### Solution: Git-Based Locking

Use Git as the coordination mechanism:

```
┌─────────────────────────────────────────────────────────────────┐
│                  DISTRIBUTED LOCK FLOW                          │
│                                                                 │
│   1. Developer wants to lock src/features/auth/                 │
│                                                                 │
│   2. Pear fetches latest lock state                             │
│      $ git fetch origin pear-locks                              │
│                                                                 │
│   3. Pear checks if path is already locked                      │
│      - Read .pear-locks/locks.yaml from pear-locks branch       │
│                                                                 │
│   4. If not locked, add lock entry                              │
│      - Modify .pear-locks/locks.yaml                            │
│      - Commit to pear-locks branch                              │
│      - Push to origin                                           │
│                                                                 │
│   5. If push succeeds → lock acquired                           │
│      If push fails (conflict) → another developer got it first  │
│         - Fetch again, check who has it, notify user            │
│                                                                 │
│   6. When done, remove lock entry and push                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```typescript
// src/lock/distributed/gitLockManager.ts

export class GitDistributedLockManager implements LockManager {
  private readonly git: SimpleGit;
  private readonly lockBranch = 'pear-locks';
  private readonly lockFile = '.pear-locks/locks.yaml';
  
  constructor(
    private readonly repoPath: string,
    private readonly remote: string = 'origin',
  ) {
    this.git = simpleGit(repoPath);
  }
  
  async acquireWriteLock(
    path: string,
    sessionId: string,
    metadata: LockMetadata
  ): Promise<LockResult> {
    const maxRetries = 3;
    
    for (let attempt = 0; attempt < maxRetries; attempt++) {
      try {
        // 1. Fetch latest lock state
        await this.git.fetch(this.remote, this.lockBranch);
        
        // 2. Check current locks
        const locks = await this.readLocks();
        const existing = locks.find(l => l.path === path);
        
        if (existing && existing.sessionId !== sessionId) {
          return {
            success: false,
            reason: 'locked_by_other',
            holder: existing,
          };
        }
        
        // 3. Add/update lock
        const newLock: FolderLock = {
          path,
          sessionId,
          type: 'write',
          acquiredAt: new Date(),
          ...metadata,
        };
        
        const updatedLocks = locks.filter(l => l.path !== path);
        updatedLocks.push(newLock);
        
        // 4. Commit and push
        await this.writeLocks(updatedLocks);
        await this.git.add(this.lockFile);
        await this.git.commit(`lock: ${path} by ${metadata.user}`);
        await this.git.push(this.remote, this.lockBranch);
        
        return { success: true, lock: newLock };
        
      } catch (error) {
        if (this.isConflictError(error)) {
          // Another developer acquired the lock, retry
          await sleep(100 * (attempt + 1));
          continue;
        }
        throw error;
      }
    }
    
    // Max retries exceeded
    const locks = await this.readLocks();
    const holder = locks.find(l => l.path === path);
    return {
      success: false,
      reason: 'conflict',
      holder,
    };
  }
  
  async releaseLock(path: string, sessionId: string): Promise<void> {
    await this.git.fetch(this.remote, this.lockBranch);
    
    const locks = await this.readLocks();
    const lock = locks.find(l => l.path === path);
    
    if (!lock || lock.sessionId !== sessionId) {
      return; // Not our lock or already released
    }
    
    const updatedLocks = locks.filter(l => l.path !== path);
    await this.writeLocks(updatedLocks);
    await this.git.add(this.lockFile);
    await this.git.commit(`unlock: ${path}`);
    await this.git.push(this.remote, this.lockBranch);
  }
  
  private async readLocks(): Promise<FolderLock[]> {
    try {
      const content = await this.git.show(`${this.remote}/${this.lockBranch}:${this.lockFile}`);
      return yaml.parse(content)?.locks ?? [];
    } catch {
      return [];
    }
  }
  
  private async writeLocks(locks: FolderLock[]): Promise<void> {
    const content = yaml.stringify({ locks, updatedAt: new Date() });
    await fs.writeFile(path.join(this.repoPath, this.lockFile), content);
  }
}
```

### Lock Branch Structure

```yaml
# .pear-locks/locks.yaml (on pear-locks branch)

updatedAt: 2025-01-15T14:30:00Z

locks:
  - path: src/features/auth
    sessionId: abc123
    type: write
    user: jake
    machine: jakes-macbook
    feature: "User Authentication"
    acquiredAt: 2025-01-15T10:00:00Z
    
  - path: src/features/payments
    sessionId: def456
    type: write
    user: sarah
    machine: sarahs-laptop
    feature: "Payment Processing"
    acquiredAt: 2025-01-15T10:30:00Z
```

### GitHub Integration

```typescript
// src/lock/distributed/githubNotifier.ts

export class GitHubLockNotifier {
  constructor(
    private readonly octokit: Octokit,
    private readonly repo: { owner: string; repo: string },
  ) {}
  
  // Post lock status as PR comment
  async notifyPR(prNumber: number, locks: FolderLock[]): Promise<void> {
    const body = this.formatLockStatus(locks);
    await this.octokit.issues.createComment({
      ...this.repo,
      issue_number: prNumber,
      body,
    });
  }
  
  private formatLockStatus(locks: FolderLock[]): string {
    if (locks.length === 0) {
      return '🍐 No active Pear locks on this branch.';
    }
    
    let body = '🍐 **Active Pear Locks**\n\n';
    body += '| Path | User | Feature | Since |\n';
    body += '|------|------|---------|-------|\n';
    
    for (const lock of locks) {
      body += `| \`${lock.path}\` | @${lock.user} | ${lock.feature} | ${formatTimeAgo(lock.acquiredAt)} |\n`;
    }
    
    return body;
  }
}
```

---

## Team Collaboration

### Session Visibility

Show active sessions from all team members:

```
┌─────────────────────────────────────────────────────────────────────┐
│ 🍐 Team Activity                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Active Sessions                                                     │
│                                                                     │
│ ┌───────────────────────────────────────────────────────────────┐   │
│ │ 👤 jake                                       🟢 Active        │   │
│ │ Feature: User Authentication                                  │   │
│ │ Path: src/features/auth/                                      │   │
│ │ Phase: Implementation (4/7)                                   │   │
│ │ Since: 2 hours ago                                            │   │
│ └───────────────────────────────────────────────────────────────┘   │
│                                                                     │
│ ┌───────────────────────────────────────────────────────────────┐   │
│ │ 👤 sarah                                      🟡 Paused        │   │
│ │ Feature: Payment Processing                                   │   │
│ │ Path: src/features/payments/                                  │   │
│ │ Phase: Testing                                                │   │
│ │ Since: 1 day ago (paused 3 hours ago)                         │   │
│ └───────────────────────────────────────────────────────────────┘   │
│                                                                     │
│ Recent Completions                                                  │
│                                                                     │
│ ┌───────────────────────────────────────────────────────────────┐   │
│ │ ✓ alex completed "Email Notifications" - 4 hours ago          │   │
│ │ ✓ jordan completed "User Profiles" - yesterday                │   │
│ └───────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Slack/Discord Integration

```typescript
// src/integrations/slack.ts

export class SlackIntegration {
  constructor(
    private readonly webhookUrl: string,
  ) {}
  
  async notifyLockAcquired(lock: FolderLock): Promise<void> {
    await this.send({
      text: `🍐 *${lock.user}* started working on *${lock.feature}*`,
      blocks: [
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: `🍐 *${lock.user}* started working on *${lock.feature}*\n\`${lock.path}\``,
          },
        },
      ],
    });
  }
  
  async notifySessionComplete(session: SessionInfo): Promise<void> {
    await this.send({
      text: `✅ *${session.user}* completed *${session.featureName}*`,
      blocks: [
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: `✅ *${session.user}* completed *${session.featureName}*\n` +
                  `Duration: ${formatDuration(session.duration)}\n` +
                  `Files: ${session.filesCreated} created, ${session.filesModified} modified`,
          },
        },
      ],
    });
  }
}
```

### Configuration

```yaml
# .pear/config.yaml

team:
  # Distributed locking
  distributedLocking:
    enabled: true
    provider: git  # git | redis | custom
    remote: origin
    branch: pear-locks
  
  # Visibility
  visibility:
    showTeamActivity: true
    showRecentCompletions: true
    maxRecentToShow: 10
  
  # Notifications
  notifications:
    slack:
      enabled: true
      webhookUrl: ${PEAR_SLACK_WEBHOOK}
      events:
        - lock_acquired
        - session_complete
        - lock_conflict
    
    discord:
      enabled: false
      webhookUrl: ${PEAR_DISCORD_WEBHOOK}
```

---

## Platform Expansion

### Windows Build

```powershell
# scripts/build-windows.ps1

Write-Host "Building Pear for Windows..."

# Build
yarn gulp vscode-win32-x64-min

# Sign (requires certificate)
& signtool sign /f certificate.pfx /p $env:CERT_PASSWORD /t http://timestamp.comodoca.com `
    "..\VSCode-win32-x64\Pear.exe"

# Create installer
& "C:\Program Files (x86)\NSIS\makensis.exe" installer.nsi

Write-Host "Build complete!"
```

### Linux Build

```bash
# scripts/build-linux.sh

#!/bin/bash
set -e

echo "Building Pear for Linux..."

# Build
yarn gulp vscode-linux-x64-min

# Create .deb package
electron-installer-debian \
  --src ../VSCode-linux-x64 \
  --dest ../installers \
  --arch amd64

# Create .rpm package
electron-installer-redhat \
  --src ../VSCode-linux-x64 \
  --dest ../installers \
  --arch x86_64

# Create AppImage
electron-builder --linux AppImage

echo "Build complete!"
```

### Auto-Update Infrastructure

```typescript
// src/update/autoUpdater.ts

import { autoUpdater } from 'electron-updater';

export class PearAutoUpdater {
  constructor() {
    autoUpdater.setFeedURL({
      provider: 'github',
      owner: 'pear',
      repo: 'editor',
    });
    
    autoUpdater.on('update-available', (info) => {
      this.notifyUpdateAvailable(info);
    });
    
    autoUpdater.on('update-downloaded', (info) => {
      this.promptInstall(info);
    });
  }
  
  async checkForUpdates(): Promise<void> {
    await autoUpdater.checkForUpdates();
  }
  
  private notifyUpdateAvailable(info: UpdateInfo): void {
    // Show notification in status bar
  }
  
  private promptInstall(info: UpdateInfo): void {
    // Show dialog: "Restart to update?"
  }
}
```

---

## Analytics & Insights

### Privacy-First Analytics

All analytics are opt-in and anonymized:

```typescript
// src/analytics/collector.ts

export interface SessionMetrics {
  // Timing (no identifiable info)
  totalDuration: number;
  planningDuration: number;
  interfaceDuration: number;
  implementationDuration: number;
  testingDuration: number;
  
  // Counts
  unitsImplemented: number;
  approvalIterations: number;
  goBackCount: number;
  
  // LLM usage
  llmCallCount: number;
  totalTokens: number;
  
  // Outcome
  completed: boolean;
  quickMode: boolean;
}

export class AnalyticsCollector {
  constructor(
    private readonly storage: AnalyticsStorage,
    private readonly enabled: boolean,
  ) {}
  
  async recordSession(metrics: SessionMetrics): Promise<void> {
    if (!this.enabled) return;
    
    // Anonymize - no session ID, user, or feature names
    const anonymous = {
      ...metrics,
      timestamp: new Date(),
      pearVersion: PEAR_VERSION,
    };
    
    await this.storage.store(anonymous);
  }
  
  async getInsights(): Promise<Insights> {
    const sessions = await this.storage.getRecent(100);
    
    return {
      averageDuration: this.calculateAverage(sessions, 'totalDuration'),
      completionRate: this.calculateCompletionRate(sessions),
      mostTimeSpentPhase: this.findLongestPhase(sessions),
      averageIterations: this.calculateAverage(sessions, 'approvalIterations'),
      quickModeUsage: this.calculateQuickModePercentage(sessions),
    };
  }
}
```

### Productivity Dashboard

```
┌─────────────────────────────────────────────────────────────────────┐
│ 🍐 Your Pear Insights                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Last 30 Days                                                        │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Sessions Completed: 23                                          │ │
│ │ Average Duration: 45 minutes                                    │ │
│ │ Quick Mode Usage: 35%                                           │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ Time Breakdown                                                      │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Planning       ████░░░░░░░░░░░░░░░░ 15%                         │ │
│ │ Interface      ██████░░░░░░░░░░░░░░ 20%                         │ │
│ │ Implementation ████████████████░░░░ 55%                         │ │
│ │ Testing        ██░░░░░░░░░░░░░░░░░░ 10%                         │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ Trends                                                              │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ ↓ 20% fewer iterations per session vs last month                │ │
│ │ ↑ 15% more quick mode usage                                     │ │
│ │ → Session duration stable                                       │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Custom Workflows

### Workflow Definition

Allow users to customize the workflow:

```yaml
# .pear/workflows/custom.yaml

name: "Quick Feature"
description: "Streamlined workflow for small features"

phases:
  - name: planning
    required: true
    autoApprove: false
    
  - name: interface
    required: true
    autoApprove: false
    # Skip tests for quick features
    components:
      - types
      # - tests  (commented out - skipped)
    
  - name: implementation
    required: true
    autoApprove: false
    # Batch approve - don't review every unit
    batchApprove: true
    batchSize: 3
    
  - name: testing
    required: false  # Optional for quick features
    autoApprove: true

# Conditions for when to suggest this workflow
suggestWhen:
  featureNameContains:
    - "small"
    - "quick"
    - "minor"
  maxEstimatedUnits: 5
```

### Workflow Engine

```typescript
// src/workflow/customWorkflow.ts

export interface WorkflowDefinition {
  name: string;
  description: string;
  phases: PhaseDefinition[];
  suggestWhen?: SuggestionConditions;
}

export interface PhaseDefinition {
  name: Phase;
  required: boolean;
  autoApprove: boolean;
  components?: string[];
  batchApprove?: boolean;
  batchSize?: number;
}

export class CustomWorkflowEngine {
  constructor(
    private readonly definitions: WorkflowDefinition[],
  ) {}
  
  suggestWorkflow(options: CreateSessionOptions): WorkflowDefinition {
    for (const def of this.definitions) {
      if (this.matchesConditions(def.suggestWhen, options)) {
        return def;
      }
    }
    return this.getDefaultWorkflow();
  }
  
  private matchesConditions(
    conditions: SuggestionConditions | undefined,
    options: CreateSessionOptions
  ): boolean {
    if (!conditions) return false;
    
    if (conditions.featureNameContains) {
      const lower = options.featureName.toLowerCase();
      return conditions.featureNameContains.some(s => lower.includes(s));
    }
    
    return false;
  }
}
```

---

## Implementation Roadmap

### Weeks 1-2: Flexibility

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-2 | Multiple LLM providers interface | Provider abstraction |
| 3-4 | OpenAI provider implementation | GPT-4 works |
| 5 | Ollama provider implementation | Local models work |
| 6-7 | Provider selection UI | Can switch in settings |
| 8-9 | Quick mode workflow | Quick mode works |
| 10 | Quick mode UI | Integrated in editor |

### Weeks 3-5: Team

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-3 | Git-based distributed locking | Locks sync across machines |
| 4-5 | Lock conflict handling | Graceful conflict resolution |
| 6-7 | Team activity panel | See other developers |
| 8-9 | Slack/Discord integration | Notifications work |
| 10-12 | GitHub integration | PR lock status |
| 13-15 | Testing & bug fixes | Stable team features |

### Weeks 6-7: Platform

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-3 | Windows build pipeline | Windows installer |
| 4-5 | Windows testing | Works on Windows |
| 6-7 | Linux build pipeline | .deb and .rpm packages |
| 8-9 | Linux testing | Works on Linux |
| 10 | Auto-update infrastructure | Updates work |

### Week 8+: Insights & Polish

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-2 | Analytics collection | Opt-in metrics |
| 3-4 | Insights dashboard | View productivity |
| 5+ | Custom workflows | Workflow definitions |
| Ongoing | Bug fixes, polish | Stable release |

---

## Testing Strategy

### Provider Tests

```typescript
describe('LLM Providers', () => {
  describe('AnthropicProvider', () => {
    it('should stream responses', async () => {});
    it('should handle rate limits', async () => {});
    it('should report token usage', async () => {});
  });
  
  describe('OpenAIProvider', () => {
    it('should stream responses', async () => {});
    it('should handle API errors', async () => {});
  });
  
  describe('OllamaProvider', () => {
    it('should connect to local instance', async () => {});
    it('should handle connection failure', async () => {});
  });
});
```

### Distributed Locking Tests

```typescript
describe('GitDistributedLockManager', () => {
  it('should acquire lock on empty branch', async () => {});
  it('should fail when path already locked', async () => {});
  it('should retry on conflict', async () => {});
  it('should release lock', async () => {});
  it('should handle concurrent acquisitions', async () => {});
});
```

### Platform Tests

- [ ] Windows: Fresh install
- [ ] Windows: Upgrade from previous version
- [ ] Linux (Ubuntu): Fresh install
- [ ] Linux (Fedora): Fresh install
- [ ] Auto-update on all platforms

---

## Exit Criteria

Phase 4 is considered complete (for v1.0) when:

1. **Multi-platform**: Windows, Linux, macOS all stable
2. **Team ready**: Distributed locking works for teams of 5+
3. **Provider choice**: 3+ LLM providers available
4. **Quick mode**: Works for simple changes
5. **Documented**: All features documented

### Deliverables

- [ ] Windows installer
- [ ] Linux packages (.deb, .rpm, AppImage)
- [ ] Multiple LLM provider support
- [ ] Quick mode workflow
- [ ] Distributed locking
- [ ] Team activity dashboard
- [ ] Slack/Discord integration
- [ ] Analytics dashboard
- [ ] Updated documentation

---

## Beyond v1.0

### Exploratory Features

- **Voice interface**: "Hey Pear, implement the login function"
- **AI code review**: Second LLM reviews first LLM's code
- **Learning mode**: Pear learns team's coding preferences
- **IDE plugins**: Lightweight plugins for VS Code/JetBrains (not full workflow)
- **Enterprise**: SSO, audit logs, centralized management
- **Marketplace**: Share custom workflows and prompts

### Long-Term Vision

Pear becomes the standard way for developers to pair with AI:
- Every developer has a Pear instance
- Teams share workflows and best practices
- AI coding becomes structured and predictable
- Human stays in control while AI handles the typing

