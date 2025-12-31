# Phase Transitions

**Document**: `07-phase-transitions.md`  
**Purpose**: State machine definition and go-back logic

---

## Overview

The workflow follows a strict phase progression with mandatory human approval at each gate. Going backward is allowed but requires careful state management to preserve work.

---

## State Machine

```
                      ┌─────────────────────────────────────────────────────┐
                      │                                                     │
                      │           FORWARD FLOW (requires approval)          │
                      │                                                     │
                      ▼                                                     │
┌─────────────┐  approve  ┌─────────────┐  approve  ┌────────────────┐     │
│   PLANNING  │──────────▶│  INTERFACE  │──────────▶│ IMPLEMENTATION │     │
│             │           │             │           │                │     │
│  in_progress│           │  in_progress│           │  in_progress   │     │
│  awaiting   │           │  awaiting   │           │  awaiting      │     │
│  approved ──┼───────────│  approved ──┼───────────│  approved ─────┼──┐  │
└─────────────┘           └─────────────┘           └────────────────┘  │  │
      ▲                         ▲                          ▲            │  │
      │                         │                          │            │  │
      │    ┌────────────────────┼──────────────────────────┘            │  │
      │    │   go_back          │   go_back                             │  │
      │    │   (amend design)   │   (modify API)                        │  │
      │    │                    │                                       │  │
      │    │              ┌─────┴─────┐                                 │  │
      │    │              │           │                                 │  │
      │    │              │           ▼                                 │  │
      │    │              │    ┌─────────────┐  all tests   ┌─────────┐│  │
      │    │              │    │   TESTING   │────pass────▶│ COMPLETE ││  │
      │    │              │    │             │              └─────────┘│  │
      │    └──────────────┼────│  in_progress│◀────────────────────────┘  │
      │                   │    │  awaiting   │                            │
      │                   │    └─────────────┘                            │
      │                   │          │                                    │
      │                   │          │ go_back (test spec wrong)          │
      │                   │          ▼                                    │
      │                   └──────────┘                                    │
      │                                                                   │
      └───────────────────────────────────────────────────────────────────┘
                              go_back (amend design)
```

---

## Forward Transitions

### Planning → Interface

**Trigger**: Human approves design document

**Actions**:
1. Save DESIGN.md to disk
2. Set `planning.approved = true`
3. Set `currentPhase = 'interface'`
4. Set `phaseStatus = 'in_progress'`
5. Checkpoint state

**Validation**:
- Design document content must exist
- Design document must not be empty

```typescript
async function transitionToInterface(state: WorkflowState): Promise<void> {
  // Validate
  if (!state.planning.designDocContent) {
    throw new WorkflowError('No design document', 'PHASE_TRANSITION_INVALID');
  }
  
  // Save artifact
  await fileManager.writeFile(
    state.planning.designDocPath!,
    state.planning.designDocContent
  );
  
  // Update state
  state.planning.approved = true;
  state.currentPhase = 'interface';
  state.phaseStatus = 'in_progress';
  
  // Checkpoint
  await stateManager.saveCheckpoint(state);
}
```

### Interface → Implementation

**Trigger**: Human approves types, tests, and implementation order

**Actions**:
1. Save types.ts to disk
2. Save test file with skeletons
3. Set `interface.approved = true`
4. Create implementation units from dependency analysis
5. Set `currentPhase = 'implementation'`
6. Set `phaseStatus = 'in_progress'`
7. Checkpoint state

**Validation**:
- Types content must exist
- Tests content must exist
- Dependency analysis must be approved

```typescript
async function transitionToImplementation(state: WorkflowState): Promise<void> {
  // Validate
  if (!state.interface.typesContent) {
    throw new WorkflowError('No type definitions', 'PHASE_TRANSITION_INVALID');
  }
  if (!state.interface.testsContent) {
    throw new WorkflowError('No test cases', 'PHASE_TRANSITION_INVALID');
  }
  if (!state.interface.dependencyAnalysis?.orderApproved) {
    throw new WorkflowError('Implementation order not approved', 'PHASE_TRANSITION_INVALID');
  }
  
  // Save artifacts
  await fileManager.writeFile(
    state.interface.typesPath!,
    state.interface.typesContent
  );
  await fileManager.writeFile(
    state.interface.testsPath!,
    state.interface.testsContent
  );
  
  // Create implementation units
  state.implementation.units = createUnitsFromAnalysis(
    state.interface.dependencyAnalysis
  );
  state.implementation.dependencyOrder = 
    state.interface.dependencyAnalysis.suggestedOrder;
  state.implementation.currentUnitIndex = 0;
  state.implementation.stage = 'production';
  
  // Update state
  state.interface.approved = true;
  state.currentPhase = 'implementation';
  state.phaseStatus = 'in_progress';
  
  await stateManager.saveCheckpoint(state);
}
```

### Implementation → Testing

**Trigger**: All implementation units (production + tests) approved

**Actions**:
1. Set `currentPhase = 'testing'`
2. Set `phaseStatus = 'in_progress'`
3. Run initial test suite
4. Checkpoint state

**Validation**:
- All production units must be approved
- All test implementations must be approved

```typescript
async function transitionToTesting(state: WorkflowState): Promise<void> {
  // Validate
  const allUnitsApproved = state.implementation.units.every(
    u => u.status === 'approved'
  );
  const allTestsApproved = state.implementation.testImplementations.every(
    t => t.status === 'approved'
  );
  
  if (!allUnitsApproved || !allTestsApproved) {
    throw new WorkflowError(
      'Not all units approved', 
      'PHASE_TRANSITION_INVALID'
    );
  }
  
  // Update state
  state.currentPhase = 'testing';
  state.phaseStatus = 'in_progress';
  
  // Run initial tests
  const result = await testRunner.runTests({
    projectPath: process.cwd(),
    testPattern: `${state.featurePath}/**/*.test.ts`,
  });
  
  state.testing.testRuns.push(result);
  state.testing.lastTestRun = result;
  
  await stateManager.saveCheckpoint(state);
}
```

### Testing → Complete

**Trigger**: All tests pass AND manual tests complete

**Actions**:
1. Generate TEST_EVIDENCE.md
2. Archive session to done/ folder
3. Set `currentPhase = 'complete'`
4. Display completion summary

**Validation**:
- All automated tests must pass
- All manual tests must be verified

```typescript
async function transitionToComplete(state: WorkflowState): Promise<CompletionResult> {
  // Validate
  if (state.testing.lastTestRun?.failed !== 0) {
    throw new WorkflowError(
      'Tests are failing', 
      'PHASE_TRANSITION_INVALID'
    );
  }
  if (!state.testing.manualTestsComplete) {
    throw new WorkflowError(
      'Manual tests incomplete', 
      'PHASE_TRANSITION_INVALID'
    );
  }
  
  // Generate evidence document
  const evidence = await llmClient.generateTestEvidence(state);
  const evidencePath = `${state.featurePath}/TEST_EVIDENCE.md`;
  await fileManager.writeFile(evidencePath, evidence);
  state.testing.testEvidencePath = evidencePath;
  state.testing.testEvidenceGenerated = true;
  
  // Update state
  state.currentPhase = 'complete';
  
  // Archive session
  const archivePath = await stateManager.archiveSession(state.sessionId);
  
  // Build result
  return {
    success: true,
    summary: buildCompletionSummary(state),
    filesCreated: getCreatedFiles(state),
    filesModified: [],
    testsPassed: state.testing.lastTestRun!.passed,
    sessionArchivePath: archivePath,
  };
}
```

---

## Backward Transitions

Going backward preserves work while allowing changes to earlier decisions.

### Interface → Planning (Amend Design)

**When**: User wants to change the design document

**Preserves**:
- Interface state (types, tests, order)
- Implementation state (if any units approved)

**Process**:
1. Save current interface/implementation state
2. Set `currentPhase = 'planning'`
3. Set `planning.approved = false`
4. Reopen design document for editing

```typescript
async function goBackToPlanning(state: WorkflowState): Promise<GoBackResult> {
  const previousPhase = state.currentPhase;
  
  // Preserve state (no changes needed, just re-enable editing)
  state.currentPhase = 'planning';
  state.phaseStatus = 'in_progress';
  state.planning.approved = false;
  
  await stateManager.saveCheckpoint(state);
  
  return {
    previousPhase,
    targetPhase: 'planning',
    preservedState: {
      interface: state.interface,
      implementation: state.implementation,
    },
    invalidatedItems: [],
    userActionRequired: true,
    userPrompt: 'Design document reopened for editing. Make your changes, then approve to continue.',
  };
}
```

**After Re-approval**:
- If interface state exists, ask: "Design changed. Review types/tests for compatibility?"
- User can choose to keep or regenerate interface

### Implementation → Interface (Modify API)

**When**: User realizes the types or tests need changes

**Preserves**:
- Implementation units that don't depend on changed types

**Process**:
1. Set `currentPhase = 'interface'`
2. Set `interface.approved = false`
3. Track which types/functions changed
4. Mark dependent units as `'needs_review'`

```typescript
async function goBackToInterface(state: WorkflowState): Promise<GoBackResult> {
  const previousPhase = state.currentPhase;
  
  // Preserve implementation state
  const approvedUnits = state.implementation.units.filter(
    u => u.status === 'approved'
  );
  
  // Update state
  state.currentPhase = 'interface';
  state.phaseStatus = 'in_progress';
  state.interface.approved = false;
  
  await stateManager.saveCheckpoint(state);
  
  return {
    previousPhase,
    targetPhase: 'interface',
    preservedState: {
      implementation: state.implementation,
    },
    invalidatedItems: [], // Will be determined after types change
    userActionRequired: true,
    userPrompt: `Types and tests reopened for editing. ${approvedUnits.length} implemented units preserved.`,
  };
}
```

**After Types Change**:
1. Compare old types to new types
2. Identify changed function signatures
3. Mark units using those functions as `'needs_review'`
4. Show user which units may need reimplementation

```typescript
async function handleTypesChanged(
  state: WorkflowState,
  oldTypes: string,
  newTypes: string
): Promise<InvalidatedItem[]> {
  const invalidated: InvalidatedItem[] = [];
  
  // Parse and compare types (simplified)
  const oldFunctions = parseFunctionSignatures(oldTypes);
  const newFunctions = parseFunctionSignatures(newTypes);
  
  for (const [name, oldSig] of oldFunctions) {
    const newSig = newFunctions.get(name);
    if (!newSig || oldSig !== newSig) {
      // Function changed, find dependent units
      for (const unit of state.implementation.units) {
        if (unit.dependencies.includes(name) || unit.name === name) {
          unit.status = 'needs_review';
          invalidated.push({
            type: 'unit',
            id: unit.id,
            name: unit.name,
            reason: `Depends on changed function: ${name}`,
          });
        }
      }
    }
  }
  
  return invalidated;
}
```

### Testing → Interface (Test Spec Wrong)

**When**: A test fails because the test itself is wrong (not the implementation)

**Preserves**:
- All implementation
- Other tests

**Process**:
1. Set `currentPhase = 'interface'`
2. Mark specific test for modification
3. After test fix, return to testing

```typescript
async function goBackToFixTest(
  state: WorkflowState,
  testId: string
): Promise<GoBackResult> {
  const previousPhase = state.currentPhase;
  const test = state.implementation.testImplementations.find(
    t => t.id === testId
  );
  
  // Update state
  state.currentPhase = 'interface';
  state.phaseStatus = 'in_progress';
  state.interface.approved = false;
  
  await stateManager.saveCheckpoint(state);
  
  return {
    previousPhase,
    targetPhase: 'interface',
    preservedState: {
      implementation: state.implementation,
      testing: state.testing,
    },
    invalidatedItems: [{
      type: 'test',
      id: testId,
      name: test?.testName || 'Unknown test',
      reason: 'Test specification needs correction',
    }],
    userActionRequired: true,
    userPrompt: `Test "${test?.testName}" marked for correction. Fix the test, then rerun.`,
  };
}
```

### Testing → Implementation (Debug Fix)

**When**: A test fails due to implementation bug

**Preserves**:
- All other units
- Test results

**Process**:
1. Stay in testing phase (or brief return to implementation)
2. LLM proposes fix
3. Human approves fix
4. Rerun tests

This is handled by the debug loop within the testing phase, not a full phase transition.

---

## State Preservation Rules

| Transition | Preserved | May Need Re-work |
|------------|-----------|------------------|
| Interface → Planning | Everything | Nothing (design is source of truth) |
| Implementation → Interface | Approved units | Units using changed types |
| Testing → Interface | All implementation | Corrected test only |
| Testing → Implementation | Other units | Fixed unit only |

---

## Transition Validation

```typescript
function validateTransition(
  currentPhase: Phase,
  targetPhase: Phase,
  direction: 'forward' | 'backward'
): boolean {
  const validForward: Record<Phase, Phase | null> = {
    planning: 'interface',
    interface: 'implementation',
    implementation: 'testing',
    testing: 'complete',
    complete: null,
  };
  
  const validBackward: Record<Phase, Phase[]> = {
    planning: [],
    interface: ['planning'],
    implementation: ['interface', 'planning'],
    testing: ['implementation', 'interface', 'planning'],
    complete: [],
  };
  
  if (direction === 'forward') {
    return validForward[currentPhase] === targetPhase;
  } else {
    return validBackward[currentPhase].includes(targetPhase);
  }
}
```

---

## Implementation in Workflow Controller

```typescript
async function goBackToPhase(
  state: WorkflowState,
  targetPhase: Phase
): Promise<GoBackResult> {
  // Validate transition
  if (!validateTransition(state.currentPhase, targetPhase, 'backward')) {
    throw new WorkflowError(
      `Cannot go from ${state.currentPhase} to ${targetPhase}`,
      'PHASE_TRANSITION_INVALID'
    );
  }
  
  // Execute appropriate transition
  switch (targetPhase) {
    case 'planning':
      return goBackToPlanning(state);
    case 'interface':
      return goBackToInterface(state);
    default:
      throw new WorkflowError(
        `Unknown target phase: ${targetPhase}`,
        'PHASE_TRANSITION_INVALID'
      );
  }
}
```

---

## Related Documents

- [03-data-models.md](./03-data-models.md) — State types
- [02-components.md](./02-components.md) — Workflow Controller
- [01-architecture.md](./01-architecture.md) — State machine overview

