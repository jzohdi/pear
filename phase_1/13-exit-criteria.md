# Exit Criteria

**Document**: `13-exit-criteria.md`  
**Purpose**: Define what "done" means for Phase 1 and criteria for proceeding to Phase 2

---

## Overview

Phase 1 is complete when the CLI prototype successfully validates the structured pair-programming workflow. This document defines the specific criteria that must be met and the deliverables expected.

---

## Completion Criteria

### Functional Criteria

| # | Criterion | Verification |
|---|-----------|--------------|
| 1 | All four phases complete end-to-end | Build one real feature using the CLI |
| 2 | Sessions persist across restarts | Start session, quit, resume, continue |
| 3 | Crash recovery works | Kill process mid-session, resume successfully |
| 4 | Go-back transitions preserve state | Return to Interface, verify units preserved |
| 5 | Tests can be run and debugged | Phase 4 debug loop completes |
| 6 | Manual tests can be recorded | Record 3+ manual test results |
| 7 | Artifacts are created correctly | DESIGN.md, types.ts, tests, TEST_EVIDENCE.md all exist |

### Quality Criteria

| # | Criterion | Target | Measurement |
|---|-----------|--------|-------------|
| 1 | No blocking bugs | 0 | Can complete workflow without manual workarounds |
| 2 | LLM output usability | <20% | Percentage requiring significant revision |
| 3 | Approval time reasonable | <30s avg | Time from seeing output to approving |
| 4 | No data loss | 100% | No work lost to crashes or errors |
| 5 | Clear error messages | Yes | User understands what went wrong |

### Subjective Criteria

These are assessed based on the experience of using the CLI:

| Question | Expected Answer |
|----------|-----------------|
| Does the four-phase workflow feel natural? | Yes, progression makes sense |
| Is unit-by-unit approval the right granularity? | Yes, or specific feedback on what to change |
| Would you use this tool again? | Yes |
| Do feedback loops feel tight enough? | Yes, never lost or disconnected |

---

## Required Deliverables

### 1. Working CLI Application

**Location**: `pear-cli/` directory  
**Requirements**:
- [ ] Compiles without errors (`npm run build`)
- [ ] All tests pass (`npm run test`)
- [ ] Can be run locally (`npm run dev` or `tsx src/index.ts`)

### 2. At Least One Complete Feature

**Evidence**: Feature built using the CLI  
**Requirements**:
- [ ] Feature works (can be demonstrated)
- [ ] All artifacts exist:
  - [ ] `DESIGN.md`
  - [ ] `types.ts`
  - [ ] Implementation files
  - [ ] Test file with passing tests
  - [ ] `TEST_EVIDENCE.md`

### 3. Refined LLM Prompts

**Location**: `src/llm/prompts.ts`  
**Requirements**:
- [ ] All four phase prompts documented
- [ ] Prompts produce usable output consistently
- [ ] Notes on what worked/didn't work

### 4. Learnings Document

**Location**: `phase_1/LEARNINGS.md`  
**Contents**:
- [ ] What worked well
- [ ] What didn't work
- [ ] Friction points encountered
- [ ] LLM behavior observations
- [ ] Recommendations for Phase 2

---

## Evaluation Questions

After completing Phase 1, answer these questions to guide the Phase 2 decision:

### Workflow Validation

1. **Does the four-phase structure make sense?**
   - Did Planning → Interface → Implementation → Testing feel like a natural progression?
   - Were any phases unnecessary or missing?

2. **Is the approval granularity correct?**
   - Did approving each unit feel valuable or tedious?
   - Should we batch approvals differently?

3. **Do go-back transitions work as expected?**
   - When you needed to change earlier decisions, did the workflow accommodate it?
   - Was state preserved correctly?

### LLM Performance

4. **How was the LLM output quality?**
   - What percentage required significant changes?
   - Were there patterns in what went wrong?

5. **Which prompts need improvement?**
   - Rank the phases by prompt effectiveness
   - Note specific issues

6. **Were there context window issues?**
   - Did the LLM ever seem to "forget" earlier context?
   - Did summarization work?

### User Experience

7. **How long did the workflow take?**
   - Compare to estimated time coding manually
   - Where were the bottlenecks?

8. **What was frustrating?**
   - List specific friction points
   - Prioritize for Phase 2

9. **What was surprisingly good?**
   - What exceeded expectations?
   - What should be preserved?

### Technical Assessment

10. **How is the code quality?**
    - Is the codebase ready to be extracted into a library?
    - What needs refactoring?

11. **Are there performance concerns?**
    - LLM response times
    - File operations
    - State management

12. **What's the test coverage?**
    - Current coverage percentage
    - Critical paths covered?

---

## Go/No-Go Decision

### Proceed to Phase 2 IF:

✅ All functional criteria are met  
✅ Quality targets are achieved or have clear mitigation plans  
✅ Subjective experience is positive  
✅ Learnings document identifies clear improvements (not blockers)  
✅ Team confidence is high  

### Do NOT Proceed IF:

❌ Core workflow has fundamental issues requiring redesign  
❌ LLM output quality is consistently poor (<50% usable)  
❌ The tool feels slower than manual coding with no productivity benefit  
❌ Users would not use this again  

### Pivot IF:

🔄 Specific phases need major rework (iterate on Phase 1)  
🔄 Different workflow structure would work better (redesign, then retry)  
🔄 LLM limitations are fundamental (wait for better models or adjust approach)  

---

## Phase 2 Prerequisites

If proceeding to Phase 2, ensure:

| Prerequisite | Verification |
|--------------|--------------|
| Core components are reusable | Components can be imported as library |
| State format is stable | Session JSON structure documented |
| Prompts are versioned | Can track changes to prompts |
| Error handling is robust | All error scenarios have recovery |
| Code is documented | JSDoc on public APIs |

---

## Sign-Off Checklist

Before declaring Phase 1 complete:

### Functional Sign-Off

- [ ] Complete workflow demonstrated (recorded or witnessed)
- [ ] Session persistence verified
- [ ] Crash recovery verified
- [ ] Go-back transitions verified
- [ ] All artifacts generated correctly

### Quality Sign-Off

- [ ] All automated tests passing
- [ ] No known blocking bugs
- [ ] Error handling tested

### Documentation Sign-Off

- [ ] LEARNINGS.md completed
- [ ] Evaluation questions answered
- [ ] Go/No-Go decision documented
- [ ] Phase 2 prerequisites checked

### Team Sign-Off

- [ ] Demo to stakeholders (if applicable)
- [ ] Team agreement on next steps

---

## Post Phase 1 Actions

### If Proceeding to Phase 2:

1. Archive Phase 1 branch/tag
2. Create Phase 2 planning document
3. Prioritize Phase 2 features based on learnings
4. Schedule Phase 2 kickoff

### If Iterating on Phase 1:

1. Document specific issues to address
2. Create iteration plan
3. Set new completion target
4. Re-evaluate after iteration

### If Pivoting:

1. Document why current approach isn't working
2. Archive current work
3. Design alternative approach
4. Create new plan

---

## Appendix: Learnings Template

```markdown
# Phase 1 Learnings

## Summary
[One paragraph summary of Phase 1 outcomes]

## What Worked Well
- [Item 1]
- [Item 2]
- ...

## What Didn't Work
- [Item 1]: [What happened, why, potential fix]
- [Item 2]: ...

## Friction Points
| Point | Severity | Phase 2 Priority |
|-------|----------|------------------|
| [Description] | High/Med/Low | Must fix / Should fix / Nice to have |

## LLM Observations
### Planning Phase
- Output quality: [1-5 rating]
- Issues: [specific problems]
- Prompt changes made: [iterations]

### Interface Phase
- Output quality: [1-5 rating]
- Issues: [specific problems]
- Prompt changes made: [iterations]

### Implementation Phase
- Output quality: [1-5 rating]
- Issues: [specific problems]
- Prompt changes made: [iterations]

### Testing Phase
- Output quality: [1-5 rating]
- Issues: [specific problems]
- Prompt changes made: [iterations]

## Time Analysis
| Activity | Expected | Actual |
|----------|----------|--------|
| Feature X with Pear | [estimate] | [actual] |
| Manual coding estimate | [estimate] | N/A |

## Phase 2 Recommendations
1. [Most important change]
2. [Second most important]
3. ...

## Questions for Phase 2
- [Unresolved question 1]
- [Unresolved question 2]

## Decision
[ ] Proceed to Phase 2
[ ] Iterate on Phase 1
[ ] Pivot approach

### Rationale
[Explanation of decision]
```

---

## Related Documents

- [00-overview.md](./00-overview.md) — Original objectives
- [10-implementation-plan.md](./10-implementation-plan.md) — Implementation timeline
- [README.md](./README.md) — Phase 1 index

