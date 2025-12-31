# Context Management

**Document**: `06-context-management.md`  
**Purpose**: Token budgeting and conversation history management

---

## Overview

The Context Manager builds appropriate context for each LLM request while staying within token limits. This ensures:

1. **Relevant information is included**: Design doc, types, current unit
2. **Token limits are respected**: Don't exceed Claude's context window
3. **History is preserved**: Recent conversation provides continuity

---

## Token Budget

### Claude Sonnet Limits

| Limit | Value |
|-------|-------|
| Context window | 200,000 tokens |
| Practical limit | ~100,000 tokens (leave room for response) |
| Reserved for response | 4,000 tokens |

### Estimation

Rough approximation: **~4 characters per token** for English/code.

```typescript
const CHARS_PER_TOKEN = 4;

function estimateTokens(text: string): number {
  return Math.ceil(text.length / CHARS_PER_TOKEN);
}
```

**Note**: This is an approximation. For production, use the Anthropic tokenizer or count actual tokens from API responses.

---

## Context Window Structure

```typescript
interface ContextWindow {
  // ─────────────────────────────────────────────────────────────
  // High Priority (always included)
  // ─────────────────────────────────────────────────────────────
  
  /** System prompt for the current phase */
  systemPrompt: string;
  
  /** Design document (source of truth) */
  designDocument: string;
  
  // ─────────────────────────────────────────────────────────────
  // Medium Priority (included based on phase)
  // ─────────────────────────────────────────────────────────────
  
  /** Type definitions (Phase 2+) */
  typeDefinitions?: string;
  
  /** Test cases relevant to current work */
  relevantTests?: string;
  
  /** Current unit being implemented */
  currentUnit?: string;
  
  // ─────────────────────────────────────────────────────────────
  // Lower Priority (trimmed first if needed)
  // ─────────────────────────────────────────────────────────────
  
  /** Recent conversation messages */
  recentMessages: Message[];
  
  /** Previous implementation units for context */
  previousUnits?: string[];
  
  // ─────────────────────────────────────────────────────────────
  // Metadata
  // ─────────────────────────────────────────────────────────────
  
  /** Total estimated tokens */
  estimatedTokens: number;
}
```

---

## Budget Allocation

### Total Budget: ~96,000 tokens

```
┌────────────────────────────────────────────────────────────────┐
│                     TOKEN BUDGET                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Reserved for Response        4,000  ████                      │
│                                                                │
│  System Prompt                  500  █                         │
│                                                                │
│  Design Document             1,000  ██                         │
│                                                                │
│  Type Definitions            2,000  ████                       │
│  (Phase 2+)                                                    │
│                                                                │
│  Current Context             2,000  ████                       │
│  (unit, tests)                                                 │
│                                                                │
│  Message History            20,000  ████████████████████       │
│                                                                │
│  Available Buffer           70,500  ██████████████████████████ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Phase-Specific Allocations

| Phase | Must Include | Optional |
|-------|--------------|----------|
| Planning | System prompt, feature description | N/A |
| Interface | System prompt, design doc | Existing types |
| Implementation | System prompt, design doc, types, current unit | Previous 2-3 units |
| Testing | System prompt, design doc, types, failed tests | Implementation code |

---

## Building Context

### Main Function

```typescript
function buildContext(state: WorkflowState, phase: Phase): ContextWindow {
  const context: ContextWindow = {
    systemPrompt: getSystemPrompt(phase),
    designDocument: state.planning.designDocContent || '',
    recentMessages: [],
    estimatedTokens: 0,
  };
  
  // Calculate available tokens
  let availableTokens = MAX_CONTEXT_TOKENS - RESERVED_FOR_RESPONSE;
  
  // Always include system prompt and design doc
  availableTokens -= estimateTokens(context.systemPrompt);
  availableTokens -= estimateTokens(context.designDocument);
  
  // Add phase-specific context
  switch (phase) {
    case 'planning':
      // Planning only needs basic context
      break;
      
    case 'interface':
      // Include any existing types
      if (state.interface.typesContent) {
        context.typeDefinitions = state.interface.typesContent;
        availableTokens -= estimateTokens(context.typeDefinitions);
      }
      break;
      
    case 'implementation':
      // Include types (required)
      context.typeDefinitions = state.interface.typesContent || '';
      availableTokens -= estimateTokens(context.typeDefinitions);
      
      // Include current unit info
      const currentUnit = getCurrentUnit(state);
      if (currentUnit) {
        context.currentUnit = formatUnitContext(currentUnit);
        availableTokens -= estimateTokens(context.currentUnit);
      }
      
      // Include relevant tests
      const tests = getRelevantTests(state, currentUnit);
      if (tests) {
        context.relevantTests = tests;
        availableTokens -= estimateTokens(context.relevantTests);
      }
      
      // Include previous units (up to budget)
      context.previousUnits = getPreviousUnits(state, availableTokens / 2);
      break;
      
    case 'testing':
      // Include types
      context.typeDefinitions = state.interface.typesContent || '';
      availableTokens -= estimateTokens(context.typeDefinitions);
      
      // Include failure details
      if (state.testing.lastTestRun?.failures.length) {
        context.relevantTests = formatFailures(state.testing.lastTestRun.failures);
        availableTokens -= estimateTokens(context.relevantTests);
      }
      break;
  }
  
  // Fill remaining space with message history
  context.recentMessages = buildMessageWindow(
    state.messages,
    availableTokens
  );
  
  // Calculate total
  context.estimatedTokens = calculateTotalTokens(context);
  
  return context;
}
```

---

## Message Sliding Window

### Strategy

1. **Always include first message**: The original feature description
2. **Include recent messages**: From newest to oldest
3. **Stop when budget exhausted**: Don't truncate mid-message
4. **Add summary marker**: If messages were skipped

### Implementation

```typescript
function buildMessageWindow(
  allMessages: Message[], 
  maxTokens: number
): Message[] {
  if (allMessages.length === 0) return [];
  
  const result: Message[] = [];
  let tokenCount = 0;
  
  // Always include first message (feature description)
  const firstMessage = allMessages[0];
  result.push(firstMessage);
  tokenCount += estimateTokens(firstMessage.content);
  
  // If only one message, we're done
  if (allMessages.length === 1) return result;
  
  // Add recent messages from the end
  const recentMessages: Message[] = [];
  for (let i = allMessages.length - 1; i > 0; i--) {
    const message = allMessages[i];
    const msgTokens = estimateTokens(message.content);
    
    if (tokenCount + msgTokens > maxTokens) {
      break;
    }
    
    recentMessages.unshift(message);
    tokenCount += msgTokens;
  }
  
  // If we skipped messages, add a summary marker
  const skippedCount = allMessages.length - 1 - recentMessages.length;
  if (skippedCount > 0) {
    result.push({
      role: 'system',
      content: `[${skippedCount} earlier messages omitted for brevity]`,
      timestamp: new Date(),
      phase: recentMessages[0]?.phase || 'planning',
    });
  }
  
  // Add recent messages
  result.push(...recentMessages);
  
  return result;
}
```

### Example

**Input**: 50 messages, 15,000 token budget

**Output**:
```
[Message 1] Original feature request
[System] [45 earlier messages omitted for brevity]
[Message 47] User asked about error handling
[Message 48] Assistant proposed solution
[Message 49] User approved
[Message 50] Assistant confirmed
```

---

## Previous Units for Context

When implementing a new unit, include previously approved units for:

1. **Import context**: What's available to import
2. **Style consistency**: Follow established patterns
3. **Integration**: How to interact with existing code

### Implementation

```typescript
function getPreviousUnits(
  state: ImplementationState,
  maxTokens: number
): string[] {
  const approved = state.units.filter(u => u.status === 'approved');
  const result: string[] = [];
  let tokenCount = 0;
  
  // Include most recently approved units (up to 3)
  const recent = approved.slice(-3);
  
  for (const unit of recent) {
    if (!unit.implementation) continue;
    
    const formatted = formatUnitForContext(unit);
    const tokens = estimateTokens(formatted);
    
    if (tokenCount + tokens > maxTokens) break;
    
    result.push(formatted);
    tokenCount += tokens;
  }
  
  return result;
}

function formatUnitForContext(unit: ImplementationUnit): string {
  return `
// Previously implemented: ${unit.name}
// File: ${unit.filePath}
${unit.implementation}
`.trim();
}
```

---

## Context for Testing Phase

When debugging test failures, include:

### 1. Failure Details

```typescript
function formatFailures(failures: TestFailure[]): string {
  return failures.map(f => `
### Failed Test: ${f.testName}
**Error**: ${f.errorMessage}
**Expected**: ${f.expected}
**Received**: ${f.received}
${f.stack ? `**Stack**:\n\`\`\`\n${f.stack}\n\`\`\`` : ''}
  `).join('\n\n');
}
```

### 2. Implementation Being Tested

```typescript
function getImplementationForTests(
  state: WorkflowState,
  failures: TestFailure[]
): string {
  // Find which units the failing tests relate to
  const relevantUnits = state.implementation.units.filter(unit => {
    return failures.some(f => 
      f.testName.toLowerCase().includes(unit.name.toLowerCase())
    );
  });
  
  return relevantUnits
    .map(u => `// ${u.name}\n${u.implementation}`)
    .join('\n\n');
}
```

---

## Trimming Context

When context exceeds budget:

### Priority Order (trim from bottom)

1. Previous units (least recent first)
2. Older messages
3. **Never trim**: System prompt, design doc, current unit

### Implementation

```typescript
function trimToLimit(
  context: ContextWindow, 
  maxTokens: number
): ContextWindow {
  let current = { ...context };
  
  // Trim previous units first
  while (
    current.previousUnits?.length &&
    calculateTotalTokens(current) > maxTokens
  ) {
    current.previousUnits = current.previousUnits.slice(1);
  }
  
  // Trim messages next
  while (
    current.recentMessages.length > 1 &&
    calculateTotalTokens(current) > maxTokens
  ) {
    // Keep first message (feature description) and last few
    const messages = current.recentMessages;
    current.recentMessages = [
      messages[0],
      ...messages.slice(-Math.max(2, messages.length - 2))
    ];
  }
  
  current.estimatedTokens = calculateTotalTokens(current);
  return current;
}
```

---

## Monitoring Token Usage

Track token usage for debugging and optimization:

```typescript
interface TokenUsageLog {
  phase: Phase;
  systemPrompt: number;
  designDoc: number;
  types: number;
  currentContext: number;
  messages: number;
  total: number;
  timestamp: Date;
}

function logTokenUsage(context: ContextWindow, phase: Phase): TokenUsageLog {
  return {
    phase,
    systemPrompt: estimateTokens(context.systemPrompt),
    designDoc: estimateTokens(context.designDocument),
    types: context.typeDefinitions ? estimateTokens(context.typeDefinitions) : 0,
    currentContext: estimateTokens(
      (context.currentUnit || '') + (context.relevantTests || '')
    ),
    messages: context.recentMessages.reduce(
      (sum, m) => sum + estimateTokens(m.content), 
      0
    ),
    total: context.estimatedTokens,
    timestamp: new Date(),
  };
}
```

---

## Testing Context Management

```typescript
describe('ContextManager', () => {
  it('should always include design document', () => {
    const state = createMockState();
    const context = buildContext(state, 'implementation');
    
    expect(context.designDocument).toBeTruthy();
  });
  
  it('should respect token limits', () => {
    const state = createMockStateWithLongHistory();
    const context = buildContext(state, 'implementation');
    
    expect(context.estimatedTokens).toBeLessThan(MAX_CONTEXT_TOKENS);
  });
  
  it('should preserve first and recent messages', () => {
    const state = createMockStateWith50Messages();
    const context = buildContext(state, 'implementation');
    
    // First message should be original feature request
    expect(context.recentMessages[0].content).toContain('Add auth');
    
    // Last messages should be most recent
    expect(context.recentMessages.slice(-1)[0]).toEqual(
      state.messages.slice(-1)[0]
    );
  });
});
```

---

## Related Documents

- [04-llm-prompts.md](./04-llm-prompts.md) — System prompts to include
- [03-data-models.md](./03-data-models.md) — Message and state types
- [02-components.md](./02-components.md) — Context Manager component

