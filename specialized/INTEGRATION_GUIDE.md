# Novel Agents Integration Guide

## Executive Summary

This document integrates the **26 Novel Production agents** into the **Agents Orchestrator** and **Boss** agent frameworks. The novel pipeline operates as a specialized 4-phase production system with score-driven iteration, parallel processing, and strict quality gates.

---

## 1. Novel Pipeline Architecture

### 1.1 Phase Structure

```
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 1: FOUNDATION (loop until foundation_score > 7.5)        │
│ World Builder → Character Architect → Outline Architect →       │
│ Voice Discovery → Foundation Evaluator                         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 2: DRAFTING (loop until all chapters score > 6.0)        │
│ Chapter Drafter → Chapter Evaluator (per chapter, sequential)  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 3: REVISION (min 3, max 6 cycles until plateau)          │
│ Cut Finder → Four Persona Panel → Revision Brief →             │
│ Chapter Reviser → Novel Evaluator → Dual Persona Reviewer      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 4: EXPORT (human checkpoint required)                    │
│ Outline Rebuilder → Summary Rebuilder → Manuscript Assembler → │
│ Visual Style Deriver → Audiobook Script Generator (optional)   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Score Thresholds & Exit Criteria

| Phase | Metric | Threshold | Action if Below |
|-------|--------|-----------|-----------------|
| Foundation | foundation_score | > 7.5 | Iterate (max 20) |
| Foundation | lore_score | > 7.0 | Iterate (max 20) |
| Drafting | chapter_score | > 6.0 | Retry chapter (max 5) |
| Revision | novel_score delta | < 0.3 for 2 cycles | Plateau detected |
| Revision | stars | ≥ 4.5 + no major unqualified | Stop revising |
| All | slop_penalty | < 2.0 | Fix slop before advance |

---

## 2. Orchestrator Integration

### 2.1 Orchestrator Workflow Mapping

The novel pipeline integrates as a **specialized workflow** within the Agents Orchestrator's phase-gated system:

```markdown
## Phase 1: Project Analysis & Planning
```bash
# Verify novel seed exists
ls -la project-specs/novel-seed.txt

# If missing, spawn Novel Idea Interviewer
"Please spawn novel-idea-interviewer to interview user about novel concept. Save responses to project-specs/novel-seed.txt"

# Spawn Novel Production Manager to orchestrate entire pipeline
"Please spawn NovelProductionManager to execute complete novel production pipeline from seed.txt through export. Track scores and iterations."
```

### Phase 2: Foundation Loop
```bash
# Foundation iteration loop (max 20)
LOOP:
  1. [PARALLEL] Spawn World Builder, Character Architect, Outline Architect
  2. [SEQUENTIAL] Spawn Voice Discovery Agent
  3. Spawn Foundation Evaluator → get foundation_score, lore_score
  4. IF score > prev_best:
     - COMMIT "foundation iter N: score X (lore Y)"
     - Update state
  5. ELSE: DISCARD, reset to last good state
  6. IF foundation_score > 7.5 AND lore_score > 7.0: EXIT LOOP
  7. ELSE: Retry with targeted improvements
```

### Phase 3: Drafting Loop
```bash
# Get total chapters from outline
CHAPTER_COUNT=$(grep -c "^### Ch" project-docs/outline.md)

# For each chapter (can parallelize 3-4 at a time)
FOR chapter in 1..CHAPTER_COUNT:
  LOOP (max 5 attempts):
    1. Spawn Chapter Drafter → produces ch_NN.md
    2. Spawn Chapter Evaluator → get chapter_score
    3. IF score > 6.0: COMMIT, break
    4. ELSE: retry or keep best after 5 attempts
  # Extract new canon entries → append to canon.md
```

### Phase 4: Revision Loop
```bash
# Revision cycles (min 3, max 6)
FOR cycle in 1..6:
  1. [PARALLEL] Spawn Cut Finder on all chapters
  2. Apply top cuts (OVER-EXPLAIN, REDUNDANT types)
  3. Spawn Four Persona Panel → consensus items
  4. FOR each consensus item (priority order):
     - Spawn Revision Brief Generator → brief.md
     - Spawn Chapter Reviser → revised chapter
     - Spawn Chapter Evaluator → check if improved
     - IF improved: COMMIT, ELSE: DISCARD
  5. Spawn Novel Evaluator → novel_score
  6. IF |novel_score - prev_novel_score| < 0.3 AND cycle >= 3:
     - PLATEAU DETECTED → spawn Dual Persona Reviewer
  7. prev_novel_score = novel_score
```

### Phase 5: Export
```bash
# Parallel export tasks
[PARALLEL]:
  - Spawn Outline Rebuilder
  - Spawn Summary Rebuilder
  - Spawn Visual Style Deriver

# Sequential assembly
- Spawn Manuscript Assembler → manuscript.md
- Spawn Audiobook Script Generator (optional)

# HUMAN CHECKPOINT required
REQUEST: "Novel export complete. Word count: N. Approval required before final commit."
IF approved: COMMIT "export: [title] — [word_count] words"
```
"
```

### 2.2 Novel Production Manager as Sub-Orchestrator

The **NovelProductionManager** agent (`novel-production-manager.md`) serves as a **sub-orchestrator** that:
- Manages all 26 novel agents
- Tracks iteration state and scores
- Makes KEEP/DISCARD decisions
- Handles retry logic
- Reports progress to parent Orchestrator

**State Tracking Format**:
```json
{
  "phase": "foundation|drafting|revision|export",
  "iteration": 0,
  "foundation_score": 0.0,
  "lore_score": 0.0,
  "chapters_completed": 0,
  "chapters_total": 0,
  "novel_score": 0.0,
  "revision_cycle": 0,
  "prev_novel_score": 0.0,
  "debts": []
}
```

### 2.3 Quality Gate Enforcement

The Orchestrator must enforce these **non-negotiable gates**:

```markdown
## Quality Gate Rules

1. **Foundation Gate**: NO drafting until foundation_score > 7.5 AND lore_score > 7.0
2. **Chapter Gate**: NO next chapter until current chapter_score > 6.0
3. **Revision Gate**: NO export until revision plateau detected (delta < 0.3 for 2 cycles)
4. **Human Gate**: NO final commit without human approval

**Retry Limits**:
- Foundation: max 20 iterations
- Chapter drafting: max 5 attempts per chapter
- Revision cycles: min 3, max 6
- After 5 failures on same task: ESCALATE to user
```

### 2.4 Parallel Processing Rules

**Independent tasks run in parallel**:
- Foundation: World Builder + Character Architect + Outline Architect (all parallel)
- Chapter evaluations: Multiple chapter evals after all drafted
- Cut-finding: All chapters simultaneously
- Four Persona Panel: Can run in parallel
- Export: Outline Rebuilder + Summary Rebuilder + Visual Style Deriver (parallel)

**Sequential dependencies**:
- Chapter N must complete before Chapter N+1 (canon continuity)
- Revision brief must precede chapter revision
- Dual Persona Reviewer after Novel Evaluator
- Manuscript Assembler after all export components ready

---

## 3. Boss Agent Integration

### 3.1 Boss Delegation Matrix for Novel Projects

When the Boss receives a novel-related request, use this delegation logic:

| Request Type | Primary Agent | Secondary Support | Tertiary |
|--------------|---------------|-------------------|----------|
| "Write a novel from idea" | **NovelProductionManager** | — | — |
| "Build novel foundation" | Novel Idea Interviewer → World Builder + Character Architect + Outline Architect | Voice Discovery | Foundation Evaluator |
| "Draft chapter N" | Chapter Drafter | Chapter Evaluator | — |
| "Evaluate chapter quality" | Chapter Evaluator | — | — |
| "Find cuts in prose" | Cut Finder | — | — |
| "Get reader feedback" | Four Persona Panel | — | — |
| "Revise chapter based on feedback" | Revision Brief Generator → Chapter Reviser | Chapter Evaluator | — |
| "Assess novel readiness" | Dual Persona Reviewer | Novel Evaluator | — |
| "Assemble final manuscript" | Manuscript Assembler | Outline Rebuilder + Summary Rebuilder | Visual Style Deriver |
| "Create audiobook script" | Audiobook Script Generator | — | — |

### 3.2 Boss Capability-Weighted Delegation

For novel tasks, weight agents as:

1. **Agent Capability Match (60%)**: Does this agent specialize in novel production?
   - NovelProductionManager: 100% for orchestration
   - Chapter Drafter: 100% for drafting
   - Foundation Evaluator: 100% for evaluation

2. **Specialty Alignment (25%)**: Is this within their domain?
   - World Builder: World creation (not character)
   - Character Architect: Character creation (not plot)
   - Outline Architect: Structure (not prose)

3. **Availability (15%)**: Can they take this on now?
   - Check if agent is already running in pipeline
   - Parallel tasks OK if no shared state

### 3.3 Boss Accountability Tracking for Novel Projects

```markdown
## Novel Delegation Record Template

### Delegation: [Task Name]
- **Agent**: [Agent Name]
- **Assigned**: [timestamp]
- **Expected Completion**: [timestamp]
- **Success Criteria**: [score threshold, deliverable format]
- **Actual Completion**: [timestamp]
- **Deliverable**: [file produced]
- **Quality Assessment**: [PASS/FAIL based on score]
- **Retry Count**: [N]
- **Final Status**: [COMPLETED/ESCALATED/DISCARDED]
```

### 3.4 Boss Multi-Agent Orchestration Patterns

#### Pattern 1: Foundation Build (Parallel + Sequential)

```
User Request: "Build a novel foundation from this seed"
     ↓
Boss → NovelProductionManager (orchestrates)
     ↓
     ├→ World Builder (parallel)
     ├→ Character Architect (parallel)
     ├→ Outline Architect (parallel)
     ↓
     → Voice Discovery Agent (sequential)
     ↓
     → Foundation Evaluator
     ↓
     IF score < 7.5: Loop back with improvements
     ELSE: Advance to drafting
```

#### Pattern 2: Chapter Drafting Loop

```
For each chapter:
     ↓
Boss → Chapter Drafter
     ↓
     → Chapter Evaluator
     ↓
     IF score < 6.0:
        ├→ Retry (max 5)
        └→ Keep best after 5
     ELSE: Commit, next chapter
```

#### Pattern 3: Revision Cycle

```
Revision Cycle N:
     ↓
     ├→ Cut Finder (all chapters parallel)
     ├→ Four Persona Panel (parallel)
     ↓
     → Revision Brief Generator (per consensus item)
     ↓
     → Chapter Reviser (per chapter)
     ↓
     → Chapter Evaluator (verify improvement)
     ↓
     → Novel Evaluator (whole novel score)
     ↓
     IF plateau detected:
        → Dual Persona Reviewer (final gate)
     ELSE: Next cycle
```

### 3.5 Boss Escalation Rules

**Escalate to user when**:
- Foundation stuck after 15 iterations (score < 7.5)
- Any chapter fails after 5 attempts (score < 6.0)
- Novel score drops significantly in revision
- Major structural revision needed (cutting entire chapters)
- Dual Persona Reviewer recommends stopping but score < target
- API costs spike beyond budget alert threshold

---

## 4. Key Roles & Responsibilities

### 4.1 Novel Production Manager

**Role**: Sub-orchestrator for novel pipeline
**Reports to**: Agents Orchestrator or Boss
**Manages**: All 26 novel agents

**Responsibilities**:
- Track phase state and iteration counts
- Make KEEP/DISCARD decisions after each iteration
- Manage retry logic within limits
- Coordinate parallel vs. sequential execution
- Report score progression to parent orchestrator
- Enforce human checkpoints

**Decision Authority**:
- KEEP iteration if score improved (any dimension)
- DISCARD if score regressed or contradictions introduced
- ESCALATE after 5 consecutive failures

### 4.2 Foundation Agents

| Agent | Role | Deliverable | Score Target |
|-------|------|-------------|--------------|
| **World Builder** | Creates world systems, geography, history, cultures | world.md | — |
| **Character Architect** | Builds psychologically deep characters with wound/want/need/lie chains | characters.md | — |
| **Outline Architect** | Constructs chapter-by-chapter outline with Save the Cat beats | outline.md | — |
| **Voice Discovery Agent** | Discovers narrative voice through trial passages | voice.md Part 2 | — |
| **Foundation Evaluator** | Evaluates all foundation docs against 13 dimensions | evaluation.md | foundation_score > 7.5, lore_score > 7.0 |

### 4.3 Drafting Agents

| Agent | Role | Deliverable | Score Target |
|-------|------|-------------|--------------|
| **Chapter Drafter** | Writes complete chapters (3000+ words) following voice | ch_NN.md | chapter_score > 6.0 |
| **Chapter Evaluator** | Evaluates chapters against voice, beats, prose quality, AI patterns | evaluation.md | chapter_score > 6.0 |

### 4.4 Revision Agents

| Agent | Role | Deliverable | Score Target |
|-------|------|-------------|--------------|
| **Cut Finder** | Identifies fat, redundancy, over-explanation in prose | cut-list.md | fat % < 15% |
| **Four Persona Panel** | Evaluates from 4 reader perspectives (Editor, Genre, Writer, First) | panel.md | consensus items |
| **Revision Brief Generator** | Generates revision briefs from consensus items | brief.md | — |
| **Chapter Reviser** | Rewrites chapters based on briefs | revised ch_NN.md | improved score |
| **Novel Evaluator** | Evaluates complete novel across 9 dimensions | novel-eval.md | plateau detection |
| **Dual Persona Reviewer** | Opus-level review as Literary Critic + Professor | review.md | stars ≥ 4.5, stop revising |

### 4.5 Export Agents

| Agent | Role | Deliverable | Human Checkpoint |
|-------|------|-------------|------------------|
| **Outline Rebuilder** | Rebuilds final outline from revised manuscript | final-outline.md | — |
| **Summary Rebuilder** | Rebuilds chapter summaries | summaries.md | — |
| **Manuscript Assembler** | Assembles all chapters into final manuscript | manuscript.md | REQUIRED |
| **Visual Style Deriver** | Derives visual style guide for adaptation | visual-style.md | — |
| **Audiobook Script Generator** | Generates audiobook narration script (optional) | audiobook.md | — |

---

## 5. Delegation Rules & Decision Logic

### 5.1 When to Delegate to Novel Production Manager

**Delegate when**:
- Request is "write a novel" or "produce a novel from [seed]"
- User wants full pipeline execution
- Multi-phase coordination required
- Score-driven iteration needed

**Do NOT delegate when**:
- Single task needed (e.g., "just draft chapter 3")
- Human creative direction needed first
- Unclear requirements or high ambiguity
- Cross-functional dependencies with non-novel agents

### 5.2 When to Delegate to Individual Novel Agents

**Direct delegation** (bypass NovelProductionManager):

| Scenario | Delegate To | Why |
|----------|-------------|-----|
| "Evaluate this chapter" | Chapter Evaluator | Single evaluation task |
| "Find cuts in chapter 5" | Cut Finder | Specific, isolated task |
| "Build the world for this idea" | World Builder | Foundation component only |
| "Review the complete novel" | Dual Persona Reviewer | Final quality gate only |
| "Assemble the manuscript" | Manuscript Assembler | Export task only |

### 5.3 Orchestrator Decision Logic

```markdown
## Decision Tree for Novel Pipeline

### Step 1: Check Seed Exists
IF no seed.txt:
  → Spawn Novel Idea Interviewer
  → Wait for completion
  → Proceed to Step 2
ELSE:
  → Proceed to Step 2

### Step 2: Choose Orchestration Mode
IF request is "write full novel":
  → Spawn NovelProductionManager (sub-orchestrator)
  → Monitor progress, enforce quality gates
ELSE IF request is single task:
  → Delegate to specific agent
  → Skip pipeline orchestration

### Step 3: Foundation Phase
LOOP (max 20 iterations):
  1. Spawn foundation agents (parallel where independent)
  2. Spawn Foundation Evaluator
  3. IF foundation_score > 7.5 AND lore_score > 7.0:
     → EXIT LOOP, advance to drafting
  4. ELSE IF iteration >= 20:
     → ESCALATE to user
  5. ELSE:
     → Target weakest dimension, retry

### Step 4: Drafting Phase
FOR each chapter (sequential):
  LOOP (max 5 attempts):
    1. Spawn Chapter Drafter
    2. Spawn Chapter Evaluator
    3. IF chapter_score > 6.0:
       → COMMIT, break, next chapter
    4. ELSE IF attempt >= 5:
       → Keep best, log debt, next chapter
    5. ELSE:
       → Retry with evaluator feedback

### Step 5: Revision Phase
FOR cycle in 1..6:
  1. Spawn Cut Finder (parallel all chapters)
  2. Spawn Four Persona Panel
  3. Apply consensus cuts (priority order)
  4. FOR each consensus item:
     - Spawn Revision Brief Generator
     - Spawn Chapter Reviser
     - Spawn Chapter Evaluator (verify)
  5. Spawn Novel Evaluator
  6. IF |novel_score - prev_novel_score| < 0.3 AND cycle >= 3:
     → PLATEAU DETECTED, proceed to Step 6
  7. ELSE:
     → Continue cycle

### Step 6: Final Review
Spawn Dual Persona Reviewer
IF stars >= 4.5 AND no major unqualified items:
  → READY FOR EXPORT
ELSE IF stars >= 4 AND qualified_items > 50%:
  → STOP REVISING (diminishing returns)
  → READY FOR EXPORT
ELSE:
  → Continue revision (max 6 cycles)

### Step 7: Export
1. Spawn export agents (parallel)
2. Spawn Manuscript Assembler
3. **HUMAN CHECKPOINT**: Request approval
4. IF approved:
   → COMMIT final manuscript
5. ELSE:
   → Hold for revision
```

### 5.4 Retry Logic & Escalation

```markdown
## Retry Limits

| Task | Max Retries | Escalation Trigger |
|------|-------------|-------------------|
| Foundation iteration | 20 | Score < 7.5 after 15 iterations |
| Chapter drafting | 5 per chapter | Score < 6.0 after 5 attempts |
| Revision cycle | 6 | No plateau after 6 cycles |
| Dual Persona Review | 4 rounds | Same issue persists 3+ rounds |

## Escalation Protocol

When escalation triggered:
1. Document failure pattern
2. Identify root cause (if possible)
3. Present options to user:
   - Continue with modified approach
   - Accept lower score threshold
   - Manual intervention required
   - Abort and discard work

User decision required before proceeding.
```

---

## 6. Quality Gates & Evidence Requirements

### 6.1 Foundation Quality Gate

**Evidence Required**:
- world.md exists with:
  - Speculative system with 3+ limitations
  - History with 3+ present-day tensions
  - Geography with distinct sensory signatures
  - Interconnection map (elements affect 2+ others)
- characters.md exists with:
  - Complete wound/want/need/lie chains (causally linked)
  - Three-slider profiles for all major characters
  - Voice profiles across 8 dimensions
  - Antagonist with coherent philosophy (not "evil")
- outline.md exists with:
  - 15 Save the Cat beats at correct percentages
  - 2-4 specific beats per chapter
  - Foreshadowing ledger (zero orphaned plants)
  - MICE thread map (reverse closure order)
- voice.md Part 2 exists with:
  - 3-5 exemplar passages
  - 3-4 anti-exemplars
  - Body-before-emotion mappings
  - Zero AI tells in exemplars
- canon.md has 50+ entries

**Exit Criteria**:
- foundation_score > 7.5
- lore_score > 7.0
- No major gaps in speculative system
- No unresolved contradictions
- Canon has 50+ entries

### 6.2 Chapter Quality Gate

**Evidence Required**:
- ch_NN.md exists (3000+ words)
- Chapter evaluation report with:
  - All 9 dimensions scored
  - 3 weakest + 3 strongest sentences quoted
  - AI patterns counted and listed
  - Beat coverage audited
  - Canon violations listed
  - Scene/summary ratio estimated
  - Slop penalty calculated

**Exit Criteria**:
- chapter_score > 6.0
- Voice matches exemplars
- Zero banned AI patterns
- 70%+ in-scene
- All beats dramatized
- All plants seeded

### 6.3 Revision Quality Gate

**Evidence Required**:
- Cut lists for all chapters
- Four Persona Panel consensus items
- Revision briefs generated
- Revised chapters with improved scores
- Novel evaluation across 9 dimensions
- Dual Persona Review with:
  - Star rating explicit
  - Severity classification
  - Qualified vs. unqualified items
  - Stopping condition evaluation

**Exit Criteria**:
- Revision plateau detected (delta < 0.3 for 2 cycles)
- Stars ≥ 4.5 OR qualified items > 50%
- No major unqualified items persisting
- Fat % < 15% across all chapters

### 6.4 Export Quality Gate

**Evidence Required**:
- Final manuscript assembled
- Outline rebuilt
- Summaries rebuilt
- Visual style derived
- Human approval recorded

**Exit Criteria**:
- All export components complete
- Human checkpoint passed
- Manuscript ready for publication

---

## 7. Communication & Reporting

### 7.1 Novel Production Manager Status Reports

```markdown
# Novel Production Status Report

## Phase Status
**Current Phase**: [foundation|drafting|revision|export]
**Iteration**: [N]
**Started**: [timestamp]

## Score Progression
**Foundation Score**: [X] (target: > 7.5)
**Lore Score**: [X] (target: > 7.0)
**Chapter Scores**: [Ch 1: X, Ch 2: X, ...]
**Novel Score**: [X]
**Previous Novel Score**: [X]
**Delta**: [X]

## Iteration History
**Last Iteration**: [N]
**Score Change**: [X] → [Y] ([+/-])
**Decision**: [KEEP|DISCARD]
**Reason**: [Why]

## Parallel Tasks Running
- [Task 1]
- [Task 2]
- [Task 3]

## Next Action
**Immediate**: [specific next step]
**Estimated Completion**: [time]
**Blockers**: [any]

---
**Report Time**: [timestamp]
**Status**: [ON_TRACK|DELAYED|BLOCKED]
```

### 7.2 Boss Delegation Reports

```markdown
# Novel Delegation Report

## What Was Delegated
**Task**: [Task description]
**Agent**: [Agent name]
**Timestamp**: [assigned]

## Deliverables
**File Produced**: [filename]
**Quality Assessment**: [PASS/FAIL]
**Score**: [if applicable]

## Accountability Record
**Completed**: [Y/N]
**On Time**: [Y/N]
**Retry Count**: [N]
**Final Status**: [COMPLETED|ESCALATED|DISCARDED]

## Key Findings
[Brief summary of what was learned/produced]

## Next Steps
[What happens next, if anything]

---
**Reported By**: Boss
**Timestamp**: [timestamp]
```

### 7.3 Orchestrator Completion Summary

```markdown
# Novel Pipeline Completion Report

## Pipeline Success Summary
**Title**: [novel title]
**Total Duration**: [start to finish]
**Final Status**: [COMPLETED|NEEDS_WORK|BLOCKED]

## Phase Results
**Foundation**: [X] iterations, final score [Y]
**Drafting**: [X] chapters, avg score [Y]
**Revision**: [X] cycles, plateau at score [Y]
**Export**: [completed|pending human approval]

## Agent Performance
**NovelProductionManager**: [orchestration quality]
**World Builder**: [foundation quality]
**Character Architect**: [character depth]
**Outline Architect**: [structure quality]
**Chapter Drafter**: [prose quality]
**Foundation Evaluator**: [evaluation rigor]
**Chapter Evaluator**: [prose sensitivity]
**Cut Finder**: [cut precision]
**Four Persona Panel**: [consensus quality]
**Dual Persona Reviewer**: [review sophistication]

## Quality Metrics
**Foundation Score**: [X]
**Avg Chapter Score**: [X]
**Final Novel Score**: [X]
**Dual Persona Stars**: [X]
**Fat %**: [X]%

## Production Readiness
**Status**: [READY|NEEDS_WORK|NOT_READY]
**Human Approval**: [granted|pending|denied]
**Quality Confidence**: [HIGH|MEDIUM|LOW]

---
**Pipeline Completed**: [timestamp]
**Reported By**: Agents Orchestrator
```

---

## 8. Memory & Iteration Tracking

### 8.1 Iteration Memory Format

After each iteration, record:

```json
{
  "phase": "foundation",
  "iteration": 5,
  "timestamp": "2026-03-24T10:30:00Z",
  "scores": {
    "foundation_score": 6.8,
    "lore_score": 6.5,
    "prev_foundation_score": 6.2,
    "prev_lore_score": 6.0
  },
  "delta": {
    "foundation": +0.6,
    "lore": +0.5
  },
  "dimensions_improved": ["Speculative System", "Character Depth"],
  "dimensions_regressed": [],
  "decision": "KEEP",
  "reason": "Score improved in foundation and lore, no contradictions introduced",
  "commit_message": "foundation iter 5: 6.2 → 6.8 (lore 6.5)"
}
```

### 8.2 Debt Tracking

Track unresolved issues:

```json
{
  "debts": [
    {
      "type": "chapter_score",
      "chapter": 5,
      "target": 6.0,
      "actual": 5.8,
      "attempts": 5,
      "resolution": "Accepted after max attempts, minor revision needed later"
    },
    {
      "type": "canon_contradiction",
      "description": "Character eye color differs between Ch 3 and Ch 7",
      "resolution": "To fix in revision cycle 2"
    }
  ]
}
```

### 8.3 State Persistence

Maintain state across sessions:

```json
{
  "project_id": "novel-[title]",
  "current_phase": "revision",
  "current_iteration": 12,
  "foundation_score": 8.2,
  "lore_score": 7.8,
  "chapters_completed": 24,
  "chapters_total": 24,
  "novel_score": 7.6,
  "revision_cycle": 3,
  "prev_novel_score": 7.4,
  "debts": [...],
  "last_commit": "revision cycle 3: 7.3 → 7.6",
  "human_checkpoints_passed": ["foundation_exit", "export_approval"]
}
```

---

## 9. Anti-Pattern Enforcement

### 9.1 Universal AI Pattern Bans

All novel agents must enforce these bans:

**BANNED WORDS**:
- delve, utilize, leverage, facilitate, elucidate, embark, endeavor, encompass, tapestry, myriad, plethora

**BANNED PHRASES**:
- "a sense of [emotion]"
- "couldn't help but [verb]"
- "the weight of [abstract noun]"
- "the air was thick with [emotion/tension]"
- "eyes widened"
- "[color] hair [spilled/cascaded/tumbled]"
- "a knowing smile"
- "heart pounded in [his/her] chest"

**BANNED STRUCTURES**:
- "Not X, but Y" formula (in narration)
- Triadic lists (X. Y. Z.)
- "He did not [verb]" more than once per chapter
- "He thought about [X]" — replace with thought or action
- "the way [X] did [Y]" more than twice
- Section breaks (---) more than twice
- Same paragraph structure three times in a row

**SENTENCE VARIATION REQUIREMENTS**:
- Never 3+ consecutive sentences starting the same way
- Include at least one 1-2 sentence paragraph
- Include at least one 6+ sentence paragraph
- Sentence length coefficient of variation > 0.3

### 9.2 Voice-Specific Anti-Patterns

Each novel must define voice-specific anti-patterns:

```markdown
## This Voice Is NOT:
- "Too modern" — example: [passage]
- "Too flowery" — example: [passage]
- "Too distant" — example: [passage]
- "Too generic" — example: [passage]
```

### 9.3 Slop Penalty Calculation

```python
# Slop penalty formula (from Chapter Evaluator)
slop_penalty = 0
if tier1_banned_words > 5: slop_penalty += 2
if tier2_cluster_words > 10: slop_penalty += 1
if fiction_tells > 3: slop_penalty += 2
if structural_tics > 5: slop_penalty += 1
if em_dash_density > 15: slop_penalty += 1
if sentence_cv < 0.3: slop_penalty += 1
if transition_ratio > 30: slop_penalty += 1

# Slop penalty > 2.0 = automatic rejection
```

---

## 10. Human Checkpoints

### 10.1 Required Human Checkpoints

**Mandatory human approval points**:

1. **Foundation Exit**: Before advancing to drafting
   - Review foundation_score, lore_score
   - Approve world, characters, outline, voice
   - Confirm ready for drafting

2. **Major Structural Revision**: Cutting entire chapters
   - Review cut recommendations
   - Approve structural changes
   - Confirm direction

3. **Export Approval**: Before final commit
   - Review complete manuscript
   - Approve publication
   - Confirm ready for distribution

4. **Budget Alerts**: If API costs spike
   - Review cost vs. progress
   - Approve continued spending
   - Set new budget limits

### 10.2 Human Checkpoint Format

```markdown
## Human Checkpoint Required

**Checkpoint**: [foundation_exit|structural_revision|export_approval|budget_alert]

**Status**: [phase complete, awaiting approval]

**Summary**:
[Brief summary of what was accomplished]

**Metrics**:
- Foundation Score: [X]
- Lore Score: [X]
- Chapters: [N]
- Novel Score: [X]
- Word Count: [N]

**Deliverables**:
- [File 1]
- [File 2]
- [File 3]

**Recommendation**:
[Agent recommendation: approve/review/revise]

**Action Required**:
Please review [deliverables] and confirm approval to proceed.

**Approval**: [ ] Approved  [ ] Needs Revision  [ ] Reject
```

---

## 11. Integration Examples

### 11.1 Example: Full Novel Pipeline via Orchestrator

```bash
# Single command to launch full pipeline
"Please spawn agents-orchestrator to execute complete novel production pipeline for project-specs/novel-seed.txt. Run autonomous workflow:
  1. Novel Idea Interviewer (if seed missing)
  2. NovelProductionManager → Foundation loop (World Builder + Character Architect + Outline Architect + Voice Discovery → Foundation Evaluator)
  3. Drafting loop (Chapter Drafter → Chapter Evaluator per chapter)
  4. Revision loop (Cut Finder + Four Persona Panel → Revision Brief → Chapter Reviser → Novel Evaluator → Dual Persona Reviewer)
  5. Export (Manuscript Assembler + export agents → human approval)
Each phase must meet score thresholds before advancing."
```

### 11.2 Example: Boss Delegating Novel Project

```markdown
## Boss Delegation Decision

**Request**: "Write a historical fiction novel about the fall of Rome"

**Analysis**:
- Core challenge: Full novel production
- Specialties needed: Historical research, world building, character development, drafting, revision
- Complexity: Multi-agent workflow (26 agents)
- Success criteria: Publication-quality manuscript
- Timeline: Autonomous pipeline execution

**Delegation Decision**:
Primary: **NovelProductionManager** (orchestrates entire pipeline)
Support: Historical Fiction Historian, Historical Fiction Geopolitical, Historical Fiction Anthropologist (domain expertise)

**Instructions to NovelProductionManager**:
"Execute complete novel production pipeline from seed. Integrate historical research specialists in foundation phase. Enforce score thresholds: foundation > 7.5, chapters > 6.0, novel plateau. Report progress after each phase."

**Accountability Record**:
- Delegated: [timestamp]
- Expected: [timeline]
- Checkpoints: Foundation exit, export approval
- Quality gates: Score thresholds enforced
```

### 11.3 Example: Parallel Chapter Evaluation

```bash
# After all chapters drafted, run evaluations in parallel
"Please spawn Chapter Evaluator to evaluate chapters 1-24. Run evaluations in parallel where no shared state. Aggregate results and identify chapters needing revision (score < 6.0)."

# Parallel execution spawns 24 Chapter Evaluator instances
# Results aggregated, chapters flagged for retry
```

---

## 12. Success Metrics

### 12.1 Pipeline Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Foundation score | > 7.5 | Foundation Evaluator |
| Lore score | > 7.0 | Foundation Evaluator |
| Chapter scores | > 6.0 each | Chapter Evaluator |
| Novel score | > 7.5 | Novel Evaluator |
| Dual Persona stars | ≥ 4.5 | Dual Persona Reviewer |
| Fat percentage | < 15% | Cut Finder |
| Slop penalty | < 2.0 | Chapter Evaluator |
| Iteration efficiency | < 10 foundation, < 4 revision | State tracking |

### 12.2 Agent Performance Metrics

| Agent | Success Metric |
|-------|---------------|
| NovelProductionManager | Pipeline completion rate, on-time phase exits |
| World Builder | Lore score contribution, interconnection depth |
| Character Architect | Character depth score, voice distinctiveness |
| Outline Architect | Structure score, beat placement accuracy |
| Chapter Drafter | Chapter score, voice adherence |
| Foundation Evaluator | Evaluation rigor, gap identification |
| Chapter Evaluator | Prose sensitivity, AI pattern detection |
| Cut Finder | Cut precision, fat % reduction |
| Four Persona Panel | Consensus quality, actionable items |
| Dual Persona Reviewer | Review sophistication, stopping condition accuracy |

### 12.3 Quality Confidence Indicators

**HIGH confidence**:
- Foundation score > 8.0
- All chapters > 7.0
- Novel score > 8.0
- Dual Persona stars ≥ 4.5
- Zero major unqualified items
- Human approval granted

**MEDIUM confidence**:
- Foundation score 7.5-8.0
- Chapters 6.0-7.0
- Novel score 7.0-8.0
- Dual Persona stars 4.0-4.5
- Some minor issues persist
- Human approval pending

**LOW confidence**:
- Foundation score barely > 7.5
- Chapters with debts (score < 6.0 accepted)
- Novel score < 7.0
- Dual Persona stars < 4.0
- Major issues unresolved
- Human approval denied

---

## Appendix A: Complete Novel Agent Roster

| # | Agent | Phase | Role |
|---|-------|-------|------|
| 1 | Novel Idea Interviewer | Foundation | User interview, seed creation |
| 2 | World Builder | Foundation | World systems, geography, history |
| 3 | Character Architect | Foundation | Character psychology, voice profiles |
| 4 | Outline Architect | Foundation | Chapter structure, beat placement |
| 5 | Voice Discovery Agent | Foundation | Narrative voice discovery |
| 6 | Foundation Evaluator | Foundation | 13-dimension evaluation |
| 7 | Chapter Drafter | Drafting | Chapter writing (3000+ words) |
| 8 | Chapter Evaluator | Drafting | Chapter quality evaluation |
| 9 | Cut Finder | Revision | Prose fat identification |
| 10 | Four Persona Panel | Revision | Multi-perspective evaluation |
| 11 | Revision Brief Generator | Revision | Revision brief creation |
| 12 | Chapter Reviser | Revision | Chapter rewriting |
| 13 | Novel Evaluator | Revision | Whole-novel evaluation |
| 14 | Dual Persona Reviewer | Revision | Final quality gate |
| 15 | Outline Rebuilder | Export | Final outline assembly |
| 16 | Summary Rebuilder | Export | Chapter summary assembly |
| 17 | Manuscript Assembler | Export | Final manuscript assembly |
| 18 | Visual Style Deriver | Export | Visual style guide |
| 19 | Audiobook Script Generator | Export | Audiobook script (optional) |
| 20 | Historical Fiction Historian | Specialized | Historical accuracy |
| 21 | Historical Fiction Geopolitical | Specialized | Geopolitical context |
| 22 | Historical Fiction Anthropologist | Specialized | Cultural accuracy |
| 23 | Historical Fiction Narratologist | Specialized | Narrative structure |
| 24 | Period Consultant | Specialized | Period-specific details |
| 25 | Novel Evaluator | Revision | Full novel assessment |
| 26 | Visual Style Deriver | Export | Visual adaptation guide |

---

## Appendix B: File Structure

```
project-specs/
  └── novel-seed.txt          # Initial novel idea

project-docs/
  ├── world.md                # World bible
  ├── characters.md           # Character registry
  ├── outline.md              # Chapter outline
  ├── voice.md                # Voice identity (Part 1 + Part 2)
  ├── canon.md                # Fact registry
  ├── historical-analysis.md  # (if historical fiction)
  └── geopolitical-analysis.md # (if geopolitical)

chapters/
  ├── ch_01.md                # Chapter 1
  ├── ch_02.md                # Chapter 2
  └── ...                     # All chapters

project-tasks/
  └── novel-tasklist.md       # Task breakdown

evaluations/
  ├── foundation-eval-N.md    # Foundation evaluations
  ├── chapter-N-eval.md       # Chapter evaluations
  ├── cut-list-N.md           # Cut lists
  ├── panel-N.md              # Four Persona Panel
  └── novel-eval-N.md         # Novel evaluations

revisions/
  ├── brief-N.md              # Revision briefs
  └── revised-ch-N.md         # Revised chapters

exports/
  ├── final-outline.md        # Final outline
  ├── summaries.md            # Chapter summaries
  ├── manuscript.md           # Final manuscript (HUMAN CHECKPOINT)
  ├── visual-style.md         # Visual style guide
  └── audiobook.md            # Audiobook script (optional)

state/
  └── novel-state.json        # Persistent state tracking
```

---

## Appendix C: Glossary

| Term | Definition |
|------|------------|
| **Foundation Score** | Weighted average of 13 foundation dimensions |
| **Lore Score** | Average of speculative system, history, geography, interconnection |
| **Chapter Score** | Average of 9 chapter dimensions minus slop penalty |
| **Novel Score** | Average of 9 novel dimensions |
| **Slop Penalty** | Deduction for AI patterns, mechanical slop |
| **Plateau** | Novel score delta < 0.3 for 2 consecutive cycles |
| **Qualified Item** | Hedged critique (diminishing returns signal) |
| **Unqualified Item** | Direct critique (requires action) |
| **KEEP** | Iteration committed (score improved) |
| **DISCARD** | Iteration rejected (score regressed) |
| **ESCALATE** | User intervention required |
| **Human Checkpoint** | Mandatory human approval point |

---

**Version**: 1.0
**Last Updated**: 2026-03-24
**Maintained By**: Agents Orchestrator + Boss
