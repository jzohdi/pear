# Architecture

**Document**: `01-architecture.md`  
**Purpose**: Define the high-level architecture and component relationships

---

## System Overview

The CLI application follows a layered architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            USER LAYER                                    │
│                                                                          │
│                         Terminal (stdin/stdout)                          │
│                                                                          │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         PRESENTATION LAYER                               │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        CLI Handler                               │    │
│  │                                                                  │    │
│  │  • Display formatted output (code, markdown, progress)          │    │
│  │  • Prompt for user input (actions, free text)                   │    │
│  │  • Stream LLM responses to terminal                             │    │
│  │  • Handle keyboard shortcuts                                     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        ORCHESTRATION LAYER                               │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Workflow Controller                           │    │
│  │                                                                  │    │
│  │  • Manage phase transitions (Planning → Interface → ...)        │    │
│  │  • Enforce approval gates                                        │    │
│  │  • Coordinate component interactions                             │    │
│  │  • Handle go-back transitions                                    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└───────┬───────────────────┬───────────────────┬───────────────────┬─────┘
        │                   │                   │                   │
        ▼                   ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ State Manager │   │  LLM Client   │   │ File Manager  │   │  Test Runner  │
│               │   │               │   │               │   │               │
│ • Sessions    │   │ • API calls   │   │ • Read files  │   │ • Run Vitest  │
│ • Checkpoints │   │ • Streaming   │   │ • Write files │   │ • Parse output│
│ • Messages    │   │ • Context     │   │ • Artifacts   │   │ • Results     │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                   │                   │
        ▼                   ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          INFRASTRUCTURE LAYER                            │
│                                                                          │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐  │
│  │   JSON      │   │  Anthropic  │   │   File      │   │   Child     │  │
│  │   Files     │   │   API       │   │   System    │   │   Process   │  │
│  └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                       CLI APPLICATION                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │   CLI Handler   │────▶│  Workflow       │                   │
│  │                 │◀────│  Controller     │                   │
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
│         │                 ┌───────┴───────┐                 │   │
│         │                 │               │                 │   │
│         │                 ▼               ▼                 │   │
│         │         ┌─────────────┐ ┌─────────────┐           │   │
│         │         │  Response   │ │  Context    │           │   │
│         │         │  Parser     │ │  Manager    │           │   │
│         │         └─────────────┘ └─────────────┘           │   │
│         │                                                   │   │
│         │                 ┌─────────────┐                   │   │
│         │                 │  Test       │                   │   │
│         │                 │  Runner     │                   │   │
│         │                 └─────────────┘                   │   │
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

---

## Data Flow

### Request Flow (User → LLM)

```
User Input
    │
    ▼
┌─────────────┐
│ CLI Handler │─────────────────────────────────────┐
└─────────────┘                                     │
    │                                               │
    │ Action (approve/modify/go_back)               │ Display prompt
    ▼                                               │
┌─────────────┐                                     │
│ Workflow    │                                     │
│ Controller  │                                     │
└─────────────┘                                     │
    │                                               │
    │ Build context for phase                       │
    ▼                                               │
┌─────────────┐                                     │
│ Context     │                                     │
│ Manager     │                                     │
└─────────────┘                                     │
    │                                               │
    │ Messages + system prompt                      │
    ▼                                               │
┌─────────────┐                                     │
│ LLM Client  │                                     │
└─────────────┘                                     │
    │                                               │
    │ API request                                   │
    ▼                                               │
┌─────────────┐                                     │
│ Anthropic   │                                     │
│ API         │                                     │
└─────────────┘                                     │
    │                                               │
    │ Streamed response                             │
    ▼                                               │
┌─────────────┐                                     │
│ Response    │                                     │
│ Parser      │                                     │
└─────────────┘                                     │
    │                                               │
    │ Artifacts + formatted output                  │
    ▼                                               │
┌─────────────┐         ┌─────────────┐             │
│ File        │         │ State       │             │
│ Manager     │         │ Manager     │             │
└─────────────┘         └─────────────┘             │
    │                         │                     │
    │ Write files             │ Update state        │
    ▼                         ▼                     │
Project Files            Checkpoint                 │
                              │                     │
                              └─────────────────────┘
                                      │
                                      ▼
                               Terminal Output
```

### Checkpoint Flow

```
State Change (any)
    │
    ▼
┌─────────────┐
│ Workflow    │
│ Controller  │
└─────────────┘
    │
    │ saveCheckpoint()
    ▼
┌─────────────┐
│ State       │
│ Manager     │
└─────────────┘
    │
    ├──────────────────────┐
    │                      │
    ▼                      ▼
┌─────────────┐    ┌─────────────┐
│ Backup      │    │ Current     │
│ (previous)  │    │ (new)       │
│ .backup.json│    │ .json       │
└─────────────┘    └─────────────┘
```

---

## Component Responsibilities

| Component | Primary Responsibility | Dependencies |
|-----------|----------------------|--------------|
| **CLI Handler** | User interaction, display | Workflow Controller |
| **Workflow Controller** | Phase management, gates | All other components |
| **State Manager** | Session persistence | File system |
| **LLM Client** | API communication | Anthropic SDK, Context Manager |
| **Context Manager** | Token budgeting | State Manager |
| **Response Parser** | Extract artifacts from LLM output | None |
| **File Manager** | File operations | File system |
| **Test Runner** | Execute and parse tests | Child process, File Manager |

---

## Phase-Specific Flow

### Planning Phase

```
┌────────┐    ┌────────────┐    ┌────────┐    ┌────────────┐
│ Human  │───▶│ Questions? │───▶│ Human  │───▶│ Design Doc │
│ Input  │    │ (LLM)      │    │ Answers│    │ (LLM)      │
└────────┘    └────────────┘    └────────┘    └────────────┘
                                                    │
                                                    ▼
                                              ┌──────────┐
                                              │ Approve? │
                                              └──────────┘
                                                    │
                                    ┌───────────────┼───────────────┐
                                    │               │               │
                                    ▼               ▼               ▼
                               [Approve]       [Modify]        [Go Back]
                                    │               │               │
                                    ▼               │               │
                              Save DESIGN.md       └───────────────┘
                                    │                     │
                                    ▼                     ▼
                              Interface Phase       Revise design
```

### Interface Phase

```
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│ Design Doc │───▶│ Types      │───▶│ Test Cases │───▶│ Impl Order │
│ (input)    │    │ (LLM)      │    │ (LLM)      │    │ (LLM)      │
└────────────┘    └────────────┘    └────────────┘    └────────────┘
                        │                 │                 │
                        ▼                 ▼                 ▼
                   [Approve?]        [Approve?]        [Approve?]
                        │                 │                 │
                        └─────────────────┼─────────────────┘
                                          │
                                          ▼
                                  Save types.ts + 
                                  test skeletons
                                          │
                                          ▼
                                Implementation Phase
```

### Implementation Phase

```
┌─────────────────────────────────────────────────────────────────┐
│                    IMPLEMENTATION LOOP                          │
│                                                                 │
│                      Production Units                           │
│                            │                                    │
│   ┌───────────────────────┬┴───────────────────────┐           │
│   │                       │                        │           │
│   ▼                       ▼                        ▼           │
│ Unit 1                 Unit 2        ...        Unit N         │
│ (leaf)                  ...                     (root)         │
│   │                       │                        │           │
│   ▼                       ▼                        ▼           │
│ [Approve?]            [Approve?]              [Approve?]       │
│                                                                 │
│                       Test Units                                │
│                            │                                    │
│   ┌───────────────────────┬┴───────────────────────┐           │
│   │                       │                        │           │
│   ▼                       ▼                        ▼           │
│ Test 1                 Test 2        ...        Test M         │
│   │                       │                        │           │
│   ▼                       ▼                        ▼           │
│ [Approve?]            [Approve?]              [Approve?]       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                      Testing Phase
```

### Testing Phase

```
┌─────────────────────────────────────────────────────────────────┐
│                     TEST-DEBUG LOOP                             │
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

---

## Key Interfaces

See [02-components.md](./02-components.md) for detailed component specifications.

```typescript
// Core interfaces (summary)
interface WorkflowController {
  startNewSession(description: string, path: string): Promise<Session>;
  resumeSession(sessionId: string): Promise<Session>;
  runCurrentPhase(): Promise<PhaseResult>;
  approveCurrentStep(): Promise<void>;
  requestChanges(feedback: string): Promise<void>;
  goBackToPhase(phase: Phase): Promise<GoBackResult>;
}

interface StateManager {
  createSession(initial: Partial<WorkflowState>): WorkflowState;
  getSession(sessionId: string): WorkflowState | null;
  saveCheckpoint(state: WorkflowState): Promise<void>;
  loadCheckpoint(sessionId: string): Promise<WorkflowState | null>;
}

interface LLMClient {
  chat(options: ChatOptions): AsyncIterable<string>;
}

interface FileManager {
  readFile(path: string): Promise<string | null>;
  writeFile(path: string, content: string): Promise<void>;
  applyArtifacts(artifacts: Artifact[]): Promise<ApplyResult>;
}

interface TestRunner {
  runTests(options: TestRunOptions): Promise<TestRunResult>;
}
```

---

## Design Decisions

### Why This Architecture?

| Decision | Alternatives Considered | Rationale |
|----------|------------------------|-----------|
| **Layered architecture** | Monolithic, microservices | Clear separation, testable, appropriate scale |
| **Single orchestrator** | Event-driven, pub/sub | Simpler control flow, easier debugging |
| **File-based storage** | SQLite, in-memory | Human-readable, no dependencies, portable |
| **Streaming responses** | Wait for complete | Better UX, immediate feedback |
| **Component interfaces** | Direct coupling | Testability, future extensibility |

### Tradeoffs

| Benefit | Cost |
|---------|------|
| Clear component boundaries | More boilerplate/interfaces |
| Testable in isolation | Integration complexity |
| File-based simplicity | No concurrent access |
| Streaming UX | More complex error handling |

---

## Related Documents

- [02-components.md](./02-components.md) — Detailed component specifications
- [03-data-models.md](./03-data-models.md) — TypeScript interfaces
- [07-phase-transitions.md](./07-phase-transitions.md) — State machine details

