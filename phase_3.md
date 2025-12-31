# Phase 3: VS Code Fork

**Duration**: 4-8 weeks  
**Goal**: Build a complete, purpose-built editor by forking VS Code and integrating the Pear core engine.

**Prerequisites**: Phase 2 complete with working `@pear/core` library

---

## Table of Contents

1. [Objectives](#objectives)
2. [Success Criteria](#success-criteria)
3. [Scope](#scope)
4. [VS Code Fork Strategy](#vs-code-fork-strategy)
5. [Architecture](#architecture)
6. [UI Components](#ui-components)
7. [Core Integration](#core-integration)
8. [Custom Features](#custom-features)
9. [Build & Distribution](#build--distribution)
10. [Implementation Plan](#implementation-plan)
11. [Testing Strategy](#testing-strategy)
12. [Risk Mitigation](#risk-mitigation)
13. [Exit Criteria](#exit-criteria)

---

## Objectives

### Primary Objectives

1. **Purpose-built editor**: A standalone editor optimized for the Pear workflow
2. **Seamless integration**: `@pear/core` powers all workflow logic
3. **Workflow-first UX**: Phases are the primary navigation paradigm
4. **Familiar foundation**: Leverage VS Code's editor, LSP support, and keybindings

### Secondary Objectives

1. Custom branding (Pear identity)
2. Streamlined UI (remove unused VS Code features)
3. Phase-aware keybindings
4. Lock visualization in file explorer
5. Onboarding flow for new projects

---

## Success Criteria

| Criteria | Measurement |
|----------|-------------|
| Complete workflow in editor | All phases work end-to-end |
| Faster than CLI | Subjective: better UX |
| Stable | No crashes during normal use |
| Performant | <100ms UI response, streaming works smoothly |
| Distributable | Can build and distribute packaged app |

---

## Scope

### In Scope

- Fork VS Code (latest stable)
- Integrate `@pear/core` for workflow logic
- Custom UI: Workflow Sidebar, Conversation Panel
- Phase-aware keybindings
- Lock status display
- Project onboarding flow
- Custom branding
- macOS distribution (primary)

### Out of Scope (Phase 4)

- Windows/Linux distribution
- Multiple LLM providers UI
- Team collaboration features
- Quick mode
- Analytics

---

## VS Code Fork Strategy

### Why Fork vs Extension

The Pear workflow requires capabilities that VS Code's extension API doesn't support:

| Requirement | Extension API | Fork |
|-------------|---------------|------|
| Replace sidebar with workflow panel | ❌ Limited webview | ✅ Full control |
| Phase-aware keybindings | ⚠️ Hacky | ✅ Native |
| Restrict file editing | ❌ Not possible | ✅ Full control |
| Custom diff views | ⚠️ Limited | ✅ Native |
| Integrated conversation panel | ⚠️ Webview only | ✅ Native |
| Remove unused UI elements | ❌ Not possible | ✅ Full control |

### Fork Maintenance Strategy

VS Code releases monthly. Our strategy:

1. **Initial fork**: Based on latest stable (e.g., 1.88.0)
2. **Selective updates**: Cherry-pick security fixes
3. **Quarterly rebase**: Merge upstream changes every 3 months
4. **Isolation**: Keep Pear-specific code in separate modules

### Fork Structure

```
pear-editor/
├── .vscode/                    # Editor config
├── build/                      # Build scripts
├── extensions/                 # Built-in extensions (pruned)
├── resources/                  # Pear branding assets
├── scripts/                    # Dev scripts
├── src/
│   ├── vs/                     # VS Code source (modified)
│   │   ├── base/               # Base utilities (minimal changes)
│   │   ├── editor/             # Editor core (minimal changes)
│   │   ├── platform/           # Platform services
│   │   ├── workbench/          # Main workbench (most changes here)
│   │   │   ├── contrib/
│   │   │   │   └── pear/       # ← NEW: All Pear-specific code
│   │   │   │       ├── browser/
│   │   │   │       │   ├── pear.contribution.ts
│   │   │   │       │   ├── workflowSidebar/
│   │   │   │       │   ├── conversationPanel/
│   │   │   │       │   └── lockStatus/
│   │   │   │       ├── common/
│   │   │   │       │   ├── pear.ts
│   │   │   │       │   └── types.ts
│   │   │   │       └── node/
│   │   │   │           └── pearEngine.ts
│   │   │   └── ... (other contrib modules)
│   │   └── code/               # Main entry points
│   └── pear/                   # ← NEW: Standalone Pear modules
│       ├── core/               # @pear/core integration
│       └── ui/                 # React components
├── test/
└── package.json
```

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PEAR EDITOR                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    VS Code Workbench                         │   │
│  │  ┌────────────┐  ┌─────────────────────┐  ┌───────────────┐ │   │
│  │  │ Activity   │  │    Editor Area      │  │ Secondary     │ │   │
│  │  │ Bar        │  │    (Monaco)         │  │ Sidebar       │ │   │
│  │  │            │  │                     │  │               │ │   │
│  │  │ [Pear]     │  │  ┌───────────────┐  │  │ ┌───────────┐ │ │   │
│  │  │ [Explorer] │  │  │ Code Editor   │  │  │ │ Workflow  │ │ │   │
│  │  │ [Search]   │  │  │               │  │  │ │ Sidebar   │ │ │   │
│  │  │            │  │  │               │  │  │ │           │ │ │   │
│  │  │            │  │  └───────────────┘  │  │ │ Phase 2/4 │ │ │   │
│  │  │            │  │                     │  │ │ ────────  │ │ │   │
│  │  │            │  │                     │  │ │ ☑ Plan    │ │ │   │
│  │  │            │  │                     │  │ │ ☐ Iface   │ │ │   │
│  │  │            │  │                     │  │ │ ☐ Impl    │ │ │   │
│  │  │            │  │                     │  │ │ ☐ Test    │ │ │   │
│  │  │            │  │                     │  │ └───────────┘ │ │   │
│  │  └────────────┘  └─────────────────────┘  └───────────────┘ │   │
│  │                                                              │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │                 Conversation Panel                    │   │   │
│  │  │  [Pear]: Here are the proposed types...              │   │   │
│  │  │  [You]: Add lastLoginAt field                        │   │   │
│  │  │  ─────────────────────────────────────               │   │   │
│  │  │  [Approve]  [Modify]  [Go Back]                      │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                         PEAR SERVICES                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐        │
│  │ PearEngine     │  │ WorkflowState  │  │ Lock           │        │
│  │ Service        │  │ Service        │  │ Service        │        │
│  │ (@pear/core)   │  │                │  │                │        │
│  └───────┬────────┘  └───────┬────────┘  └───────┬────────┘        │
│          │                   │                   │                  │
│          └───────────────────┴───────────────────┘                  │
│                              │                                      │
│                              ▼                                      │
│                    ┌────────────────┐                               │
│                    │  @pear/core    │                               │
│                    │  Library       │                               │
│                    └────────────────┘                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Service Architecture

VS Code uses a dependency injection system. We'll add Pear services:

```typescript
// src/vs/workbench/contrib/pear/common/pear.ts

import { createDecorator } from 'vs/platform/instantiation/common/instantiation';
import { PearEngine, PearSession, WorkflowState } from '@pear/core';

// Service identifiers
export const IPearEngineService = createDecorator<IPearEngineService>('pearEngineService');
export const IWorkflowStateService = createDecorator<IWorkflowStateService>('workflowStateService');
export const ILockService = createDecorator<ILockService>('lockService');

// Interfaces
export interface IPearEngineService {
  readonly _serviceBrand: undefined;
  
  readonly engine: PearEngine;
  readonly currentSession: PearSession | null;
  
  createSession(options: CreateSessionOptions): Promise<PearSession>;
  resumeSession(sessionId: string): Promise<PearSession | null>;
  listIncompleteSessions(): Promise<SessionInfo[]>;
}

export interface IWorkflowStateService {
  readonly _serviceBrand: undefined;
  
  readonly state: WorkflowState | null;
  readonly onStateChange: Event<WorkflowState>;
  
  getState(): WorkflowState | null;
  getCurrentPhase(): Phase | null;
  getAvailableActions(): WorkflowAction[];
}

export interface ILockService {
  readonly _serviceBrand: undefined;
  
  readonly locks: FolderLock[];
  readonly onLocksChange: Event<FolderLock[]>;
  
  getLocks(): Promise<FolderLock[]>;
  isLocked(path: string): boolean;
  getLocker(path: string): LockHolder | null;
}
```

---

## UI Components

### 1. Workflow Sidebar

The primary navigation for Pear sessions.

```
┌─────────────────────────────────┐
│ 🍐 PEAR                         │
├─────────────────────────────────┤
│                                 │
│ Current Session                 │
│ ┌─────────────────────────────┐ │
│ │ User Authentication         │ │
│ │ src/features/auth/          │ │
│ │ ─────────────────────       │ │
│ │ Phase 3 of 4                │ │
│ └─────────────────────────────┘ │
│                                 │
│ Progress                        │
│ ┌─────────────────────────────┐ │
│ │ ✓ Planning        [Done]    │ │
│ │ ✓ Interface       [Done]    │ │
│ │ ◐ Implementation  [4/7]     │ │
│ │ ○ Testing         [Pending] │ │
│ └─────────────────────────────┘ │
│                                 │
│ Implementation Units            │
│ ┌─────────────────────────────┐ │
│ │ ✓ getOAuthUrl()             │ │
│ │ ✓ createSession()           │ │
│ │ ✓ handleCallback()          │ │
│ │ ◐ validateSession()  [Now]  │ │
│ │ ○ revokeSession()           │ │
│ │ ○ authMiddleware()          │ │
│ │ ○ login()                   │ │
│ └─────────────────────────────┘ │
│                                 │
│ Artifacts                       │
│ ┌─────────────────────────────┐ │
│ │ 📄 DESIGN.md                │ │
│ │ 📄 types.ts                 │ │
│ │ 📄 auth.test.ts             │ │
│ └─────────────────────────────┘ │
│                                 │
│ [New Feature] [Resume...]       │
│                                 │
└─────────────────────────────────┘
```

**Implementation**: React component in VS Code webview.

```typescript
// src/vs/workbench/contrib/pear/browser/workflowSidebar/workflowSidebar.tsx

import * as React from 'react';
import { useWorkflowState } from './hooks/useWorkflowState';

export const WorkflowSidebar: React.FC = () => {
  const { state, currentPhase, units, actions } = useWorkflowState();
  
  if (!state) {
    return <NoSessionView onNewFeature={handleNewFeature} onResume={handleResume} />;
  }
  
  return (
    <div className="workflow-sidebar">
      <SessionHeader 
        featureName={state.featureName}
        featurePath={state.featurePath}
        phase={currentPhase}
      />
      
      <PhaseProgress phases={PHASES} currentPhase={currentPhase} state={state} />
      
      {currentPhase === 'implementation' && (
        <ImplementationUnits units={units} currentIndex={state.implementation.currentUnitIndex} />
      )}
      
      <Artifacts state={state} />
      
      <SidebarActions actions={actions} />
    </div>
  );
};
```

### 2. Conversation Panel

The chat interface for human-LLM dialogue.

```
┌─────────────────────────────────────────────────────────────────────┐
│ Conversation                                              [Clear]   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 🍐 Pear                                           10:32 AM      │ │
│ │                                                                 │ │
│ │ Here's the implementation for `validateSession()`:              │ │
│ │                                                                 │ │
│ │ ┌─────────────────────────────────────────────────────────────┐ │ │
│ │ │ export async function validateSession(                      │ │ │
│ │ │   sessionId: string                                         │ │ │
│ │ │ ): Promise<UserSession | null> {                            │ │ │
│ │ │   const session = await db.sessions.find(sessionId);        │ │ │
│ │ │   if (!session) return null;                                │ │ │
│ │ │   if (session.expiresAt < new Date()) return null;          │ │ │
│ │ │   return session;                                           │ │ │
│ │ │ }                                                           │ │ │
│ │ └─────────────────────────────────────────────────────────────┘ │ │
│ │                                                                 │ │
│ │ **Notes:**                                                      │ │
│ │ - Checks expiry before returning                                │ │
│ │ - Returns null for expired sessions (matches test expectations) │ │
│ │                                                                 │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 👤 You                                            10:33 AM      │ │
│ │                                                                 │ │
│ │ Also delete expired sessions from the database                  │ │
│ │                                                                 │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 🍐 Pear                                           10:33 AM      │ │
│ │                                                                 │ │
│ │ Updated implementation:                                         │ │
│ │ ...                                                             │ │
│ │                                                                 │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Type a message...                                               │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐                           │
│ │ Approve  │  │  Modify  │  │ Go Back  │                           │
│ └──────────┘  └──────────┘  └──────────┘                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Implementation**:

```typescript
// src/vs/workbench/contrib/pear/browser/conversationPanel/conversationPanel.tsx

export const ConversationPanel: React.FC = () => {
  const { messages, isStreaming, streamingContent } = useConversation();
  const { actions, executeAction } = useWorkflowActions();
  const [inputValue, setInputValue] = useState('');
  
  const handleSend = async () => {
    if (!inputValue.trim()) return;
    await executeAction({ type: 'user_message', content: inputValue });
    setInputValue('');
  };
  
  return (
    <div className="conversation-panel">
      <MessageList 
        messages={messages} 
        isStreaming={isStreaming}
        streamingContent={streamingContent}
      />
      
      <InputArea 
        value={inputValue}
        onChange={setInputValue}
        onSend={handleSend}
        disabled={isStreaming}
      />
      
      <ActionBar 
        actions={actions}
        onAction={executeAction}
        disabled={isStreaming}
      />
    </div>
  );
};

// Message component with code highlighting
const Message: React.FC<{ message: ChatMessage }> = ({ message }) => {
  const rendered = useMemo(() => {
    return renderMarkdown(message.content, {
      codeHighlight: true,
      syntaxTheme: 'vs-dark',
    });
  }, [message.content]);
  
  return (
    <div className={`message message--${message.role}`}>
      <div className="message__header">
        <span className="message__author">
          {message.role === 'assistant' ? '🍐 Pear' : '👤 You'}
        </span>
        <span className="message__time">{formatTime(message.timestamp)}</span>
      </div>
      <div 
        className="message__content" 
        dangerouslySetInnerHTML={{ __html: rendered }}
      />
    </div>
  );
};
```

### 3. Lock Status Indicator

Show lock status in file explorer and status bar.

```typescript
// src/vs/workbench/contrib/pear/browser/lockStatus/lockDecorations.ts

import { IDecorationsProvider } from 'vs/workbench/services/decorations/common/decorations';

export class LockDecorationsProvider implements IDecorationsProvider {
  readonly label = 'Pear Locks';
  
  constructor(
    @ILockService private readonly lockService: ILockService
  ) {}
  
  provideDecorations(uri: URI): IDecorationData | undefined {
    const path = uri.fsPath;
    const lock = this.lockService.getLocker(path);
    
    if (!lock) return undefined;
    
    if (lock.sessionId === this.getCurrentSessionId()) {
      return {
        badge: '✎',
        tooltip: 'You have write lock',
        color: 'green',
      };
    } else {
      return {
        badge: '🔒',
        tooltip: `Locked by ${lock.user} (${lock.feature})`,
        color: 'yellow',
      };
    }
  }
}
```

### 4. Onboarding Flow

First-time setup for a project.

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│                         🍐 Welcome to Pear                          │
│                                                                     │
│   I'll help you set up Pear for this project.                       │
│                                                                     │
│   Analyzing project structure...                                    │
│                                                                     │
│   ┌───────────────────────────────────────────────────────────────┐ │
│   │ ✓ Detected: TypeScript project                                │ │
│   │ ✓ Detected: Test framework: Vitest                            │ │
│   │ ✓ Detected: Package manager: pnpm                             │ │
│   │ ✓ Found 47 source files across 12 directories                 │ │
│   └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│   Configuration                                                     │
│   ┌───────────────────────────────────────────────────────────────┐ │
│   │ Features directory:  [src/features           ] [Browse]       │ │
│   │ Shared code:         [src/shared             ] [Browse]       │ │
│   │ Test command:        [pnpm test              ]                │ │
│   └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│   LLM API Key                                                       │
│   ┌───────────────────────────────────────────────────────────────┐ │
│   │ Anthropic API Key:   [sk-ant-...             ] [Verify]       │ │
│   │ ✓ Key verified successfully                                   │ │
│   └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│                              [Get Started]                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Core Integration

### Integrating @pear/core

```typescript
// src/vs/workbench/contrib/pear/node/pearEngine.ts

import { PearEngine } from '@pear/core';
import { IPearEngineService } from '../common/pear';

export class PearEngineService implements IPearEngineService {
  readonly _serviceBrand: undefined;
  
  private _engine: PearEngine | null = null;
  private _currentSession: PearSession | null = null;
  
  private readonly _onSessionChange = new Emitter<PearSession | null>();
  readonly onSessionChange = this._onSessionChange.event;
  
  constructor(
    @IConfigurationService private readonly configService: IConfigurationService,
    @IWorkspaceContextService private readonly workspaceService: IWorkspaceContextService,
  ) {}
  
  async initialize(): Promise<void> {
    const workspaceRoot = this.workspaceService.getWorkspace().folders[0]?.uri.fsPath;
    if (!workspaceRoot) return;
    
    const apiKey = await this.configService.getValue<string>('pear.anthropicApiKey');
    if (!apiKey) return;
    
    this._engine = new PearEngine({
      pearDir: path.join(workspaceRoot, '.pear'),
      anthropic: { apiKey },
    });
  }
  
  get engine(): PearEngine {
    if (!this._engine) {
      throw new Error('PearEngine not initialized');
    }
    return this._engine;
  }
  
  get currentSession(): PearSession | null {
    return this._currentSession;
  }
  
  async createSession(options: CreateSessionOptions): Promise<PearSession> {
    this._currentSession = await this.engine.createSession(options);
    this._onSessionChange.fire(this._currentSession);
    return this._currentSession;
  }
  
  async resumeSession(sessionId: string): Promise<PearSession | null> {
    this._currentSession = await this.engine.resumeSession(sessionId);
    this._onSessionChange.fire(this._currentSession);
    return this._currentSession;
  }
}

// Registration
registerSingleton(IPearEngineService, PearEngineService);
```

### Workflow Event Handling

```typescript
// src/vs/workbench/contrib/pear/browser/workflowController.ts

export class WorkflowController {
  constructor(
    @IPearEngineService private readonly pearEngine: IPearEngineService,
    @IWorkflowStateService private readonly stateService: IWorkflowStateService,
    @IConversationService private readonly conversationService: IConversationService,
  ) {}
  
  async runCurrentPhase(): Promise<void> {
    const session = this.pearEngine.currentSession;
    if (!session) return;
    
    const phase = session.getCurrentPhase();
    
    switch (phase) {
      case 'planning':
        await this.runPlanning();
        break;
      case 'interface':
        await this.runInterface();
        break;
      case 'implementation':
        await this.runImplementation();
        break;
      case 'testing':
        await this.runTesting();
        break;
    }
  }
  
  private async runPlanning(): Promise<void> {
    const session = this.pearEngine.currentSession!;
    
    this.conversationService.startStreaming();
    
    for await (const event of session.runPlanning(this.getDescription())) {
      switch (event.type) {
        case 'llm_chunk':
          this.conversationService.appendStreamingContent(event.content);
          break;
        case 'awaiting_approval':
          this.conversationService.endStreaming();
          this.stateService.setAwaitingApproval('planning');
          break;
      }
    }
  }
  
  // ... similar for other phases
}
```

---

## Custom Features

### 1. Phase-Aware Keybindings

```typescript
// src/vs/workbench/contrib/pear/browser/keybindings.ts

// Define keybindings that change based on phase
KeybindingsRegistry.registerCommandAndKeybindingRule({
  id: 'pear.approve',
  weight: KeybindingWeight.WorkbenchContrib,
  when: ContextKeyExpr.equals('pear.awaitingApproval', true),
  primary: KeyMod.CtrlCmd | KeyCode.Enter,
  handler: (accessor) => {
    const controller = accessor.get(IWorkflowController);
    controller.approveCurrentStep();
  }
});

KeybindingsRegistry.registerCommandAndKeybindingRule({
  id: 'pear.modify',
  weight: KeybindingWeight.WorkbenchContrib,
  when: ContextKeyExpr.equals('pear.awaitingApproval', true),
  primary: KeyMod.CtrlCmd | KeyMod.Shift | KeyCode.Enter,
  handler: (accessor) => {
    const controller = accessor.get(IWorkflowController);
    controller.requestModification();
  }
});

KeybindingsRegistry.registerCommandAndKeybindingRule({
  id: 'pear.goBack',
  weight: KeybindingWeight.WorkbenchContrib,
  when: ContextKeyExpr.equals('pear.hasActiveSession', true),
  primary: KeyMod.CtrlCmd | KeyCode.Backspace,
  handler: (accessor) => {
    const controller = accessor.get(IWorkflowController);
    controller.goBackToPreviousPhase();
  }
});
```

### 2. Restricted Edit Mode

During certain phases, prevent direct file editing:

```typescript
// src/vs/workbench/contrib/pear/browser/editRestriction.ts

export class EditRestrictionService {
  constructor(
    @IWorkflowStateService private readonly stateService: IWorkflowStateService,
    @IEditorService private readonly editorService: IEditorService,
  ) {
    this.stateService.onStateChange(this.updateRestrictions.bind(this));
  }
  
  private updateRestrictions(state: WorkflowState): void {
    const phase = state.currentPhase;
    const status = state.phaseStatus;
    
    // During implementation review, make file read-only
    if (phase === 'implementation' && status === 'awaiting_approval') {
      this.setEditorReadOnly(true);
    } else {
      this.setEditorReadOnly(false);
    }
  }
  
  private setEditorReadOnly(readOnly: boolean): void {
    const editors = this.editorService.visibleTextEditorControls;
    for (const editor of editors) {
      editor.updateOptions({ readOnly });
    }
  }
}
```

### 3. Integrated Diff View

When reviewing implementation, show diff:

```typescript
// src/vs/workbench/contrib/pear/browser/reviewDiffEditor.ts

export class ReviewDiffEditorService {
  async showImplementationReview(
    unit: ImplementationUnit,
    proposedCode: string
  ): Promise<void> {
    const originalUri = URI.file(unit.filePath);
    const proposedUri = URI.parse(`pear-proposed:${unit.filePath}`);
    
    // Register proposed content provider
    this.textModelService.registerTextModelContentProvider('pear-proposed', {
      provideTextContent: () => Promise.resolve(
        this.modelService.createModel(proposedCode, null, proposedUri)
      )
    });
    
    // Open diff editor
    await this.editorService.openEditor({
      original: { resource: originalUri },
      modified: { resource: proposedUri },
      label: `Review: ${unit.name}`,
      options: {
        renderSideBySide: true,
      }
    });
  }
}
```

---

## Build & Distribution

### Build Configuration

```typescript
// build/pear/product.json
{
  "nameShort": "Pear",
  "nameLong": "Pear Editor",
  "applicationName": "pear",
  "dataFolderName": ".pear-editor",
  "win32MutexName": "peareditor",
  "licenseName": "MIT",
  "licenseUrl": "https://github.com/pear/editor/blob/main/LICENSE",
  "serverLicenseUrl": "https://github.com/pear/editor/blob/main/LICENSE",
  "win32DirName": "Pear",
  "win32NameVersion": "Pear",
  "win32RegValueName": "PearEditor",
  "win32AppId": "{{PEAR-EDITOR-UUID}}",
  "win32x64AppId": "{{PEAR-EDITOR-UUID-X64}}",
  "darwinBundleIdentifier": "com.pear.editor",
  "urlProtocol": "pear"
}
```

### Build Scripts

```bash
# scripts/build-macos.sh

#!/bin/bash
set -e

echo "Building Pear for macOS..."

# Build the app
yarn gulp vscode-darwin-x64-min

# Sign the app (requires Apple Developer account)
codesign --deep --force --verify --verbose \
  --sign "Developer ID Application: Your Name" \
  "../VSCode-darwin-x64/Pear.app"

# Create DMG
create-dmg \
  --volname "Pear" \
  --window-pos 200 120 \
  --window-size 600 400 \
  --icon-size 100 \
  --icon "Pear.app" 150 180 \
  --app-drop-link 450 180 \
  "Pear-darwin-x64.dmg" \
  "../VSCode-darwin-x64/Pear.app"

echo "Build complete!"
```

### Branding Assets

```
resources/
├── darwin/
│   ├── pear.icns              # macOS app icon
│   └── dmg-background.png     # DMG background
├── linux/
│   └── pear.png
├── win32/
│   └── pear.ico
└── web/
    ├── favicon.ico
    └── pear-logo.svg
```

---

## Implementation Plan

### Week 1-2: Fork Setup

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-2 | Fork VS Code, set up build | Building successfully |
| 3-4 | Custom branding, product.json | Branded app |
| 5 | Remove unused extensions | Smaller bundle |
| 6-7 | Service registration setup | Pear services scaffold |
| 8-10 | @pear/core integration | Can call core library |

### Week 3-4: UI Components

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-3 | Workflow Sidebar | Sidebar displays state |
| 4-6 | Conversation Panel | Chat works |
| 7-8 | Connect to workflow events | Streaming works |
| 9-10 | Action buttons | Approve/Modify/Back work |

### Week 5-6: Workflow Integration

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-2 | Phase transitions | All phases navigate correctly |
| 3-4 | Phase-aware keybindings | Cmd+Enter approves |
| 5-6 | Edit restrictions | Read-only during review |
| 7-8 | Diff view for review | Side-by-side diffs |
| 9-10 | Bug fixes, polish | Stable workflow |

### Week 7-8: Polish & Distribution

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-2 | Lock status display | Lock indicators |
| 3-4 | Onboarding flow | First-run setup |
| 5-6 | Session recovery UI | Resume prompts |
| 7-8 | macOS build & sign | DMG ready |
| 9-10 | Testing, final polish | Release candidate |

---

## Testing Strategy

### Unit Tests

```typescript
// Test Pear services
describe('PearEngineService', () => {
  it('should initialize with workspace', async () => {});
  it('should create session', async () => {});
  it('should resume session', async () => {});
});

describe('WorkflowStateService', () => {
  it('should reflect session state', () => {});
  it('should emit state changes', () => {});
});
```

### Integration Tests

```typescript
// Test full workflow in editor
describe('Editor Workflow Integration', () => {
  it('should complete planning phase', async () => {});
  it('should navigate between phases', async () => {});
  it('should handle session recovery', async () => {});
});
```

### Manual Testing Checklist

- [ ] Fresh install experience
- [ ] Onboarding flow
- [ ] Create new session
- [ ] Complete all phases
- [ ] Resume interrupted session
- [ ] Keybindings work correctly
- [ ] Diff view during review
- [ ] Lock indicators visible
- [ ] Conversation streaming smooth
- [ ] No crashes during normal use

---

## Risk Mitigation

### Risk 1: VS Code Upstream Changes

**Risk**: VS Code updates break our customizations

**Mitigation**:
- Keep Pear code isolated in `contrib/pear/`
- Minimal changes to core VS Code files
- Document all VS Code modifications
- Quarterly rebase schedule

### Risk 2: Build Complexity

**Risk**: VS Code build system is complex

**Mitigation**:
- Start with working fork before customizing
- Document build process thoroughly
- Set up CI early
- Have rollback plan

### Risk 3: Performance

**Risk**: Custom components slow down editor

**Mitigation**:
- Profile early and often
- Use React.memo and virtualization
- Lazy load Pear components
- Keep conversation history bounded

### Risk 4: Distribution Challenges

**Risk**: Code signing, notarization issues

**Mitigation**:
- Set up Apple Developer account early
- Test notarization on each build
- Have fallback unsigned distribution

---

## Exit Criteria

Phase 3 is complete when:

1. **Functional**: All phases work in the editor
2. **Stable**: No crashes during normal use
3. **Distributed**: macOS app is downloadable
4. **Documented**: User guide written
5. **Tested**: Manual testing complete

### Deliverables

- [ ] Pear Editor macOS application
- [ ] Installation DMG
- [ ] User documentation
- [ ] Build scripts and CI
- [ ] Known issues documented

### Questions to Answer

After completing Phase 3:

1. Is the UX significantly better than CLI?
2. Which features need the most polish?
3. What's missing for daily use?
4. Is build/distribution sustainable?
5. Ready for Phase 4 features?

