# User Interface

**Document**: `08-user-interface.md`  
**Purpose**: CLI layout, user interactions, and display formatting

---

## Overview

The CLI provides a structured, guided experience through the four-phase workflow. Key design principles:

1. **Clear phase indication**: Always show where the user is in the workflow
2. **Streaming output**: Display LLM responses as they arrive
3. **Obvious actions**: Make available actions clear at every step
4. **Progress visibility**: Show what's done and what's remaining

---

## Terminal Layout

### Standard View

```
╔══════════════════════════════════════════════════════════════════════╗
║  🍐 Pear CLI - Feature: User Authentication                          ║
║  Phase: 2/4 - Interface & Tests                                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  [Pear]: Based on the design document, here are the proposed         ║
║          type definitions:                                           ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────┐    ║
║  │ // src/features/auth/types.ts                                │    ║
║  │                                                              │    ║
║  │ export interface AuthConfig {                                │    ║
║  │   provider: 'google';                                        │    ║
║  │   clientId: string;                                          │    ║
║  │   clientSecret: string;                                      │    ║
║  │   callbackUrl: string;                                       │    ║
║  │   sessionDurationSeconds: number;                            │    ║
║  │ }                                                            │    ║
║  │                                                              │    ║
║  │ export interface User {                                      │    ║
║  │   id: string;                                                │    ║
║  │   email: string;                                             │    ║
║  │   ...                                                        │    ║
║  │ }                                                            │    ║
║  └──────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  Would you like to approve these types or request changes?           ║
║                                                                      ║
╠══════════════════════════════════════════════════════════════════════╣
║  [A] Approve   [M] Modify   [B] Back to Planning   [Q] Quit          ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Implementation Progress View

```
╔══════════════════════════════════════════════════════════════════════╗
║  🍐 Pear CLI - Feature: User Authentication                          ║
║  Phase: 3/4 - Implementation                                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Progress: Production Code (3/5)                                     ║
║  ────────────────────────────────────────────────────────────────    ║
║  ☑ getOAuthUrl()        [Approved]                                   ║
║  ☑ createSession()      [Approved]                                   ║
║  ☑ handleCallback()     [Approved]                                   ║
║  ☐ validateSession()    [In Progress] ◀── Current                    ║
║  ☐ logout()             [Pending]                                    ║
║                                                                      ║
║  Progress: Test Implementations (0/8)                                ║
║  ────────────────────────────────────────────────────────────────    ║
║  ☐ All tests pending (after production code)                         ║
║                                                                      ║
╠══════════════════════════════════════════════════════════════════════╣
║  Current: validateSession()                                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  [Pear]: Here's the implementation for validateSession():            ║
║                                                                      ║
║  ┌──────────────────────────────────────────────────────────────┐    ║
║  │ export async function validateSession(                       │    ║
║  │   sessionId: string                                          │    ║
║  │ ): Promise<User | null> {                                    │    ║
║  │   const session = await db.sessions.findUnique({             │    ║
║  │     where: { id: sessionId }                                 │    ║
║  │   });                                                        │    ║
║  │                                                              │    ║
║  │   if (!session) return null;                                 │    ║
║  │   if (session.expiresAt < new Date()) return null;           │    ║
║  │                                                              │    ║
║  │   return db.users.findUnique({                               │    ║
║  │     where: { id: session.userId }                            │    ║
║  │   });                                                        │    ║
║  │ }                                                            │    ║
║  └──────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║  **Implementation notes:**                                           ║
║  - Checks for session existence and expiry before returning user     ║
║  - Returns null for invalid/expired sessions (no exceptions)         ║
║                                                                      ║
╠══════════════════════════════════════════════════════════════════════╣
║  [A] Approve   [M] Modify   [B] Back to Interface   [Q] Quit         ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Key Interactions

### Starting a New Session

```
$ pear new

🍐 Pear CLI v0.1.0

? What feature would you like to build?
> Add Google OAuth authentication to protect certain routes

? Where should this feature live? (e.g., src/features/auth)
> src/features/auth

Creating new session...
✓ Session ID: abc123
✓ Feature path: src/features/auth

═══════════════════════════════════════════════════════════════════════
Phase 1/4: Planning
═══════════════════════════════════════════════════════════════════════

[Pear]: I'll help you design Google OAuth authentication. 
        Let me ask a few clarifying questions:

        1. Should this protect all routes or only specific ones?
        2. How long should user sessions last?
        3. Do you have an existing database for user records?
        4. Should users be able to manually log out?

? Your response:
> 
```

### Resuming a Session

```
$ pear resume

🍐 Pear CLI v0.1.0

Found incomplete sessions:

  1. User Authentication
     📍 Phase 3/4 - Implementation (3/5 units)
     ⏰ Last activity: 2 hours ago
     🔒 Path: src/features/auth

  2. Payment Processing
     📍 Phase 1/4 - Planning
     ⏰ Last activity: 1 day ago
     🔒 Path: src/features/payments

? Select a session to resume: (Use arrow keys)
❯ 1. User Authentication
  2. Payment Processing
  Cancel

Resuming session abc123...

═══════════════════════════════════════════════════════════════════════
Phase 3/4: Implementation (3/5)
═══════════════════════════════════════════════════════════════════════

Resuming from: validateSession()

[Pear]: Welcome back! You were working on validateSession().
        Would you like me to generate the implementation?

? Action: (Use arrow keys)
❯ Yes, continue
  Show previous context
  Start this unit over
```

### Approving Output

```
[Pear]: Here's the design document for your feature:

┌──────────────────────────────────────────────────────────────┐
│ // src/features/auth/DESIGN.md                               │
│                                                              │
│ # Feature: Google OAuth Authentication                       │
│                                                              │
│ ## Problem Statement                                         │
│ The application needs authentication to protect sensitive    │
│ routes...                                                    │
│                                                              │
│ ## Proposed Solution                                         │
│ Implement Google OAuth 2.0 flow with server-side session...  │
│                                                              │
│ [rest of document]                                           │
└──────────────────────────────────────────────────────────────┘

? Action: (Use arrow keys)
❯ Approve - Save and continue to Interface phase
  Modify - Request changes to the design
  Back - (not available in Planning)
  Quit - Save progress and exit
```

### Requesting Changes

```
? Action: Modify

? What changes would you like?
> Add a note about rate limiting for the OAuth callback

[Pear]: I'll update the design document to include rate limiting:

        **Changes made:**
        + Added rate limiting consideration to Technical Approach
        + Added "Rate limiting attacks" to Out of Scope with note
        
        Here's the updated design:
        
        [updated document displayed]

? Action: (Use arrow keys)
❯ Approve
  Modify
  Quit
```

### Going Back

```
═══════════════════════════════════════════════════════════════════════
Phase 3/4: Implementation (4/5)
═══════════════════════════════════════════════════════════════════════

? Action: Back to Interface

⚠️  Going back to Interface phase.

This will:
• Preserve your 4 approved implementations
• Reopen types.ts and tests for editing
• Units using changed types may need re-implementation

? Confirm going back to Interface? (y/N) y

═══════════════════════════════════════════════════════════════════════
Phase 2/4: Interface & Tests
═══════════════════════════════════════════════════════════════════════

[Pear]: Interface phase reopened.
        
        4 implemented units preserved:
        • getOAuthUrl()
        • createSession()
        • handleCallback()
        • validateSession()
        
        What would you like to change?

? Your response:
> I need to add a rememberMe option to the login
```

---

## Manual Test Recording

```
═══════════════════════════════════════════════════════════════════════
Phase 4/4: Testing - Manual Verification
═══════════════════════════════════════════════════════════════════════

✓ Automated tests: 12/12 passed

Now let's verify the feature manually.

Manual Test Checklist:
──────────────────────────────────────────────────────────────────────
  1. [ ] Navigate to /login, click "Sign in with Google"
         → Should redirect to Google OAuth page
  
  2. [ ] Complete Google sign-in with valid credentials
         → Should redirect back to app with session created
  
  3. [ ] Access a protected route (/dashboard)
         → Should display dashboard content
  
  4. [ ] Access protected route in incognito (no session)
         → Should redirect to /login
  
  5. [ ] Click "Logout" button
         → Should clear session and redirect to home

? Select a test to record: (Use arrow keys)
❯ 1. Navigate to /login... [pending]
  2. Complete Google sign-in... [pending]
  3. Access protected route... [pending]
  4. Access incognito... [pending]
  5. Click Logout... [pending]
  ──────────────────────────────────────
  Mark all as passed
  Skip manual tests
```

### Recording a Test Result

```
? Test: Navigate to /login, click "Sign in with Google"
  Expected: Should redirect to Google OAuth page

? Result: (Use arrow keys)
❯ ✓ Pass
  ✗ Fail
  ⊘ Skip

? Notes (optional):
> Redirect took about 2 seconds, but worked correctly

✓ Test recorded: Pass

Remaining: 4 tests

? Select next test: (Use arrow keys)
  1. Navigate to /login... [✓ passed]
❯ 2. Complete Google sign-in... [pending]
  ...
```

---

## Completion Screen

```
═══════════════════════════════════════════════════════════════════════
🎉 Feature Complete: User Authentication
═══════════════════════════════════════════════════════════════════════

Summary
──────────────────────────────────────────────────────────────────────
├── Design: src/features/auth/DESIGN.md
├── Types: src/features/auth/types.ts
├── Implementation: 5 production units, 8 tests
├── Automated Tests: 12/12 passed
├── Manual Tests: 5/5 passed
└── Evidence: src/features/auth/TEST_EVIDENCE.md

Files Created
──────────────────────────────────────────────────────────────────────
  ✓ src/features/auth/DESIGN.md
  ✓ src/features/auth/types.ts
  ✓ src/features/auth/index.ts
  ✓ src/features/auth/oauth.ts
  ✓ src/features/auth/session.ts
  ✓ src/features/auth/__tests__/auth.test.ts
  ✓ src/features/auth/TEST_EVIDENCE.md

Session archived: .pear/sessions/done/abc123.yaml

──────────────────────────────────────────────────────────────────────
💡 Tip: Don't forget to commit these changes to git!

? What would you like to do?
❯ Start a new feature
  Exit
```

---

## Color Scheme

Using `chalk` for terminal colors:

| Element | Color | Code |
|---------|-------|------|
| Phase header | Cyan | `chalk.cyan` |
| Pear label | Green | `chalk.green` |
| Code blocks | Gray background | `chalk.bgGray` |
| Success | Green | `chalk.green` |
| Warning | Yellow | `chalk.yellow` |
| Error | Red | `chalk.red` |
| Prompt | Blue | `chalk.blue` |
| Dim text | Gray | `chalk.gray` |

### Example

```typescript
console.log(chalk.cyan('═══════════════════════════════════════'));
console.log(chalk.cyan('Phase 2/4: Interface & Tests'));
console.log(chalk.cyan('═══════════════════════════════════════'));
console.log();
console.log(chalk.green('[Pear]:'), 'Here are the proposed types:');
console.log();
console.log(chalk.gray('┌──────────────────────────────────────┐'));
console.log(chalk.gray('│'), highlight(code, 'typescript'));
console.log(chalk.gray('└──────────────────────────────────────┘'));
```

---

## Streaming Display

Display LLM responses character by character:

```typescript
async function displayStreamingResponse(
  stream: AsyncIterable<string>
): Promise<string> {
  let fullResponse = '';
  
  // Hide cursor during streaming
  process.stdout.write('\x1B[?25l');
  
  try {
    for await (const chunk of stream) {
      process.stdout.write(chunk);
      fullResponse += chunk;
    }
  } finally {
    // Show cursor
    process.stdout.write('\x1B[?25h');
    console.log(); // New line after streaming
  }
  
  return fullResponse;
}
```

---

## Progress Indicators

### Spinner for Long Operations

```typescript
import ora from 'ora';

const spinner = ora('Running tests...').start();

try {
  const result = await testRunner.runTests(options);
  spinner.succeed('Tests complete');
} catch (error) {
  spinner.fail('Tests failed');
}
```

### Progress Bar for Multi-Unit Operations

```
Implementation Progress
[████████████░░░░░░░░] 60% (3/5 units)
```

```typescript
function renderProgressBar(current: number, total: number): string {
  const width = 20;
  const filled = Math.round((current / total) * width);
  const empty = width - filled;
  const percent = Math.round((current / total) * 100);
  
  return `[${'█'.repeat(filled)}${'░'.repeat(empty)}] ${percent}% (${current}/${total})`;
}
```

---

## Error Display

```typescript
function displayError(error: PearError): void {
  console.log();
  console.log(chalk.red('━'.repeat(60)));
  console.log(chalk.red('❌ Error'));
  console.log(chalk.red('━'.repeat(60)));
  console.log();
  console.log(chalk.red(error.message));
  
  if (error.context) {
    console.log();
    console.log(chalk.gray('Context:'));
    console.log(chalk.gray(JSON.stringify(error.context, null, 2)));
  }
  
  if (error.recoverable) {
    console.log();
    console.log(chalk.yellow('This error is recoverable. You can:'));
    console.log(chalk.yellow('  • Try again'));
    console.log(chalk.yellow('  • Use "pear resume" to continue later'));
  }
  
  console.log();
}
```

---

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `A` | Approve (when at approval prompt) |
| `M` | Modify (when at approval prompt) |
| `B` | Back (when at approval prompt) |
| `Q` | Quit (save and exit) |
| `Ctrl+C` | Force quit (no save) |
| `Enter` | Confirm selection |
| `↑/↓` | Navigate options |

---

## Accessibility Considerations

1. **No color-only information**: Use symbols (✓, ✗, ☐) alongside colors
2. **Clear labels**: Every action is labeled, not just icons
3. **Keyboard navigation**: All actions accessible via keyboard
4. **High contrast**: Use bold/bright colors for important info

---

## Related Documents

- [02-components.md](./02-components.md) — CLI Handler component
- [12-error-handling.md](./12-error-handling.md) — Error display
- [00-overview.md](./00-overview.md) — User experience goals

