# Pear 🍐

**A Structured Pair-Programming AI Coding Tool**

Pear is an AI coding editor designed for developers who want the productivity benefits of LLM-assisted coding while maintaining tight control over their codebase. It implements a structured, phase-based workflow that keeps humans in the loop at every critical decision point.

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Solution Overview](#solution-overview)
3. [Core Principles](#core-principles)
4. [Architecture](#architecture)
5. [Onboarding & Project Analysis](#onboarding--project-analysis)
6. [Parallel Sessions & Folder Locking](#parallel-sessions--folder-locking)
7. [Session Management](#session-management)
8. [Detailed Workflow](#detailed-workflow)
9. [Technical Design](#technical-design)
10. [Tech Stack](#tech-stack)
11. [Considerations](#considerations)
12. [Limitations](#limitations)
13. [Future Roadmap](#future-roadmap)
14. [Appendix](#appendix)

---

## Problem Statement

Current LLM coding tools exist at two extremes:

### Level 1: Tab Completion
- Human maintains full control
- LLM completes individual lines/snippets
- Low productivity gain, but predictable

### Level 2: Autonomous Agents (e.g., Cursor Agent Mode)
- LLM operates autonomously for extended periods
- High productivity potential, but introduces critical problems:

| Problem | Description |
|---------|-------------|
| **Long Feedback Loops** | Agent works for minutes before human review. Human must context-switch, leave, return, and comprehend large changesets. |
| **Context Loss** | As the agent makes decisions, the human loses track of the application's growing complexity. |
| **Design Drift** | Agent makes architectural decisions that diverge from human intent. Human often accepts suboptimal solutions because "it works." |
| **Cognitive Load** | Reviewing large autonomous changes requires significant mental effort, leading to rubber-stamping rather than true review. |

**The Gap**: There is no tool optimized for developers who want LLM assistance with tight feedback loops and continuous human control.

---

## Solution Overview

Pear implements a **structured pair-programming workflow** where:

- **Human = Passenger**: Provides direction, makes decisions, approves progress
- **LLM = Driver**: Writes code, suggests approaches, handles implementation details

The workflow is divided into **four distinct phases**, each with clear entry/exit criteria and mandatory human checkpoints:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Phase 1        Phase 2         Phase 3         Phase 4       │
│  ┌────────┐    ┌──────────┐    ┌────────────┐   ┌─────────┐    │
│  │Planning│───▶│Interface │───▶│Implementation│─▶│Testing │    │
│  │        │    │& Tests   │    │             │   │        │    │
│  └────────┘    └──────────┘    └────────────┘   └─────────┘    │
│       │              │               │               │          │
│       ▼              ▼               ▼               ▼          │
│   [Human ✓]      [Human ✓]       [Human ✓]      [Human ✓]      │
│                                   (per unit)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Principles

### 1. **Tight Feedback Loops**
No autonomous work exceeds a single, reviewable unit. The LLM implements one function, one test, one type—then pauses for human review.

### 2. **Progressive Disclosure**
Each phase builds on the previous. Decisions made early (design, interfaces) constrain later phases, preventing design drift.

### 3. **Explicit Checkpoints**
Human approval is required to transition between phases. This is not a soft suggestion—the workflow enforces gates.

### 4. **Artifact-Driven**
Each phase produces concrete artifacts (design docs, type definitions, tests, implementations) that persist in the repository.

### 5. **Reversibility**
Any decision can be revisited. The human can roll back to an earlier phase and make different choices.

### 6. **Locality of Behavior**
Design documents live alongside the code they describe. When you open a feature folder, everything about that feature—design rationale, types, implementation, tests—is in one place.

### 7. **Modular Isolation**
Each feature is its own module with explicit boundaries. This enables safe parallel development and prevents unintended coupling.

---

## Architecture

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         PEAR EDITOR                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐   │
│  │   Editor    │  │  Workflow   │  │    Conversation        │   │
│  │   Pane      │  │  Sidebar    │  │    Panel               │   │
│  │             │  │             │  │                        │   │
│  │  [Code]     │  │  Phase: 1/4 │  │  Human: "Add auth"     │   │
│  │             │  │  ──────────  │  │  LLM: "I suggest..."   │   │
│  │             │  │  ☑ Planning │  │  Human: "Yes, but..."  │   │
│  │             │  │  ☐ Interface│  │  LLM: "Got it..."      │   │
│  │             │  │  ☐ Implement│  │                        │   │
│  │             │  │  ☐ Testing  │  │  [Approve & Continue]  │   │
│  │             │  │             │  │  [Request Changes]     │   │
│  │             │  │  Artifacts: │  │  [Go Back]             │   │
│  │             │  │  - design.md│  │                        │   │
│  │             │  │  - types.ts │  │                        │   │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘   │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                         CORE ENGINE                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
│  │  Workflow    │  │   Context    │  │     LLM              │    │
│  │  State       │  │   Manager    │  │     Orchestrator     │    │
│  │  Machine     │  │              │  │                      │    │
│  └──────────────┘  └──────────────┘  └──────────────────────┘    │
│          │                │                    │                 │
│          ▼                ▼                    ▼                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
│  │  Artifact    │  │  Repository  │  │     LLM Provider     │    │
│  │  Store       │  │  Interface   │  │     Adapter          │    │
│  │  (File-based)│  │  (Git)       │  │  (OpenAI/Anthropic)  │    │
│  └──────────────┘  └──────────────┘  └──────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Component Breakdown

#### 1. Editor Layer
- **Editor Pane**: Standard code editor with syntax highlighting, LSP support
- **Workflow Sidebar**: Visual progress tracker showing current phase, completed steps, and artifacts
- **Conversation Panel**: Chat interface for human-LLM dialogue with structured action buttons

#### 2. Core Engine

| Component | Responsibility |
|-----------|----------------|
| **Workflow State Machine** | Manages phase transitions, enforces gates, tracks progress |
| **Context Manager** | Maintains conversation history, relevant code context, design decisions |
| **LLM Orchestrator** | Constructs prompts, manages LLM interactions, parses responses |
| **Artifact Store** | Persists design docs, types, tests in structured repository locations |
| **Repository Interface** | Git operations, file system access, change tracking |
| **LLM Provider Adapter** | Abstraction layer for multiple LLM backends |
| **Lock Manager** | Manages folder-level write locks for parallel sessions |
| **Session Runner** | Detects incomplete sessions, handles recovery, guides resumption |

---

## Onboarding & Project Analysis

Before using Pear on a codebase, an **analysis phase** creates a representation of the project structure. This enables Pear to understand the existing architecture and integrate new features appropriately.

### First-Time Setup

When Pear is opened on a project for the first time:

```
[Pear]: I don't see a Pear configuration for this project. 
        Let me analyze the codebase structure.

        Scanning...
        ✓ Detected: TypeScript project (tsconfig.json)
        ✓ Detected: Test framework: Vitest (vitest.config.ts)
        ✓ Detected: Package manager: pnpm
        ✓ Found 47 source files across 12 directories
        
        Generating project map...
        ✓ Created: .pear/project-map.yaml
        ✓ Created: .pear/config.yaml (please review)
        
        [Review Configuration] [Start Feature]
```

### Project Map

The analysis produces a **project map** stored in `.pear/project-map.yaml`:

```yaml
# .pear/project-map.yaml
# Auto-generated by Pear - can be manually edited

version: 1
analyzedAt: 2025-01-15T10:00:00Z

structure:
  type: standard  # standard | monorepo | custom
  
  # Where features should be created
  featuresRoot: src/features
  
  # Shared code locations (require write locks)
  sharedRoots:
    - src/shared
    - src/utils
    - src/types
  
  # Entry points and important files
  entryPoints:
    - src/index.ts
    - src/app.ts
  
  # Detected patterns
  patterns:
    testLocation: colocated  # colocated | separate
    testFilePattern: "*.test.ts"
    typeLocation: colocated  # colocated | separate

# Existing modules/features detected
existingModules:
  - path: src/features/users
    description: "User management (auto-detected)"
  - path: src/features/products  
    description: "Product catalog (auto-detected)"
    
# Dependencies between modules (auto-detected from imports)
moduleDependencies:
  src/features/users:
    - src/shared/database
    - src/shared/types
  src/features/products:
    - src/shared/database
    - src/features/users  # products import from users
```

### Benefits of Project Analysis

1. **Context for LLM**: The project map helps the LLM understand where new code should go
2. **Dependency awareness**: Pear knows which modules depend on which
3. **Consistent style**: New features follow existing patterns
4. **Smart defaults**: Test framework, file patterns auto-detected

### Updating the Project Map

The project map can be:
- **Auto-refreshed**: `Pear: Refresh Project Map` command
- **Manually edited**: Human can add descriptions, correct misdetections
- **Extended**: As new features are created, they're added to the map

---

## Parallel Sessions & Folder Locking

Pear supports multiple concurrent development sessions through a folder-level locking mechanism. This enables teams to work on multiple features simultaneously while preventing conflicts.

### Why Folder Locking?

1. **Safe parallelism**: Multiple features can be developed simultaneously without merge conflicts
2. **Bounded context**: Each LLM only sees/modifies its designated folder, reducing context size
3. **Enforced modularity**: Features must be self-contained to work with Pear
4. **Team scalability**: Multiple developers can run Pear sessions concurrently

### Locking Model

```
┌─────────────────────────────────────────────────────────────────┐
│                     REPOSITORY                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  src/features/                                                  │
│  ├── auth/          ← Session A has WRITE lock                  │
│  │                    (Session B can READ from here)            │
│  ├── payments/      ← Session B has WRITE lock                  │
│  │                    (Session A can READ from here)            │
│  ├── notifications/ ← No write lock (available)                 │
│  │                                                              │
│  src/shared/                                                    │
│  ├── utils/         ← No current writer                         │
│  │                    Session A: READ                           │
│  │                    Session B: READ                           │
│  │                    (write requires QUEUED_WRITE)             │
│  ├── types/         ← Same as utils                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Cross-feature dependencies work naturally**: If Feature B (payments) needs to import from Feature A (auth), it can do so freely. Feature B just can't *modify* auth's files.

### Lock Types

| Lock Type | Behavior |
|-----------|----------|
| **WRITE** | Session can read and write to this folder. Only one session can hold a WRITE lock. Other sessions can still READ. |
| **READ** | Session can read from a folder. Multiple sessions can hold READ locks simultaneously. Automatically granted. |
| **QUEUED_WRITE** | Session wants to write to a shared folder. Waits in queue for current writer to finish. |

**Key insight**: WRITE locks are exclusive for *writing* but not for *reading*. If Feature A has a WRITE lock on `src/features/auth/`, Feature B can still import from and read `src/features/auth/`—it just can't modify those files.

### Lock Acquisition Protocol

```typescript
interface FolderLock {
  path: string;
  sessionId: string;
  type: 'write' | 'queued_write';
  acquiredAt: Date;
  queuePosition?: number;  // For queued_write
}

interface LockManager {
  // Called when session starts Phase 1 (Planning)
  // Acquires WRITE lock on feature folder
  acquireFeatureWriteLock(featurePath: string, sessionId: string): Promise<void>;
  
  // Called when session needs to modify shared resource
  // If another session has write lock, enters queue
  requestSharedWrite(sharedPath: string, sessionId: string): Promise<void>;
  
  // Called when session completes or is abandoned
  releaseAll(sessionId: string): Promise<void>;
  
  // Note: READ access is implicit and doesn't require lock acquisition
  // Any session can read from any folder at any time
}
```

### Workflow Integration

**Phase 1 (Planning)**:
- User specifies feature folder: `src/features/auth/`
- Pear acquires WRITE lock on that folder
- If write-locked by another session, user is notified and can:
  - Wait for lock release
  - Choose different feature folder
  - Force takeover (if same user's abandoned session)

**Phase 2-4 (Interface, Implementation, Testing)**:
- LLM can only *write* code within the locked folder
- LLM can freely *read* and import from any folder (including other features)
- If LLM needs to *add* something to a shared folder:
  - Request QUEUED_WRITE on that folder
  - Wait for any current writer to finish
  - Or create a feature-local version instead
- User is notified when write contention occurs

**Cross-Feature Imports**:
- Feature B can import from Feature A even if A has a write lock
- This is safe because B is only reading, not modifying A's code
- If A's API changes while B is developing, B will see the latest version on next read

### Handling Shared Folder Contention

When the LLM needs to modify a shared folder that another session is using:

```
[Pear]: I need to add a helper to `src/shared/utils/strings.ts`.
        Session B (payments) currently has a write lock on this folder.
        
        [Wait for lock] [Create local utility instead] [Skip for now]
```

**Option A: Wait for lock** — Queue up for exclusive access  
**Option B: Create feature-local utility** — Create `src/features/auth/utils/strings.ts` instead  
**Option C: Skip for now** — Continue with other work, revisit later

Feature-local utilities can be promoted to shared during a consolidation phase after the feature is complete.

### Lock File Structure

```yaml
# .pear/locks.yaml

locks:
  - path: src/features/auth
    sessionId: abc123
    type: write
    user: jake
    acquiredAt: 2025-01-15T10:00:00Z
    feature: "User Authentication"
    phase: implementation  # Current phase
    
  - path: src/features/payments
    sessionId: def456
    type: write
    user: jake
    acquiredAt: 2025-01-15T10:30:00Z
    feature: "Payment Processing"
    phase: planning

writeQueue:
  - path: src/shared/utils
    queue:
      - sessionId: abc123
        requestedAt: 2025-01-15T11:00:00Z
        reason: "Add string helper for auth tokens"
      - sessionId: def456
        requestedAt: 2025-01-15T11:02:00Z
        reason: "Add currency formatting utility"
```

### Benefits

1. **Zero write conflicts**: Only one session can write to a folder at a time
2. **Cross-feature imports work**: Sessions can read from any folder, enabling natural dependencies
3. **Predictable isolation**: Each session has a clear write boundary
4. **Forced good architecture**: Features must be modular to work with Pear
5. **Parallelism scales**: Team of 5 can run 5 sessions on 5 features
6. **Reduced context size**: LLM only needs to understand its feature folder
7. **Audit trail**: Lock history shows who worked on what, when

---

## Session Management

Pear includes a **Session Runner** that manages the lifecycle of development sessions, handles recovery from crashes, and guides users through resumption of incomplete work.

### Session States

```typescript
type SessionState = 
  | 'active'      // Currently being worked on
  | 'paused'      // Explicitly paused by user
  | 'interrupted' // Crashed or closed unexpectedly
  | 'complete'    // All phases finished successfully
  | 'cancelled';  // Explicitly abandoned by user
```

### Session Recovery

When Pear starts, the Session Runner checks for incomplete sessions:

```
[Pear]: I found an incomplete session from your last use.

        📁 Feature: User Authentication
        📍 Phase: 3/4 - Implementation (4/7 units complete)
        ⏰ Last activity: 2 hours ago
        🔒 Lock held: src/features/auth/
        
        The session was interrupted (Pear closed unexpectedly).
        Your work has been preserved.
        
        [Resume Session] [View Progress] [Abandon & Release Lock]
```

### Automatic Checkpoints

Pear automatically saves state at key moments:
- After each human approval
- After each LLM response
- Before any file write
- Every 30 seconds during active work

This ensures minimal work loss on crash.

### Session File Structure

```yaml
# .pear/sessions/abc123.yaml

sessionId: abc123
featurePath: src/features/auth
featureName: "User Authentication"
state: interrupted
user: jake

startedAt: 2025-01-15T10:00:00Z
lastActivityAt: 2025-01-15T12:30:00Z

currentPhase: implementation
phaseProgress:
  planning:
    status: approved
    designDocPath: src/features/auth/DESIGN.md
  interface:
    status: approved
    typesPath: src/features/auth/types.ts
    testsPath: src/features/auth/__tests__/auth.test.ts
  implementation:
    status: in_progress
    units:
      - name: getOAuthUrl
        status: approved
      - name: handleCallback
        status: approved
      - name: createSession
        status: approved
      - name: validateSession
        status: in_progress  # Was working on this when interrupted
      - name: revokeSession
        status: pending
    currentUnitIndex: 3
  testing:
    status: pending

# Conversation history for context restoration
conversationCheckpoint:
  messageCount: 47
  lastMessageAt: 2025-01-15T12:30:00Z
```

### Pausing and Resuming

Users can explicitly pause a session:

```
[Human]: I need to stop for today.

[Pear]: No problem. I'll save your progress.
        
        ✓ Session paused
        ✓ State saved (Phase 3, unit 4/7)
        ✓ Lock retained on src/features/auth/
        
        You can resume anytime with "Pear: Resume Session"
        or from the Pear sidebar.
```

### Cancellation

To abandon a session without completing:

```
[Human]: [Abandon Session]

[Pear]: Are you sure you want to abandon this session?
        
        This will:
        - Release the write lock on src/features/auth/
        - Keep all files created so far (you can delete manually)
        - Archive the session history
        
        The partially-implemented feature will remain in your codebase.
        You can start a new session on this folder later.
        
        [Confirm Abandon] [Cancel]
```

### Git Integration

**Important**: Pear recommends committing changes before starting a new feature.

```
[Pear]: Before starting a new feature, I recommend committing 
        your current changes.
        
        Uncommitted files detected:
        - src/features/auth/types.ts (modified)
        - src/features/auth/auth.ts (new)
        
        [Commit Now] [Start Anyway] [Cancel]
```

Git behavior is configurable:

```yaml
# .pear/config.yaml

git:
  # Remind to commit before new features
  warnUncommittedChanges: true
  
  # Auto-commit after completing a feature (Phase 4 complete)
  autoCommitOnComplete: false  # Default: false (user controls commits)
  
  # Commit message template (if autoCommitOnComplete: true)
  commitMessageTemplate: "feat({feature}): {description}"
```

---

## Detailed Workflow

### Phase 1: Planning

**Goal**: Produce a shared understanding of the feature and a written design document.

**Entry Criteria**: Human initiates with a feature request  
**Exit Criteria**: Human approves the design document

#### Process Flow

```
Human                              LLM
  │                                 │
  │  "I want to add user auth"      │
  ├────────────────────────────────▶│
  │                                 │
  │   Clarifying questions:         │
  │   - OAuth or password-based?    │
  │   - Which providers?            │
  │   - Session duration?           │
  │◀────────────────────────────────┤
  │                                 │
  │  "OAuth with Google, 24h"       │
  ├────────────────────────────────▶│
  │                                 │
  │   Draft design document:        │
  │   - Problem statement           │
  │   - Proposed solution           │
  │   - Technical approach          │
  │   - Scope / Out of scope        │
  │◀────────────────────────────────┤
  │                                 │
  │  [Approve] / [Request Changes]  │
  └─────────────────────────────────┘
```

#### Artifact: Design Document

Location: `src/features/{feature}/DESIGN.md` (co-located with feature code)

```markdown
# Feature: User Authentication

## Status: Approved
## Created: 2025-01-15
## Session ID: abc123

## Problem Statement
The application currently has no authentication mechanism...

## Proposed Solution
Implement OAuth 2.0 authentication with Google as the identity provider...

## Technical Approach
1. Add OAuth callback endpoint
2. Create session management service
3. Add auth middleware to protected routes
4. Store sessions in Redis

## Scope
- Google OAuth integration
- Session management (24h duration)
- Protected route middleware

## Out of Scope
- Multi-provider support (future)
- Username/password authentication
- MFA
```

**Why co-location?** Following the principle of locality of behavior:
- When you open the feature folder, the design rationale is right there
- If the feature is deleted, its design doc goes with it
- New developers can understand "why" without hunting through a separate directory

#### Amending the Design Document

The design doc can be amended at any point without restarting the session:

```
[Human]: Actually, we also need to support GitHub OAuth, not just Google.

[Pear]: I'll update the design document to include GitHub OAuth.
        
        📝 Amending: src/features/auth/DESIGN.md
        
        Changes:
        + Added GitHub as additional OAuth provider
        + Updated Technical Approach to handle multiple providers
        + Adjusted Scope to include provider abstraction
        
        This change may affect:
        - Types (Phase 2): AuthConfig will need provider field
        - Implementation: Provider-specific logic needed
        
        [Approve Amendment] [Modify] [Cancel]
```

When a design is amended during later phases:
- **Phase 2 (Interface)**: Types/tests may need updates—Pear will guide this
- **Phase 3 (Implementation)**: May need to revisit completed units
- **Phase 4 (Testing)**: May need additional test cases

The workflow accommodates design evolution while maintaining structure.

---

### Phase 2: Interface Design & Tests

**Goal**: Define the contracts (types, interfaces, function signatures) and write tests before implementation. The human describes what should be tested; the LLM helps refine and expand.

**Entry Criteria**: Approved design document  
**Exit Criteria**: Human approves all type definitions and test cases

**Key principle**: API/interface design and test writing happen *separately* from implementation. This phase locks in the contracts before any implementation begins.

#### Process Flow

```
Human                              LLM
  │                                 │
  │   [Phase 2 Start]               │
  ├────────────────────────────────▶│
  │                                 │
  │   Proposed type definitions:    │
  │   - AuthConfig                  │
  │   - UserSession                 │
  │   - AuthService interface       │
  │◀────────────────────────────────┤
  │                                 │
  │   [Approve] / [Modify]          │
  ├────────────────────────────────▶│
  │                                 │
  │   "Here are the behaviors I     │
  │   want to test: login flow,     │
  │   session expiry, logout"       │
  ├────────────────────────────────▶│
  │                                 │
  │   Based on your requirements,   │
  │   here are test cases:          │
  │   - shouldRedirectToOAuth       │
  │   - shouldCreateSession         │
  │   - shouldRejectExpiredSession  │
  │                                 │
  │   I'd also suggest these        │
  │   additional cases:             │
  │   - shouldHandleOAuthError      │
  │     (handles provider errors)   │
  │   - shouldRefreshNearExpiry     │
  │     (UX: refresh before expiry) │
  │◀────────────────────────────────┤
  │                                 │
  │   [Approve] / [Add/Remove Tests]│
  └─────────────────────────────────┘
```

#### Human-Driven Test Design

The human describes the behaviors they want to test in natural language. The LLM:
1. Translates these into concrete test cases
2. Suggests additional test cases with explanations of why they're valuable
3. Writes the test skeletons (implementation pending Phase 3)

The human can accept, reject, or modify any suggested tests.

#### Artifacts

**Type Definitions**: `src/features/auth/types.ts`
```typescript
export interface AuthConfig {
  provider: 'google';
  clientId: string;
  clientSecret: string;
  callbackUrl: string;
  sessionDuration: number; // seconds
}

export interface UserSession {
  id: string;
  userId: string;
  email: string;
  createdAt: Date;
  expiresAt: Date;
}

export interface AuthService {
  initiateOAuth(): Promise<{ redirectUrl: string }>;
  handleCallback(code: string): Promise<UserSession>;
  validateSession(sessionId: string): Promise<UserSession | null>;
  revokeSession(sessionId: string): Promise<void>;
}
```

**Test Cases**: `src/features/auth/__tests__/auth.test.ts`
```typescript
describe('AuthService', () => {
  describe('initiateOAuth', () => {
    it('should return valid Google OAuth redirect URL', async () => {
      // Test implementation pending
    });
  });
  
  describe('handleCallback', () => {
    it('should create session from valid OAuth code', async () => {
      // Test implementation pending
    });
    
    it('should reject invalid OAuth code', async () => {
      // Test implementation pending
    });
  });
  
  describe('validateSession', () => {
    it('should return session for valid session ID', async () => {
      // Test implementation pending
    });
    
    it('should return null for expired session', async () => {
      // Test implementation pending
    });
  });
});
```

---

### Phase 3: Implementation

**Goal**: Implement each unit defined in Phase 2, with human review after each unit.

**Entry Criteria**: Approved types and tests  
**Exit Criteria**: All units implemented and individually approved

#### Dependency-Ordered Implementation

Units are implemented in **dependency order**: leaf nodes first, working up to the root.

```
Function Dependency Tree:
                    
    AuthService.login()          ← Implement LAST (depends on others)
           │
    ┌──────┴──────┐
    │             │
validateSession() handleCallback()  ← Implement SECOND
    │             │
    └──────┬──────┘
           │
    createSession()              ← Implement SECOND  
           │
    getOAuthUrl()                ← Implement FIRST (no internal deps)
```

**Why this order?**
1. **Testable immediately**: Each unit can be tested as soon as it's implemented
2. **No forward references**: Never implement something that calls unwritten code
3. **Natural progression**: Build understanding from simple to complex

#### Process Flow

The implementation phase is broken into atomic units. For each unit:

```
┌─────────────────────────────────────────────────────────────────┐
│                    IMPLEMENTATION LOOP                          │
│                                                                 │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐ │
│   │ Select   │───▶│ LLM      │───▶│ Human    │───▶│ Next     │ │
│   │ Unit     │    │ Implement│    │ Review   │    │ Unit     │ │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘ │
│        ▲                              │                        │
│        │                              │ [Request Changes]      │
│        │                              ▼                        │
│        │                         ┌──────────┐                  │
│        └─────────────────────────│ Revise   │                  │
│                                  └──────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Example Implementation Unit

```
Human                              LLM
  │                                 │
  │   "Implement initiateOAuth"     │
  ├────────────────────────────────▶│
  │                                 │
  │   Implementation proposal:      │
  │   ```typescript                 │
  │   async initiateOAuth() {       │
  │     const params = new URL...   │
  │     return { redirectUrl };     │
  │   }                             │
  │   ```                           │
  │                                 │
  │   Explanation:                  │
  │   - Constructs OAuth URL with   │
  │     required scopes             │
  │   - Uses config values for      │
  │     client ID and callback      │
  │◀────────────────────────────────┤
  │                                 │
  │   "Add state parameter for      │
  │    CSRF protection"             │
  ├────────────────────────────────▶│
  │                                 │
  │   Revised implementation...     │
  │◀────────────────────────────────┤
  │                                 │
  │   [Approve]                     │
  └─────────────────────────────────┘
```

#### Progress Tracking

The workflow sidebar shows implementation progress:

```
Phase 3: Implementation (3/7)

☑ getOAuthUrl()          [Approved]     ← Leaf node (no deps)
☑ createSession()        [Approved]     ← Leaf node
☑ handleCallback()       [Approved]     ← Depends on above
☐ validateSession()      [In Progress]  ← Depends on createSession
☐ revokeSession()        [Pending]
☐ authMiddleware()       [Pending]      ← Depends on validateSession
☐ AuthService.login()    [Pending]      ← Root (depends on all)
```

#### Returning to Phase 2 (Modify Tests/API)

During implementation, you may realize the API or tests need changes. You can return to Phase 2 without losing implementation progress:

```
[Human]: I realized we need to add a "remember me" parameter to the login.

[Pear]: This requires an API change. Let's update the interface.
        
        🔄 Switching to Phase 2 (Interface)
        💾 Implementation progress saved (3/7 units)
        
        Current AuthService.login signature:
        ```typescript
        login(): Promise<Session>
        ```
        
        Proposed change:
        ```typescript
        login(options?: { rememberMe?: boolean }): Promise<Session>
        ```
        
        This will affect:
        - Types: LoginOptions interface needed
        - Tests: Add rememberMe test cases
        - Implementation: Modify login() (not yet implemented)
        
        [Approve Change] [Modify] [Cancel]
```

After approving interface changes:
- Pear returns to Phase 3
- Previously-approved units that aren't affected remain approved
- Affected units are marked for re-implementation
- Human only re-approves changed units

**Key principle**: API/tests and implementation are never modified simultaneously. The flow is always: Interface → Implement → Interface → Implement...

---

### Phase 4: Testing & Verification

**Goal**: Execute all tests, debug failures, perform manual testing, and document evidence that the feature works correctly.

**Entry Criteria**: All implementation units approved  
**Exit Criteria**: All tests pass, manual verification complete

#### Test-Debug Loop

Phase 4 includes a **debug loop** for handling test failures. The human only needs to re-approve files that change.

```
┌─────────────────────────────────────────────────────────────────┐
│                     PHASE 4: TEST-DEBUG LOOP                    │
│                                                                 │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐                 │
│   │  Run     │───▶│  All     │───▶│  Manual  │───▶ Complete    │
│   │  Tests   │    │  Pass?   │    │  Tests   │                 │
│   └──────────┘    └──────────┘    └──────────┘                 │
│        ▲               │ No                                     │
│        │               ▼                                        │
│        │         ┌──────────┐    ┌──────────┐                  │
│        │         │  Debug   │───▶│  Human   │                  │
│        │         │  & Fix   │    │  Approve │                  │
│        │         └──────────┘    └──────────┘                  │
│        │                              │                         │
│        └──────────────────────────────┘                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Process Flow

```
Human                              LLM
  │                                 │
  │   [Start Testing Phase]         │
  ├────────────────────────────────▶│
  │                                 │
  │   Running unit tests...         │
  │   ✗ 10 passed, 2 failed         │
  │                                 │
  │   Failed tests:                 │
  │   ✗ shouldRejectExpiredSession  │
  │     Expected: null              │
  │     Received: Session object    │
  │                                 │
  │   ✗ shouldHandleOAuthError      │
  │     Error: Unhandled rejection  │
  │◀────────────────────────────────┤
  │                                 │
  │   [Debug Failures]              │
  ├────────────────────────────────▶│
  │                                 │
  │   Analyzing failures...         │
  │                                 │
  │   Issue 1: validateSession()    │
  │   doesn't check expiry date.    │
  │   Proposed fix:                 │
  │   ```typescript                 │
  │   if (session.expiresAt < now)  │
  │     return null;                │
  │   ```                           │
  │                                 │
  │   Issue 2: handleCallback()     │
  │   missing try-catch for OAuth.  │
  │   Proposed fix: [code...]       │
  │◀────────────────────────────────┤
  │                                 │
  │   [Approve Fixes] (2 files)     │
  ├────────────────────────────────▶│
  │                                 │
  │   ✓ Fixes applied               │
  │   Re-running tests...           │
  │   ✓ 12 passed, 0 failed         │
  │◀────────────────────────────────┤
  │                                 │
  │   All tests passing!            │
  │   Ready for manual testing.     │
  │                                 │
  │   Manual test checklist:        │
  │   ☐ Click "Login with Google"   │
  │   ☐ Complete OAuth flow         │
  │   ☐ Verify session created      │
  │   ☐ Access protected route      │
  │   ☐ Test session expiry         │
  │◀────────────────────────────────┤
  │                                 │
  │   [Record test results...]      │
  ├────────────────────────────────▶│
  │                                 │
  │   [Complete Feature]            │
  └─────────────────────────────────┘
```

#### Debug Loop Rules

1. **Only changed files need re-approval**: If fixing a bug only touches `validateSession.ts`, only that file needs human approval
2. **Tests are not modified during debug**: If the test itself is wrong, return to Phase 2 to fix the test spec first
3. **Loop until green**: The debug loop continues until all tests pass
4. **Human can skip to manual**: If a test is flaky or environment-specific, human can mark it as "skip for now"

#### Artifact: Test Evidence Document

Location: `src/features/{feature}/TEST_EVIDENCE.md` (co-located with feature code)

```markdown
# Test Evidence: User Authentication

## Automated Tests

### Unit Tests
- **Run Date**: 2025-01-15 14:32:00
- **Result**: 12/12 passed
- **Coverage**: 94%

### Integration Tests  
- **Run Date**: 2025-01-15 14:33:00
- **Result**: 5/5 passed

## Manual Tests

| Test Case | Result | Verified By | Date |
|-----------|--------|-------------|------|
| OAuth redirect works | ✓ Pass | Human | 2025-01-15 |
| Session created on callback | ✓ Pass | Human | 2025-01-15 |
| Protected routes blocked without auth | ✓ Pass | Human | 2025-01-15 |
| Session expires correctly | ✓ Pass | Human | 2025-01-15 |

## Sign-off
Feature verified and approved for merge.
```

---

### Phase 5: Completion

**Goal**: Finalize the feature, release locks, and archive the session.

**Entry Criteria**: All tests pass, manual verification complete  
**Exit Criteria**: Feature complete, session archived

#### Completion Flow

```
[Pear]: 🎉 All tests passing! Feature implementation complete.
        
        Summary:
        ├── Design: src/features/auth/DESIGN.md
        ├── Types: src/features/auth/types.ts  
        ├── Implementation: 7 units across 3 files
        ├── Tests: 12 unit tests, 5 integration tests
        └── Evidence: src/features/auth/TEST_EVIDENCE.md
        
        Ready to complete this feature?
        
        [Complete Feature] [Continue Testing] [Make Changes]

[Human]: [Complete Feature]

[Pear]: ✓ Feature "User Authentication" complete!
        
        Actions taken:
        ✓ Write lock released on src/features/auth/
        ✓ Session archived to .pear/sessions/archive/abc123.yaml
        ✓ Project map updated with new module
        
        Files created/modified:
        - src/features/auth/DESIGN.md (new)
        - src/features/auth/types.ts (new)
        - src/features/auth/auth.ts (new)
        - src/features/auth/__tests__/auth.test.ts (new)
        - src/features/auth/TEST_EVIDENCE.md (new)
        
        Recommended: Commit these changes to git.
        
        [Commit Now] [I'll commit later] [Start New Feature]
```

#### What Happens on Completion

1. **Lock released**: Other sessions can now write to this folder
2. **Session archived**: Full history preserved in `.pear/sessions/archive/`
3. **Project map updated**: New module added to `.pear/project-map.yaml`
4. **Files preserved**: All generated files remain in place
5. **Git reminder**: If `warnUncommittedChanges` is enabled, prompts to commit

#### Optional Auto-Commit

If configured, Pear can auto-commit on completion:

```yaml
# .pear/config.yaml
git:
  autoCommitOnComplete: true
  commitMessageTemplate: "feat({feature}): {description}"
```

Result:
```
[Pear]: ✓ Auto-committed: "feat(auth): Add Google OAuth authentication"
        Commit: a1b2c3d
```

---

## Technical Design

### Workflow State Machine

The core workflow is managed by a finite state machine:

```typescript
type Phase = 'planning' | 'interface' | 'implementation' | 'testing' | 'complete';

type PhaseStatus = 'pending' | 'in_progress' | 'awaiting_approval' | 'approved';

interface WorkflowState {
  sessionId: string;
  featureSlug: string;
  featurePath: string;
  currentPhase: Phase;
  phaseStatus: PhaseStatus;
  
  // Phase-specific state
  planning: {
    designDocPath: string | null;
    approved: boolean;
  };
  
  interface: {
    typeDefinitions: TypeDefinition[];
    testCases: TestCase[];
    approved: boolean;
  };
  
  implementation: {
    units: ImplementationUnit[];
    currentUnitIndex: number;
    dependencyOrder: string[];  // Unit IDs in implementation order (leaf → root)
  };
  
  testing: {
    unitTestResults: TestResult | null;
    integrationTestResults: TestResult | null;
    manualTests: ManualTest[];
    debugIterations: number;  // How many debug loops we've done
  };
  
  complete: {
    completedAt: Date | null;
    lockReleased: boolean;
    archived: boolean;
  };
}

interface ImplementationUnit {
  id: string;
  name: string;
  description: string;
  filePath: string;
  status: 'pending' | 'in_progress' | 'awaiting_review' | 'approved';
  implementation: string | null;
  reviewNotes: string[];
}
```

### Context Management

To keep the LLM informed without overwhelming context windows:

```typescript
interface ContextWindow {
  // Always included
  designDocument: string;
  typeDefinitions: string;
  currentPhase: Phase;
  
  // Phase-specific
  relevantTests?: string;
  currentUnit?: ImplementationUnit;
  previousUnits?: ImplementationUnit[]; // Last 2-3 for continuity
  
  // Conversation
  recentMessages: Message[]; // Sliding window
}

class ContextManager {
  buildContext(state: WorkflowState): ContextWindow {
    // Prioritize relevant context based on current phase
    // Summarize older context to fit within token limits
    // Only include files from locked folder + read-only imports
  }
}

// Context is naturally bounded by folder isolation:
// - LLM sees: feature folder contents + imported dependencies (read-only)
// - LLM doesn't see: other features being developed in parallel
```

### LLM Orchestration

Different phases require different prompting strategies:

```typescript
interface PhasePromptConfig {
  systemPrompt: string;
  requiredOutputFormat: 'markdown' | 'code' | 'json';
  maxResponseLength: number;
  temperature: number;
}

const PHASE_CONFIGS: Record<Phase, PhasePromptConfig> = {
  planning: {
    systemPrompt: `You are helping design a software feature. 
      Ask clarifying questions. Produce a structured design document.
      Be concise but thorough.`,
    requiredOutputFormat: 'markdown',
    maxResponseLength: 2000,
    temperature: 0.7, // More creative for brainstorming
  },
  
  interface: {
    systemPrompt: `You are defining interfaces and test cases.
      Follow the approved design document strictly.
      Human describes desired behaviors; you translate to test cases.
      Suggest additional test cases with clear rationale.
      Produce valid types and test skeletons.`,
    requiredOutputFormat: 'code',
    maxResponseLength: 3000,
    temperature: 0.3, // More precise
  },
  
  implementation: {
    systemPrompt: `You are implementing a specific unit.
      Follow the type definitions exactly.
      Implement in dependency order (leaf nodes first).
      Explain your implementation choices briefly.
      Ask if anything is unclear.`,
    requiredOutputFormat: 'code',
    maxResponseLength: 1500,
    temperature: 0.2, // Very precise
  },
  
  testing: {
    systemPrompt: `You are verifying the implementation.
      Run tests and report results clearly.
      For failures, analyze root cause and propose fixes.
      Generate manual test checklists.
      Document evidence.`,
    requiredOutputFormat: 'markdown',
    maxResponseLength: 1500,
    temperature: 0.3,
  },
  
  complete: {
    systemPrompt: `You are finalizing a completed feature.
      Summarize what was built.
      Release locks and archive session.
      Update project map.
      Remind about git commit.`,
    requiredOutputFormat: 'markdown',
    maxResponseLength: 1000,
    temperature: 0.3,
  },
};
```

### File Organization

Pear follows the principle of **locality of behavior**: everything about a feature lives together.

```
project-root/
├── .pear/
│   ├── config.yaml              # Global Pear configuration
│   ├── locks.yaml               # Folder lock state for parallel sessions
│   └── sessions/                # Ephemeral session state (not for humans)
│       └── {session-id}.json
│
├── src/
│   ├── features/
│   │   └── {feature}/           # ← Each feature is self-contained
│   │       ├── DESIGN.md        # Phase 1 artifact (co-located)
│   │       ├── TEST_EVIDENCE.md # Phase 4 artifact (co-located)
│   │       ├── types.ts         # Phase 2 artifact
│   │       ├── index.ts         # Phase 3 implementation
│   │       ├── utils/           # Feature-local utilities (optional)
│   │       └── __tests__/
│   │           └── {feature}.test.ts
│   │
│   └── shared/                  # Shared code (requires lock for writes)
│       ├── utils/
│       └── types/
```

**Key points:**
- Design docs live with their feature, not in a separate `.pear/designs/` folder
- `.pear/` only contains ephemeral state and global configuration
- Features are self-contained modules with clear boundaries
- Shared code is explicitly separate and protected by write locks

---

## Tech Stack

### Why a VS Code Fork (Not an Extension)

Pear's workflow is fundamentally different from normal code editing. An extension would be constrained by VS Code's API, while a fork provides:

| Capability | Extension | Fork |
|------------|-----------|------|
| Custom file presentation | Limited | Full control |
| Phase-aware keybindings | Hacky | Native |
| Integrated diff view | Separate panels | Seamless |
| Workflow-first layout | Bolted-on | Primary paradigm |
| Restricted edit modes | Not possible | Fully supported |
| Lock visualization | Workarounds | First-class |

**Specific fork benefits:**
1. **Custom file presentation**: During implementation review, show only the unit being reviewed
2. **Phase-aware keybindings**: `Cmd+Enter` = "approve" in review mode, normal behavior otherwise
3. **Integrated diff view**: Side-by-side "proposed vs current" without separate panels
4. **Workflow-first layout**: Phases aren't a sidebar—they're the primary navigation paradigm
5. **Restricted mode**: Prevent manual edits during certain phases to enforce workflow
6. **Lock status in UI**: Show which folders are locked and by whom

### Recommended Stack

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Editor Base** | VS Code Fork (Electron) | Full control, familiar UX, existing LSP infrastructure |
| **Language** | TypeScript | Type safety, async support, LLM library ecosystem |
| **UI Framework** | React | For custom Pear-specific UI components |
| **State Management** | Zustand | Lightweight, works well with React |
| **LLM Integration** | Vercel AI SDK | Streaming, multiple providers, good TypeScript support |
| **LLM Providers** | Anthropic Claude, OpenAI | Best reasoning capabilities for code |
| **Storage** | File-based (YAML) | Design docs in repo (git-tracked), session/lock state in YAML |
| **Default Test Framework** | Vitest | Fast, TypeScript-native (configurable per project) |

### Language Support Architecture

Pear is designed to be **language-agnostic from day one**. The LLM can write code in any language; we only need minimal per-language configuration:

```typescript
interface LanguageConfig {
  // LSP handles most language intelligence
  lspCommand: string;          // e.g., "typescript-language-server --stdio"
  
  // Testing configuration (human-configurable per project)
  testCommand: string;         // e.g., "npm test", "pytest", "go test ./..."
  testFilePattern: string;     // e.g., "*.test.ts", "test_*.py", "*_test.go"
  testFramework?: string;      // e.g., "vitest", "jest", "pytest" (for LLM context)
  
  // Optional: type extraction for interface phase
  typeFileExtensions?: string[];  // e.g., [".ts", ".d.ts"]
}

// Default configurations (can be overridden in .pear/config.yaml)
const DEFAULT_LANGUAGES: Record<string, LanguageConfig> = {
  typescript: {
    lspCommand: "typescript-language-server --stdio",
    testCommand: "npx vitest run",
    testFilePattern: "*.test.ts",
    testFramework: "vitest",  // Default for TS/JS
    typeFileExtensions: [".ts", ".d.ts"],
  },
  javascript: {
    lspCommand: "typescript-language-server --stdio",
    testCommand: "npx vitest run",
    testFilePattern: "*.test.js",
    testFramework: "vitest",  // Default for TS/JS
  },
  python: {
    lspCommand: "pyright-langserver --stdio",
    testCommand: "pytest",
    testFilePattern: "test_*.py",
    testFramework: "pytest",
  },
  go: {
    lspCommand: "gopls",
    testCommand: "go test ./...",
    testFilePattern: "*_test.go",
    testFramework: "go test",
  },
  rust: {
    lspCommand: "rust-analyzer",
    testCommand: "cargo test",
    testFilePattern: "*_test.rs",
    testFramework: "cargo test",
  },
};
```

**Test Framework is Configurable**: While Vitest is the recommended default for TypeScript/JavaScript projects, the human can configure any test framework:

```yaml
# .pear/config.yaml
languages:
  typescript:
    testCommand: "npm run test"  # Custom command
    testFilePattern: "*.spec.ts"  # Different pattern
    testFramework: "jest"         # Different framework
```

**Why this works:**
- The LLM is inherently language-agnostic (trained on all languages)
- LSP (Language Server Protocol) provides language intelligence for any language with an LSP server
- Test framework is configurable per project—Pear auto-detects when possible
- The `testFramework` field helps the LLM write framework-appropriate test syntax

### Extension Points

```typescript
interface LLMProviderPlugin {
  name: string;
  chat(messages: Message[], config: PromptConfig): AsyncIterable<string>;
  embeddings?(text: string): Promise<number[]>;
}

interface LanguagePlugin {
  language: string;
  lspCommand: string;
  testCommand: string;
  testFilePattern: string;
  // Optional hooks for language-specific behavior
  parseTypes?(code: string): TypeDefinition[];
  generateTestSkeleton?(types: TypeDefinition[]): string;
}
```

---

## Considerations

### Design Tradeoffs

#### 1. VS Code Fork vs. Extension

**Decision**: Pear is a VS Code fork, not an extension.

**Tradeoff**:
- ✓ Full control over UI and workflow integration
- ✓ Can enforce workflow restrictions (e.g., prevent edits during review)
- ✓ Better UX for phase-based navigation
- ✗ Higher maintenance burden (must merge upstream VS Code changes)
- ✗ Users must switch editors (can't use existing VS Code installation)
- ✗ Larger initial development effort

**Rationale**: The structured workflow is opinionated enough that extension API limitations would create friction. A seamless, purpose-built experience outweighs the maintenance cost.

#### 2. Rigidity vs. Flexibility

**Decision**: Pear enforces a rigid phase structure.

**Tradeoff**: 
- ✓ Prevents design drift
- ✓ Creates predictable workflow
- ✗ May feel restrictive for small changes
- ✗ Overhead for trivial features

**Mitigation**: Allow "quick mode" for changes under a complexity threshold, but encourage full workflow for substantial features.

#### 3. Locality of Behavior vs. Centralized Artifacts

**Decision**: Design docs live in feature folders (`src/features/auth/DESIGN.md`), not in a central `.pear/designs/` folder.

**Tradeoff**:
- ✓ Everything about a feature is in one place
- ✓ Delete a feature, delete its design doc
- ✓ More discoverable for new developers
- ✗ Design docs are in source tree (some may prefer separation)
- ✗ Multiple DESIGN.md files across repo

**Rationale**: Locality of behavior is a proven architectural principle. The benefits of co-location outweigh the aesthetic preference for separation.

#### 4. Folder Locking for Parallel Sessions

**Decision**: Features are isolated via folder-level write locks.

**Tradeoff**:
- ✓ Enables safe parallel development
- ✓ Prevents merge conflicts
- ✓ Forces modular architecture
- ✗ Contention on shared folders can block progress
- ✗ Requires discipline around folder boundaries
- ✗ Some workflows (cross-cutting changes) don't fit this model

**Mitigation**: 
- Feature-local utilities can be promoted to shared later
- Consolidation phase for post-implementation refactoring
- Lock timeouts prevent abandoned sessions from blocking indefinitely

#### 5. Context Window Management

**Challenge**: Design documents, types, tests, and conversation history can exceed context limits.

**Strategy**:
- Prioritize current-phase context
- Summarize rather than truncate older context
- Keep design document always in context (it's the "source of truth")
- Use embeddings for semantic retrieval when needed
- Folder isolation naturally bounds context (LLM only sees its feature folder)

#### 6. Unit Granularity

**Challenge**: What constitutes a "unit" for implementation review?

**Heuristics**:
- One public function = one unit
- One test file = one review
- Complex functions may be split further
- Human can request finer granularity

#### 7. Test-First Enforcement

**Decision**: Tests are written before implementation (TDD-style).

**Rationale**:
- Prevents tests from being retrofitted to pass
- Forces interface thinking before implementation
- Tests serve as specification

### Security Considerations

| Concern | Mitigation |
|---------|------------|
| API key exposure | Store in OS keychain, never in config files |
| Code exfiltration | All LLM calls logged, user can review what's sent |
| Prompt injection | Sanitize user input, use structured message formats |
| Session hijacking | Session IDs cryptographically random, local storage only |

### Performance Considerations

| Operation | Target | Strategy |
|-----------|--------|----------|
| LLM response | <5s first token | Streaming responses |
| File operations | <100ms | Async, batch where possible |
| State persistence | <50ms | Write-ahead log, periodic flush |
| Context building | <200ms | Pre-compute, cache summaries |

---

## Limitations

### 1. Language Support

**Status**: Language-agnostic by design

Pear works with any language that has:
- An LSP server (for editor intelligence)
- A test runner (for Phase 4)

**Out-of-the-box support**: TypeScript, Python, Go, Rust  
**Extensible**: Any language with LSP can be added via configuration

**Caveat**: LLM output quality varies by language due to training data distribution. Languages with more open-source code (Python, JS/TS, Go) tend to produce better results than niche languages.

### 2. Project Size

**Limitation**: Works best for small-to-medium projects

**Challenge**: Large monorepos may have too much context for effective LLM reasoning

**Mitigation**: 
- Focus context on feature-specific directories
- Use .pearignore to exclude irrelevant paths

### 3. One Feature Per Session (But Parallel Sessions Supported)

**Design**: One Pear session = one feature, but multiple sessions can run in parallel

**Rationale**: 
- Maintains focus within each session
- Prevents context pollution
- Folder locking prevents conflicts between parallel sessions

**Capability**: Team of 5 developers can run 5 sessions on 5 different features simultaneously

### 4. Requires Human Engagement

**This is intentional**, but it means:
- Not suitable for fully automated pipelines
- Requires developer time at each checkpoint
- Progress blocked until human responds

### 5. LLM Quality Dependence

**Limitation**: Output quality depends on LLM capabilities

**Mitigation**:
- Support multiple providers
- User can switch models mid-session
- Clear feedback loop allows correction

### 6. No Real-time Collaboration

**Limitation**: Single-user tool (one human per session)

**Future consideration**: Multi-user sessions with role assignment

### 7. Local-Only Locking (For Now)

**Limitation**: Folder locks are local to one machine

**Impact**: If a team works across multiple machines, locks won't synchronize automatically

**Workaround**: 
- Communicate via Slack/chat about who's working on what
- Use separate branches per developer
- Check `.pear/locks.yaml` in source control (manual coordination)

**Future**: Git-based distributed locking (see Future Roadmap)

---

## Future Roadmap

### Phase 1: MVP (v0.1)
- [ ] VS Code fork scaffold
- [ ] Four-phase workflow implementation
- [ ] Basic conversation UI
- [ ] Co-located artifact storage (DESIGN.md in feature folders)
- [ ] Single LLM provider (Claude)
- [ ] Language-agnostic architecture (TypeScript, Python, Go out of box)
- [ ] Basic folder locking for parallel sessions

### Phase 2: Polish (v0.2)
- [ ] Workflow sidebar with phase visualization
- [ ] Lock status visualization in file explorer
- [ ] Multiple LLM provider support
- [ ] Session pause/resume
- [ ] Design document templates
- [ ] Test coverage integration

### Phase 3: Expand (v0.3)
- [ ] Quick mode for small changes (skip full workflow)
- [ ] Shared folder contention UI (wait/local/skip options)
- [ ] Consolidation workflow (promote feature-local utils to shared)
- [ ] Export session as documentation
- [ ] Git branch integration (one branch per feature session)

### Phase 4: Advanced (v1.0)
- [ ] Team collaboration features (shared lock visibility)
- [ ] Custom workflow definitions
- [ ] Plugin system for custom languages
- [ ] Analytics and productivity metrics
- [ ] Session history and replay

### Phase 5: Team Scale (v1.5+)
- [ ] **Distributed locking via Git**: Lock coordination across machines using git as the source of truth
- [ ] Lock state stored in a dedicated branch or lightweight git notes
- [ ] GitHub/GitLab integration for lock visibility in PRs
- [ ] Slack/Discord notifications when locks are acquired/released
- [ ] Team dashboard showing active sessions across all developers

#### Distributed Locking Design (Future)

For remote teams, locking must work across machines. Proposed approach using Git:

```
┌─────────────────────────────────────────────────────────────────┐
│                  DISTRIBUTED LOCK COORDINATION                  │
│                                                                 │
│   Developer A (NYC)              Developer B (London)           │
│   ┌─────────────┐                ┌─────────────┐               │
│   │   Pear      │                │   Pear      │               │
│   │   Client    │                │   Client    │               │
│   └──────┬──────┘                └──────┬──────┘               │
│          │                              │                       │
│          │         Git Remote           │                       │
│          │    ┌─────────────────┐       │                       │
│          └───▶│ pear/locks.yaml │◀──────┘                       │
│               │ (on pear-locks  │                               │
│               │  branch)        │                               │
│               └─────────────────┘                               │
│                                                                 │
│   Lock acquisition:                                             │
│   1. git fetch origin pear-locks                                │
│   2. Check if folder is locked                                  │
│   3. If free: add lock, commit, push                            │
│   4. If push fails (conflict): retry from step 1                │
│   5. Lock acquired when push succeeds                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Benefits of Git-based locking:
- No additional infrastructure needed
- Works with any Git host (GitHub, GitLab, self-hosted)
- Lock history is version-controlled
- Integrates with existing Git workflows

### Exploratory Features
- **Voice interface**: Dictate directions to LLM
- **Visual diff**: Side-by-side implementation review with accept/reject per line
- **Rollback branches**: Git integration for phase rollback
- **Learning mode**: Pear learns user preferences over time
- **AI pair reviews**: Second LLM reviews first LLM's code for quality

---

## Appendix

### A. Example Session Transcript

```
═══════════════════════════════════════════════════════════════════
SESSION: Add User Authentication
PHASE: 1/4 - Planning
═══════════════════════════════════════════════════════════════════

[Human]: I want to add Google OAuth authentication to the app.

[Pear]: I'll help you design Google OAuth authentication. Let me ask a 
few clarifying questions:

1. **Scope**: Should this protect all routes or specific ones?
2. **Session Duration**: How long should sessions last?
3. **User Storage**: Do you have a database for user records, or 
   should we create one?
4. **Logout**: Should users be able to manually log out?

[Human]: 
1. Protected routes only, I'll mark them
2. 24 hours
3. We have Postgres, use that
4. Yes, logout needed

[Pear]: I'll create the feature folder and draft a design document.

📁 Created: src/features/auth/
📄 Writing: src/features/auth/DESIGN.md

---
# Feature: Google OAuth Authentication

## Problem Statement
The application needs authentication to protect sensitive routes...

## Proposed Solution  
Implement Google OAuth 2.0 with server-side session management...

## Technical Approach
1. OAuth callback endpoint (`/api/auth/callback`)
2. Session service with 24h expiry
3. `@protected` decorator for route marking
4. User table in Postgres
5. Logout endpoint (`/api/auth/logout`)

## Scope
- Google OAuth flow
- Session management (24h)
- Protected route decorator
- Logout functionality

## Out of Scope
- Other OAuth providers
- Remember me functionality
- MFA
---

🔒 Acquired write lock on: src/features/auth/

⚠️  Reminder: Please commit any uncommitted changes before proceeding.

[Approve] [Request Changes] [Ask Questions]

[Human]: [Approve]

═══════════════════════════════════════════════════════════════════
PHASE: 2/4 - Interface & Tests
═══════════════════════════════════════════════════════════════════

[Pear]: Great! Moving to interface design. Here are the proposed types:

```typescript
// src/features/auth/types.ts

export interface AuthConfig {
  googleClientId: string;
  googleClientSecret: string;
  callbackUrl: string;
  sessionDurationSeconds: number;
}

export interface User {
  id: string;
  email: string;
  name: string;
  avatarUrl: string | null;
  createdAt: Date;
}

export interface Session {
  id: string;
  userId: string;
  expiresAt: Date;
}

export interface AuthService {
  getOAuthUrl(): string;
  handleCallback(code: string): Promise<Session>;
  validateSession(sessionId: string): Promise<User | null>;
  logout(sessionId: string): Promise<void>;
}
```

[Approve Types] [Modify]

[Human]: Add `lastLoginAt` to User

[Pear]: Updated:

```typescript
export interface User {
  id: string;
  email: string;
  name: string;
  avatarUrl: string | null;
  createdAt: Date;
  lastLoginAt: Date;  // Added
}
```

[Approve Types] [Modify]

[Human]: [Approve Types]

[Pear]: Now for test cases:

```typescript
describe('AuthService', () => {
  describe('getOAuthUrl', () => {
    it('should return valid Google OAuth URL with correct scopes');
    it('should include state parameter for CSRF protection');
  });
  
  describe('handleCallback', () => {
    it('should create user if not exists');
    it('should update lastLoginAt for existing user');
    it('should create session with correct expiry');
    it('should reject invalid OAuth code');
  });
  
  describe('validateSession', () => {
    it('should return user for valid session');
    it('should return null for expired session');
    it('should return null for non-existent session');
  });
  
  describe('logout', () => {
    it('should delete session');
    it('should handle non-existent session gracefully');
  });
});
```

[Approve Tests] [Add More Tests]

[Human]: [Approve Tests]

═══════════════════════════════════════════════════════════════════
PHASE: 3/4 - Implementation (1/6)
═══════════════════════════════════════════════════════════════════

[Pear]: Starting implementation. First unit: `getOAuthUrl()`

```typescript
// src/features/auth/authService.ts

export function getOAuthUrl(config: AuthConfig): string {
  const state = crypto.randomBytes(16).toString('hex');
  
  const params = new URLSearchParams({
    client_id: config.googleClientId,
    redirect_uri: config.callbackUrl,
    response_type: 'code',
    scope: 'openid email profile',
    state,
    access_type: 'offline',
  });
  
  // Store state for CSRF validation (implementation in next unit)
  stateStore.set(state, Date.now());
  
  return `https://accounts.google.com/o/oauth2/v2/auth?${params}`;
}
```

**Notes**:
- Using `crypto.randomBytes` for cryptographic state
- State stored temporarily for CSRF validation
- Scopes limited to what we need (email, profile)

[Approve] [Request Changes]

[Human]: Looks good. [Approve]

[Pear]: Unit 1/6 complete. Next: `handleCallback()`...
```

### B. Configuration Schema

```yaml
# .pear/config.yaml

version: 1

llm:
  provider: anthropic  # anthropic | openai | azure
  model: claude-sonnet-4-20250514
  apiKeyEnvVar: ANTHROPIC_API_KEY

workflow:
  requireDesignApproval: true
  requireTestsFirst: true
  autoRunTestsOnComplete: true
  
templates:
  designDocument: default  # or path to custom template
  
# Folder structure for feature isolation
paths:
  features: src/features     # Where feature folders live
  shared:                    # Shared code (requires write lock)
    - src/shared
    - src/utils
    - src/types
  
# Lock configuration for parallel sessions
locking:
  enabled: true
  lockTimeout: 3600          # Seconds before abandoned lock auto-releases
  
# Git integration
git:
  warnUncommittedChanges: true    # Remind to commit before new features
  autoCommitOnComplete: false     # Auto-commit when feature completes
  commitMessageTemplate: "feat({feature}): {description}"
  
# Session management
sessions:
  autoCheckpointInterval: 30      # Seconds between auto-saves
  archiveCompletedSessions: true  # Keep history of completed sessions
  
ignore:
  - node_modules
  - dist
  - "*.test.ts"  # Don't include test files in context by default
  
# Language configuration (auto-detected, but can be overridden)
languages:
  typescript:
    testCommand: "npx vitest run"
    testFilePattern: "*.test.ts"
    testFramework: "vitest"       # Helps LLM write correct test syntax
  javascript:
    testCommand: "npx vitest run"
    testFilePattern: "*.test.js"
    testFramework: "vitest"
  python:
    testCommand: "pytest"
    testFilePattern: "test_*.py"
    testFramework: "pytest"
```

---

## Contributing

Pear is designed to be extended. Key areas for contribution:

1. **Language Plugins**: Add support for new languages
2. **LLM Providers**: Integrate additional AI providers
3. **Workflow Templates**: Create specialized workflows for different project types
4. **UI/UX**: Improve the conversation and visualization interfaces

---

## License

MIT

---

*Pear: Stay in the loop. Stay in control.*

