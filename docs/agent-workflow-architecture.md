# Agent Workflow Architecture — Full Pipeline Reference

## System Overview

OhMyOpenCode uses a **multi-agent pipeline** with strict separation of concerns:

```
User Request
    │
    ▼
┌─────────────────────────────────┐
│  PROMETHEUS (Planner)           │ ← Primary agent the user talks to
│  Interview → Plan Generation    │
│  Invokes: explore, librarian,   │
│           metis, oracle, momus  │
└──────────────┬──────────────────┘
               │  Outputs: .sisyphus/plans/{name}.md
               ▼
┌─────────────────────────────────┐
│  /start-work (Command/Hook)     │ ← User triggers execution
│  Creates boulder.json           │
│  Switches agent to Atlas        │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  ATLAS (Orchestrator)           │ ← Executes the plan
│  Delegates via task()           │
│  Invokes: sisyphus-junior,      │
│           explore, librarian,   │
│           oracle                │
└─────────────────────────────────┘
```

---

## All Agents — Summary Table

| Agent | Role | Mode | Can Write Files? | Can Invoke Sub-Agents? | Temperature |
|-------|------|------|-----------------|----------------------|-------------|
| **Sisyphus** | General-purpose orchestrator (direct work) | `all` | Yes | Yes (`task()`) | thinking: 32k tokens |
| **Prometheus** | Strategic planner / interviewer | `primary` | `.sisyphus/*.md` only | Yes (`task()`) | N/A (inherits) |
| **Atlas** | Plan execution orchestrator | `all` | No (delegates everything) | Yes (`task()`) | 0.1 |
| **Metis** | Pre-planning consultant | `subagent` | No (read-only) | No (restricted) | 0.3 |
| **Momus** | Plan reviewer | `subagent` | No (read-only) | No (restricted) | 0.1 |
| **Oracle** | Strategic advisor / debugger | `subagent` | No (read-only) | No (restricted) | 0.1 |
| **Explore** | Codebase search specialist | `subagent` | No (read-only) | No (restricted) | 0.1 |
| **Librarian** | External docs / OSS research | `subagent` | No (read-only) | No (restricted) | 0.1 |
| **Multimodal Looker** | Visual/screenshot analysis | `subagent` | No (read-only) | No (restricted) | N/A |
| **Hephaestus** | Implementation agent (alternative primary) | `primary` | Yes | Yes (`task()`) | N/A |
| **Sisyphus-Junior** | Worker agent spawned by category | N/A | Yes | Limited | N/A |

---

## Stage 1: INTERVIEW MODE (Prometheus)

### Entry Point

User sends a message to Prometheus (the planning agent).

### Step 0 — Intent Classification

```
classifyIntent(userMessage: string) → IntentType
```

**Input**: Raw user message
**Output**: One of `Trivial | Simple | Refactoring | BuildFromScratch | MidSized | Collaborative | Architecture | Research`
**Side Effects**: None (pure classification)

**Routing by intent**:
- `Trivial` → Skip heavy interview, quick confirm → propose action
- `Simple` → 1-2 targeted questions → propose approach
- `Complex` (all others) → Full intent-specific deep interview

### Step 1 — Silent Research (Background Agents)

```typescript
// For BUILD FROM SCRATCH / ARCHITECTURE / RESEARCH intents:

task(
  subagent_type = "explore",
  load_skills = [],
  run_in_background = true,
  prompt = "[CONTEXT] + [GOAL] + [DOWNSTREAM] + [REQUEST]"
)
// Input:  Structured 4-section prompt about codebase patterns
// Output: File paths, patterns, conventions discovered
// Side Effects: None (read-only)

task(
  subagent_type = "librarian",
  load_skills = [],
  run_in_background = true,
  prompt = "[CONTEXT] + [GOAL] + [DOWNSTREAM] + [REQUEST]"
)
// Input:  Structured prompt about external docs/best practices
// Output: Official docs, OSS examples, security guidance
// Side Effects: None (read-only, may clone repos to /tmp/)

// For ARCHITECTURE intent specifically:
task(
  subagent_type = "oracle",
  load_skills = [],
  run_in_background = false,     // Synchronous — blocks until done
  prompt = "Architecture consultation: [context]..."
)
// Input:  Architecture question with gathered context
// Output: Bottom-line recommendation + action plan + effort estimate
// Side Effects: None (read-only)
```

**Multiple explore/librarian agents fire in parallel** (`run_in_background=true`).
Results collected later via `background_output(task_id="...")`.

### Step 2 — Create Draft (External Memory)

```typescript
Write(
  file_path = ".sisyphus/drafts/{topic-slug}.md",
  content = `# Draft: {Topic}
## Requirements (confirmed)
## Technical Decisions
## Research Findings
## Open Questions
## Scope Boundaries`
)
```

**Input**: Topic slug, initial structure
**Output**: Draft file on disk
**Side Effects**: Creates `.sisyphus/drafts/{topic-slug}.md`

**Updated after EVERY meaningful user exchange** via `Edit()`.

### Step 3 — Interview Execution

Prometheus asks intent-specific questions informed by research results.

**For BUILD**: "Found pattern X in codebase. Follow or deviate?"
**For REFACTORING**: "What specific behavior must be preserved?"
**For MID-SIZED**: "What are EXACT outputs? What must NOT be included?"

```typescript
// Structured option presentation:
Question({
  questions: [{
    question: "Which auth provider?",
    header: "Auth",
    options: [
      { label: "OAuth 2.0", description: "..." },
      { label: "JWT", description: "..." },
    ]
  }]
})
```

**Input**: Question text + options
**Output**: User's selection
**Side Effects**: None

### Step 4 — Clearance Check (After Every Turn)

```
CLEARANCE CHECKLIST:
□ Core objective clearly defined?
□ Scope boundaries established (IN/OUT)?
□ No critical ambiguities remaining?
□ Technical approach decided?
□ Test strategy confirmed (TDD/tests-after/none)?
□ No blocking questions outstanding?

→ ALL YES → Auto-transition to Plan Generation (Stage 2)
→ ANY NO  → Ask the specific unclear question
```

**Input**: Current state of interview (from draft + conversation)
**Output**: Boolean — proceed to plan generation or continue interview
**Side Effects**: None (mental checklist, not persisted)

---

## Stage 2: PLAN GENERATION (Prometheus)

**Trigger**: Clearance check passes OR user says "Create the work plan"

### Step 2.0 — Register Todo List

```typescript
TodoWrite([
  { content: "Consult Metis for gap analysis", status: "pending", activeForm: "Consulting Metis..." },
  { content: "Generate plan to .sisyphus/plans/{name}.md", status: "pending", activeForm: "Generating plan..." },
  { content: "Self-review: classify gaps", status: "pending", activeForm: "Self-reviewing..." },
  { content: "Present summary with decisions needed", status: "pending", activeForm: "Presenting summary..." },
  { content: "If decisions needed: wait for user", status: "pending", activeForm: "Waiting for decisions..." },
  { content: "Ask about high accuracy mode", status: "pending", activeForm: "Asking about accuracy..." },
  { content: "If high accuracy: Submit to Momus", status: "pending", activeForm: "Momus review loop..." },
  { content: "Delete draft, guide to /start-work", status: "pending", activeForm: "Cleaning up..." },
])
```

**Input**: Fixed 8-step plan generation checklist
**Output**: Todo list visible to user
**Side Effects**: UI todo list updated

### Step 2.1 — Metis Consultation (MANDATORY)

```typescript
task(
  subagent_type = "metis",
  load_skills = [],
  run_in_background = false,    // Synchronous
  prompt = `Review this planning session:
    **User's Goal**: {summary}
    **What We Discussed**: {key points}
    **My Understanding**: {interpretation}
    **Research Findings**: {findings}

    Identify: missed questions, guardrails, scope creep,
    unvalidated assumptions, missing acceptance criteria, edge cases`
)
```

**Input**: Full interview summary (goal, discussion points, interpretation, research)
**Output**: Structured analysis:

```markdown
## Intent Classification
**Type**: [Refactoring | Build | Mid-sized | ...]

## Pre-Analysis Findings
[explore/librarian results]

## Questions for User
1. [Priority question]

## Identified Risks
- [Risk]: [Mitigation]

## Directives for Prometheus
### Core Directives
- MUST: [Required action]
- MUST NOT: [Forbidden action]
- PATTERN: Follow [file:lines]

### QA/Acceptance Criteria Directives
- MUST: Write acceptance criteria as executable commands
- MUST NOT: Create criteria requiring human intervention
```

**Side Effects**: None (Metis is read-only). Findings incorporated silently into plan.

### Step 2.2 — Generate Plan File

```typescript
// Step A: Write skeleton
Write(
  file_path = ".sisyphus/plans/{plan-name}.md",
  content = `# {Plan Title}
## TL;DR
## Context
## Work Objectives
## Verification Strategy
## Execution Strategy
---
## TODOs
---
## Final Verification Wave
## Commit Strategy
## Success Criteria`
)

// Step B: Edit-append tasks in batches of 2-4
Edit(
  file_path = ".sisyphus/plans/{plan-name}.md",
  old_string = "---\n\n## Final Verification Wave",
  new_string = "- [ ] 1. Task Title\n  **What to do**: ...\n  **QA Scenarios**: ...\n\n---\n\n## Final Verification Wave"
)
// Repeat for each batch of tasks

// Step C: Verify completeness
Read(file_path = ".sisyphus/plans/{plan-name}.md")
```

**Input**: Interview findings + Metis directives + research results
**Output**: Complete plan file at `.sisyphus/plans/{plan-name}.md`
**Side Effects**: Creates plan file on disk

**Plan Task Format** (per TODO item):

```markdown
- [ ] N. {Task Title}
  **What to do**: [steps]
  **Must NOT do**: [exclusions]
  **Recommended Agent Profile**: [category + skills + rationale]
  **Parallelization**: Wave N | Blocks: [tasks] | Blocked By: [tasks]
  **References**: [pattern files] [API files] [test patterns]
  **Acceptance Criteria**: [agent-executable conditions]
  **QA Scenarios**: [happy path] [error/edge cases]
  **Commit**: YES/NO | Message: type(scope): desc
```

### Step 2.3 — Self-Review

```
Gap Classification:
- CRITICAL (requires user input) → [DECISION NEEDED: ...]
- MINOR (can self-resolve) → Fix silently, note in summary
- AMBIGUOUS (default exists) → Apply default, disclose
```

**Input**: Generated plan
**Output**: Classified gaps
**Side Effects**: Plan may be updated with fixes for MINOR gaps

### Step 2.4 — Present Summary to User

```markdown
## Plan Generated: {plan-name}

**Key Decisions Made**: ...
**Scope**: IN: [...] / OUT: [...]
**Guardrails Applied**: ...
**Auto-Resolved**: ...
**Defaults Applied**: ...
**Decisions Needed** (if any): ...

Plan saved to: .sisyphus/plans/{name}.md
```

### Step 2.5 — Offer Next Step

```typescript
Question({
  questions: [{
    question: "Plan is ready. How would you like to proceed?",
    header: "Next Step",
    options: [
      { label: "Start Work", description: "Execute now with /start-work" },
      { label: "High Accuracy Review", description: "Have Momus verify every detail" }
    ]
  }]
})
```

**Input**: None
**Output**: User's choice
**Side Effects**: Routes to Stage 3 or Stage 4

---

## Stage 3: HIGH ACCURACY MODE — Momus Review Loop (Optional)

**Trigger**: User selects "High Accuracy Review"

```typescript
while (true) {
  const result = task(
    subagent_type = "momus",
    load_skills = [],
    run_in_background = false,
    prompt = ".sisyphus/plans/{plan-name}.md"   // ONLY the path
  )
  // Input:  Plan file path (Momus reads it)
  // Output: "[OKAY]" or "[REJECT]" with max 3 blocking issues
  // Side Effects: None (read-only)

  if (result.verdict === "OKAY") break

  // Fix ALL issues Momus raised
  Edit(".sisyphus/plans/{plan-name}.md", ...)
  // Loop — resubmit to Momus
}
```

**Momus Review Process**:
1. Validate input (extract single `.sisyphus/plans/*.md` path)
2. Read plan → identify tasks and file references
3. Verify references → do files exist? do they contain claimed content?
4. Executability check → can each task be started?
5. Decide → any BLOCKING issues? No = `[OKAY]`. Yes = `[REJECT]` (max 3 issues)

**Momus says OKAY when**:
- 100% of file references verified
- ≥80% of tasks have clear reference sources
- ≥90% of tasks have concrete acceptance criteria
- Zero contradictions or impossible requirements

### Cleanup

```typescript
// Delete draft (interview notes no longer needed)
Bash("rm .sisyphus/drafts/{name}.md")

// Guide user
"Plan saved. Run /start-work to begin execution."
```

---

## Stage 4: EXECUTION — /start-work + Atlas

### Step 4.0 — /start-work Command (Hook)

User runs `/start-work [plan-name]`.

```typescript
// Find available plans
findPrometheusPlans(directory: string): string[]
// Input:  Project root directory
// Output: Array of .sisyphus/plans/*.md paths, sorted by mtime
// Side Effects: None

// Check existing boulder state
readBoulderState(directory: string): BoulderState | null
// Input:  Project root directory
// Output: { active_plan, started_at, session_ids, plan_name, agent?, worktree_path? } | null
// Side Effects: None

// Parse plan progress
getPlanProgress(planPath: string): PlanProgress
// Input:  Absolute path to plan file
// Output: { total: number, completed: number, isComplete: boolean }
// Side Effects: None (counts `- [ ]` vs `- [x]` checkboxes)

// Create/update boulder state
writeBoulderState(directory: string, state: BoulderState): boolean
// Input:  Directory + state object
// Output: Success boolean
// Side Effects: Writes .sisyphus/boulder.json

createBoulderState(planPath, sessionId, agent?, worktreePath?): BoulderState
// Input:  Plan path, current session ID, optional agent name, optional worktree
// Output: New BoulderState object
// Side Effects: None (in-memory only, writeBoulderState persists)
```

**Decision Logic**:
- `boulder.json` exists AND plan incomplete → Resume (append session_id)
- No active plan OR plan complete → List plans for selection
- One plan available → Auto-select
- Multiple plans → Ask user to choose

### Step 4.1 — Atlas Analyzes Plan

```typescript
// Read the full plan
Read(".sisyphus/plans/{plan-name}.md")

// Parse tasks
// Input:  Plan content
// Output: Task list with parallelization metadata
// Side Effects: None

// Register tracking
TodoWrite([{
  content: "Complete ALL tasks in work plan",
  status: "in_progress",
  activeForm: "Orchestrating plan..."
}])
```

**Task Analysis Output**:
```
TASK ANALYSIS:
- Total: [N], Remaining: [M]
- Parallelizable Groups: [list of wave groups]
- Sequential Dependencies: [dependency chain]
```

### Step 4.2 — Initialize Notepad

```bash
mkdir -p .sisyphus/notepads/{plan-name}
```

Creates:
```
.sisyphus/notepads/{plan-name}/
  learnings.md    # Conventions, patterns discovered
  decisions.md    # Architectural choices made
  issues.md       # Problems, gotchas encountered
  problems.md     # Unresolved blockers
```

**Input**: Plan name
**Output**: Directory structure
**Side Effects**: Creates notepad directory and files

### Step 4.3 — Execute Tasks (Main Loop)

For each incomplete task in the plan:

#### A. Read Notepad (Inherited Wisdom)

```typescript
Read(".sisyphus/notepads/{plan-name}/learnings.md")
Read(".sisyphus/notepads/{plan-name}/issues.md")
// Input:  Notepad file paths
// Output: Accumulated knowledge from previous tasks
// Side Effects: None
```

#### B. Delegate via task()

```typescript
// Option A: Category + Skills (spawns Sisyphus-Junior)
task(
  category = "[category-name]",        // e.g., "quick", "deep", "visual-engineering"
  load_skills = ["skill-1", "skill-2"],// e.g., ["playwright", "frontend-ui-ux"]
  run_in_background = false,           // NEVER background for execution tasks
  prompt = `
    ## 1. TASK
    [Exact checkbox item from plan]

    ## 2. EXPECTED OUTCOME
    - [ ] Files created/modified: [exact paths]
    - [ ] Functionality: [exact behavior]
    - [ ] Verification: [command] passes

    ## 3. REQUIRED TOOLS
    - [tool]: [what to search/check]

    ## 4. MUST DO
    - Follow pattern in [reference file:lines]
    - Append findings to notepad

    ## 5. MUST NOT DO
    - Do NOT modify files outside [scope]
    - Do NOT skip verification

    ## 6. CONTEXT
    ### Notepad Paths
    - READ/WRITE: .sisyphus/notepads/{plan-name}/*.md
    ### Inherited Wisdom
    [From notepad]
    ### Dependencies
    [What previous tasks built]
  `
)

// Option B: Specialized Agent
task(
  subagent_type = "[agent-name]",      // e.g., "explore", "oracle"
  load_skills = [],
  run_in_background = false,
  prompt = "..."
)
```

**Input**: 6-section prompt (TASK, EXPECTED OUTCOME, REQUIRED TOOLS, MUST DO, MUST NOT DO, CONTEXT)
**Output**: Task completion result + session_id
**Side Effects**: Files created/modified by the sub-agent

#### C. Parallel Execution

```typescript
// Independent tasks fire in ONE message:
task(category="quick", ..., prompt="Task 2...")  // Parallel
task(category="quick", ..., prompt="Task 3...")  // Parallel
task(category="quick", ..., prompt="Task 4...")  // Parallel
// All run simultaneously, results collected together
```

### Step 4.4 — Verify (MANDATORY After Every Delegation)

```typescript
// A. Automated Verification
Bash("lsp_diagnostics(filePath='.')")  // → ZERO errors
Bash("bun run build")                  // → exit code 0
Bash("bun test")                       // → ALL pass

// B. Manual Code Review (NON-NEGOTIABLE)
Read("[every file the subagent modified]")
// Line-by-line verification:
// - Logic matches requirements?
// - No stubs/TODOs/placeholders?
// - Follows codebase patterns?
// - Imports correct?

// C. Hands-On QA (if applicable)
// Frontend → Playwright
// CLI/TUI  → interactive_bash
// API      → curl

// D. Check Boulder State
Read(".sisyphus/plans/{plan-name}.md")
// Count remaining `- [ ]` tasks
```

**Input**: Modified file paths, test commands
**Output**: Pass/fail per check
**Side Effects**: None (read-only verification)

### Step 4.5 — Handle Failures (Session Resume)

```typescript
// ALWAYS resume same session on failure:
task(
  session_id = "ses_xyz789",           // From failed task output
  load_skills = [...],
  run_in_background = false,
  prompt = "FAILED: {actual error}. Fix by: {specific instruction}"
)
// Input:  Previous session_id + error details
// Output: Fixed result
// Side Effects: Subagent retains full context from prior attempt

// Max 3 retries with same session
// After 3 failures → document in notepad, continue to independent tasks
```

### Step 4.6 — Final Report

```
ORCHESTRATION COMPLETE

TODO LIST: .sisyphus/plans/{name}.md
COMPLETED: N/N
FAILED: [count]

EXECUTION SUMMARY:
- Task 1: SUCCESS (category)
- Task 2: SUCCESS (agent)

FILES MODIFIED: [list]

ACCUMULATED WISDOM: [from notepad]
```

---

## Sub-Agent Detail: How Each Agent Works Internally

### Explore Agent

```
createExploreAgent(model: string) → AgentConfig
```

**Purpose**: Codebase search specialist — "contextual grep"
**Restrictions**: Read-only (`write`, `edit`, `apply_patch`, `task` all denied)
**Internal Process**:
1. Wrap analysis in `<analysis>` tags (Literal Request → Actual Need → Success Criteria)
2. Launch 3+ tools simultaneously (grep, glob, LSP, ast_grep)
3. Return structured `<results>` with absolute paths + explanation

**Tool Strategy**:
- Semantic search → LSP tools
- Structural patterns → `ast_grep_search`
- Text patterns → `grep`
- File patterns → `glob`
- History → git commands

### Librarian Agent

```
createLibrarianAgent(model: string) → AgentConfig
```

**Purpose**: External documentation & OSS research
**Restrictions**: Read-only (same as Explore)
**Internal Process**:
1. Classify request: TYPE A (conceptual) / B (implementation) / C (context) / D (comprehensive)
2. Doc Discovery Phase: `websearch` → version check → sitemap → targeted fetch
3. Main Phase: `context7`, `grep_app`, `gh clone`, `webfetch` in parallel
4. Synthesis: Every claim cited with GitHub permalink

**Tools Available**: `context7`, `websearch`, `webfetch`, `gh` CLI, `grep_app`, `git blame/log`

### Oracle Agent

```
createOracleAgent(model: string) → AgentConfig
```

**Purpose**: Strategic technical advisor — high-IQ reasoning
**Restrictions**: Read-only
**Internal Process**: Extended thinking (32k budget for Claude, medium reasoning for GPT)
**Output Format**: Bottom line (2-3 sentences) + Action plan (≤7 steps) + Effort estimate

### Metis Agent

```
createMetisAgent(model: string) → AgentConfig
```

**Purpose**: Pre-planning consultant — catch what Prometheus missed
**Restrictions**: Read-only (with `task` nominally restricted but accessible for explore/librarian)
**Temperature**: 0.3 (slightly creative for gap-finding)
**Internal Process**:
1. Classify intent (same taxonomy as Prometheus)
2. Launch explore/librarian if BUILD/RESEARCH intent
3. Analyze for: missed questions, guardrails needed, scope creep, AI-slop patterns
4. Output: Structured directives for Prometheus

### Momus Agent

```
createMomusAgent(model: string) → AgentConfig
```

**Purpose**: Practical plan reviewer — blocker-finder, not perfectionist
**Restrictions**: Read-only
**Internal Process**:
1. Extract plan path from input
2. Read plan file
3. For each task: verify file references exist, check executability
4. Verdict: `[OKAY]` (default when no blockers) or `[REJECT]` (max 3 specific issues)

**Approval bias**: When in doubt, APPROVE.

---

## Data Flow Diagram

```
┌──────────┐  user msg   ┌─────────────┐
│   User   │────────────→│  Prometheus  │
└──────────┘             │  (Planner)   │
     ▲                   └──────┬───────┘
     │                          │
     │ questions                │ task(subagent_type=...)
     │                          │ run_in_background=true
     │                          ▼
     │                   ┌─────────────┐
     │                   │  explore ×N  │──→ codebase patterns
     │                   │  librarian   │──→ external docs
     │                   │  oracle      │──→ architecture advice
     │                   └─────────────┘
     │                          │
     │                   results│
     │                          ▼
     │   informed        ┌─────────────┐
     │   questions       │  Prometheus  │
     │←──────────────────│  (Interview) │
     │                   └──────┬───────┘
     │                          │
     │ clearance=YES            │ task(subagent_type="metis")
     │                          ▼
     │                   ┌─────────────┐
     │                   │   Metis      │──→ gap analysis
     │                   └──────┬───────┘
     │                          │
     │                          ▼
     │                   ┌─────────────┐
     │                   │  Prometheus  │──→ .sisyphus/plans/{name}.md
     │                   │  (Generate)  │──→ .sisyphus/drafts/{name}.md (deleted)
     │                   └──────┬───────┘
     │                          │
     │                          │ (optional) task(subagent_type="momus")
     │                          ▼
     │                   ┌─────────────┐
     │                   │   Momus      │──→ [OKAY] or [REJECT]
     │                   │  (loop)      │
     │                   └──────┬───────┘
     │                          │
     │   /start-work            │
     │─────────────────→┌───────┴───────┐
                        │  start-work   │──→ .sisyphus/boulder.json
                        │  (hook)       │
                        └───────┬───────┘
                                │
                                ▼
                        ┌─────────────┐
                        │   Atlas      │
                        │(Orchestrator)│
                        └───────┬──────┘
                                │
                    task(category=...) │ task(subagent_type=...)
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                  ▼
       ┌────────────┐   ┌────────────┐     ┌────────────┐
       │ Sisyphus-Jr│   │ Sisyphus-Jr│     │  explore/   │
       │ (Worker 1) │   │ (Worker 2) │     │  oracle     │
       └──────┬─────┘   └──────┬─────┘     └────────────┘
              │                │
              ▼                ▼
       [code changes]   [code changes]
              │                │
              └───────┬────────┘
                      ▼
               ┌────────────┐
               │   Atlas     │
               │ (Verify)    │──→ lsp_diagnostics, build, test, manual read
               └──────┬──────┘
                      │
                      ▼
               Mark task complete in plan
               Read notepad → update notepad
               Next task...
```

---

## File System Artifacts

| Path | Created By | Read By | Purpose |
|------|-----------|---------|---------|
| `.sisyphus/drafts/{name}.md` | Prometheus (interview) | Prometheus | Interview working memory; deleted after plan generation |
| `.sisyphus/plans/{name}.md` | Prometheus (plan gen) | Momus, Atlas, start-work | The work plan with checkboxes |
| `.sisyphus/boulder.json` | start-work hook | Atlas, start-work hook | Active plan tracking, session IDs, worktree path |
| `.sisyphus/notepads/{name}/*.md` | Atlas | Atlas, sub-agents | Accumulated knowledge across tasks (learnings, decisions, issues) |
| `.sisyphus/evidence/task-{N}-*.{ext}` | Sub-agents (QA) | Atlas (verification) | Proof that QA scenarios passed |

---

## Key Type Definitions

```typescript
// src/agents/types.ts
type AgentMode = "primary" | "subagent" | "all"
type AgentCategory = "exploration" | "specialist" | "advisor" | "utility"
type AgentCost = "FREE" | "CHEAP" | "EXPENSIVE"
type BuiltinAgentName = "sisyphus" | "hephaestus" | "oracle" | "librarian"
                      | "explore" | "multimodal-looker" | "metis" | "momus" | "atlas"

// src/features/boulder-state/types.ts
interface BoulderState {
  active_plan: string       // Absolute path to plan file
  started_at: string        // ISO timestamp
  session_ids: string[]     // All sessions that worked on this
  plan_name: string         // Derived from filename
  agent?: string            // e.g., 'atlas'
  worktree_path?: string    // Git worktree root
}

interface PlanProgress {
  total: number             // Total checkboxes
  completed: number         // Completed checkboxes
  isComplete: boolean       // All done?
}

// src/agents/types.ts
interface AgentPromptMetadata {
  category: AgentCategory
  cost: AgentCost
  triggers: DelegationTrigger[]
  useWhen?: string[]
  avoidWhen?: string[]
  dedicatedSection?: string
  promptAlias?: string
  keyTrigger?: string
}
```

---

## Permission Matrix

| Agent | write | edit | apply_patch | task | bash | webfetch | question |
|-------|-------|------|-------------|------|------|----------|----------|
| **Prometheus** | `.sisyphus/*.md` only (hook-enforced) | same | — | allow | allow | allow | allow |
| **Metis** | deny | deny | deny | deny* | — | — | — |
| **Momus** | deny | deny | deny | deny | — | — | — |
| **Oracle** | deny | deny | deny | deny | — | — | — |
| **Explore** | deny | deny | deny | deny | — | — | — |
| **Librarian** | deny | deny | deny | deny | — | — | — |
| **Sisyphus** | allow | allow | allow | allow | allow | allow | allow |
| **Atlas** | delegates | delegates | delegates | allow | allow | — | — |

*Metis has `task` restricted but may use explore/librarian for pre-analysis.

---

## Agent Creation Functions

```typescript
// All agent factories follow the same pattern:
createExploreAgent(model: string): AgentConfig
createLibrarianAgent(model: string): AgentConfig
createOracleAgent(model: string): AgentConfig
createMetisAgent(model: string): AgentConfig
createMomusAgent(model: string): AgentConfig
createSisyphusAgent(model, availableAgents?, availableToolNames?, availableSkills?, availableCategories?, useTaskSystem?): AgentConfig
createAtlasAgent(ctx: OrchestratorContext): AgentConfig
createHephaestusAgent(model: string): AgentConfig

// Master builder:
createBuiltinAgents(
  disabledAgents: string[],
  agentOverrides: AgentOverrides,
  directory?: string,
  systemDefaultModel?: string,
  categories?: CategoriesConfig,
  gitMasterConfig?: GitMasterConfig,
  discoveredSkills: LoadedSkill[],
  customAgentSummaries?: unknown,
  browserProvider?: BrowserAutomationProvider,
  uiSelectedModel?: string,
  disabledSkills?: Set<string>,
  useTaskSystem?: boolean,
  disableOmoEnv?: boolean,
): Promise<Record<string, AgentConfig>>
```

---

## Model-Specific Routing

Each major agent has model-specific prompt variants:

| Agent | Claude (default) | GPT | Gemini |
|-------|-----------------|-----|--------|
| Prometheus | `system-prompt.ts` | `gpt.ts` | `gemini.ts` |
| Atlas | `default.ts` | `gpt.ts` | `gemini.ts` |
| Sisyphus | Base + overlays | `reasoningEffort: "medium"` | Gemini overlays |
| Oracle | `thinking: 32k` | `reasoningEffort: "medium"` | — |
| Momus | `thinking: 32k` | `reasoningEffort: "medium"` | — |
| Metis | `thinking: 32k` | — | — |

Detection:
```typescript
isGptModel(model): boolean    // checks for "gpt" in model name
isGeminiModel(model): boolean // checks for "gemini-" prefix or google/* provider
```
