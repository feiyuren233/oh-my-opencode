# Agent Artifacts — Disk-Based State Passing Between Agents

Agents in OhMyOpenCode are **stateless processes**. They pass information
to each other exclusively through files on disk. This document catalogs
every artifact: its exact schema, who writes it, who reads it, and when.

---

## THE FULL ARTIFACT LIFECYCLE AS PSEUDOCODE

```typescript
// ============================================================
// STAGE 1: INTERVIEW (Prometheus writes drafts)
// ============================================================

// First substantive exchange → create draft
Write(".sisyphus/drafts/{topic-slug}.md", `
# Draft: {Topic}

## Requirements (confirmed)
- [requirement]: [user's exact words]

## Technical Decisions
- [decision]: [rationale]

## Research Findings
- [source]: [key finding]

## Open Questions
- [question not yet answered]

## Scope Boundaries
- INCLUDE: [in scope]
- EXCLUDE: [explicitly out]

## Test Strategy Decision
- **Infrastructure exists**: YES/NO
- **Automated tests**: TDD / tests-after / none
- **Framework**: [bun test / vitest / jest / none]
`)

// Every subsequent turn → append via Edit
Edit(".sisyphus/drafts/{topic-slug}.md", ...)

// FORMAT: Free-form markdown. No schema enforced.
// LIFECYCLE: Created in interview, deleted after plan generation.
// WRITER: Prometheus only
// READERS: Prometheus (own memory across turns), user (can review)


// ============================================================
// STAGE 2: PLAN GENERATION (Prometheus writes plan)
// ============================================================

// Skeleton first
Write(".sisyphus/plans/{plan-name}.md", `
# {Plan Title}

## TL;DR
> **Quick Summary**: [1-2 sentences]
> **Deliverables**: [bullet list]
> **Estimated Effort**: [Quick | Short | Medium | Large | XL]
> **Parallel Execution**: [YES - N waves | NO]
> **Critical Path**: [Task X → Task Y → Task Z]

---

## Context

### Original Request
[User's initial description]

### Interview Summary
**Key Discussions**:
- [Point 1]: [decision]

**Research Findings**:
- [Finding 1]: [implication]

### Metis Review
**Identified Gaps** (addressed):
- [Gap 1]: [resolution]

---

## Work Objectives

### Core Objective
[1-2 sentences]

### Concrete Deliverables
- [file/endpoint/feature]

### Definition of Done
- [ ] [verifiable condition with command]

### Must Have
- [non-negotiable requirement]

### Must NOT Have (Guardrails)
- [explicit exclusion]

---

## Verification Strategy

> ZERO HUMAN INTERVENTION — all verification agent-executed.

### Test Decision
- **Infrastructure exists**: YES/NO
- **Automated tests**: TDD / tests-after / none
- **Framework**: [bun test / vitest / etc.]

### QA Policy
- Frontend/UI: Playwright
- TUI/CLI: interactive_bash (tmux)
- API/Backend: Bash (curl)
- Evidence: .sisyphus/evidence/task-{N}-{slug}.{ext}

---

## Execution Strategy

### Parallel Execution Waves
\`\`\`
Wave 1 (foundation):
├── Task 1: [title] [category]
├── Task 2: [title] [category]

Wave 2 (core, after Wave 1):
├── Task 3: [title] (depends: 1) [category]
├── Task 4: [title] (depends: 2) [category]

Wave FINAL (verification, after ALL):
├── F1: Plan compliance audit (oracle)
├── F2: Code quality review (unspecified-high)
├── F3: Real manual QA (unspecified-high)
├── F4: Scope fidelity check (deep)
\`\`\`

---

## TODOs

- [ ] 1. {Task Title}

  **What to do**:
  - [implementation steps]

  **Must NOT do**:
  - [exclusions]

  **Recommended Agent Profile**:
  - **Category**: \`[visual-engineering | deep | quick | ...]\`
  - **Skills**: [\`skill-1\`, \`skill-2\`]

  **Parallelization**:
  - **Wave**: N
  - **Blocks**: [tasks depending on this]
  - **Blocked By**: [dependencies] | None

  **References**:
  - Pattern: \`src/services/auth.ts:45-78\` — [why]
  - API: \`src/types/user.ts:UserDTO\` — [why]
  - Test: \`src/__tests__/auth.test.ts\` — [why]

  **Acceptance Criteria**:
  - [ ] \`bun test src/auth\` → PASS

  **QA Scenarios**:
  \`\`\`
  Scenario: [Happy path]
    Tool: [Playwright / interactive_bash / curl]
    Steps:
      1. [exact action]
      2. [assertion with exact expected value]
    Expected Result: [concrete, binary pass/fail]
    Evidence: .sisyphus/evidence/task-1-happy.png

  Scenario: [Error case]
    Tool: [same format]
    Steps:
      1. [trigger error condition]
      2. [assert graceful handling]
    Evidence: .sisyphus/evidence/task-1-error.png
  \`\`\`

  **Commit**: YES
  - Message: \`feat(auth): add login endpoint\`

---

## Final Verification Wave
[4 parallel review agents: compliance, quality, QA, scope]

## Success Criteria
\`\`\`bash
bun test        # Expected: all pass
bun run build   # Expected: exit 0
\`\`\`
`)

// Tasks appended in batches of 2-4 via Edit
Edit(".sisyphus/plans/{plan-name}.md", ...)

// FORMAT: Markdown with `- [ ]` / `- [x]` checkboxes. Free-form sections.
// PARSING: Regex — /^\s*[-*]\s*\[\s*\]/gm (unchecked) and /^\s*[-*]\s*\[[xX]\]/gm (checked)
// LIFECYCLE: Created in plan gen, read during execution, checkboxes toggled by Atlas.
// WRITER: Prometheus (creates), Atlas (toggles checkboxes)
// READERS: Momus (review), Atlas (execution), Sisyphus-Junior (via hook, READ-ONLY)

// Delete draft after plan is finalized
Bash("rm .sisyphus/drafts/{topic-slug}.md")


// ============================================================
// STAGE 3: /start-work (creates boulder.json)
// ============================================================

writeBoulderState(directory, {
  active_plan: "/abs/path/to/.sisyphus/plans/{plan-name}.md",
  started_at: "2026-03-11T14:30:00.000Z",   // ISO 8601
  session_ids: ["ses_abc123"],                // grows as sessions resume
  plan_name: "auth-refactor",                 // basename without .md
  agent: "atlas",                             // optional: which agent orchestrates
  worktree_path: "/abs/path/to/worktree",     // optional: git worktree root
})
// → writes .sisyphus/boulder.json (JSON, pretty-printed, 2-space indent)

// FORMAT: JSON. Schema enforced by TypeScript interface.
// LIFECYCLE: Created by /start-work, updated (session_ids appended) on resume,
//            deleted or replaced when plan completes.
// WRITER: /start-work hook (creates), Atlas (appends session_ids)
// READERS: Atlas (knows which plan to execute), continuation system (resume detection)


// ============================================================
// STAGE 4: EXECUTION (Atlas creates notepads, delegates work)
// ============================================================

// --- 4a. Atlas initializes notepad directory ---
Bash("mkdir -p .sisyphus/notepads/{plan-name}")
// Creates 4 files (initially empty):
Write(".sisyphus/notepads/{plan-name}/learnings.md", "")
Write(".sisyphus/notepads/{plan-name}/decisions.md", "")
Write(".sisyphus/notepads/{plan-name}/issues.md", "")
Write(".sisyphus/notepads/{plan-name}/problems.md", "")

// FORMAT: Markdown. Append-only. Timestamped entries.
// ENTRY FORMAT (convention, not enforced by code):
//   ## [TIMESTAMP] Task: {task-id}
//   {content}
//
// LIFECYCLE: Created when execution starts, grows throughout, persists after completion.
// WRITER: Sisyphus-Junior (appends after each task via hook directive)
// READERS: Atlas (reads before every delegation as "Inherited Wisdom")


// --- 4b. Hook injects notepad directive into every sub-agent prompt ---
// The sisyphus-junior-notepad hook automatically appends this to every task() prompt:
`
<Work_Context>
## Notepad Location
NOTEPAD PATH: .sisyphus/notepads/{plan-name}/
- learnings.md: Record patterns, conventions, successful approaches
- issues.md: Record problems, blockers, gotchas encountered
- decisions.md: Record architectural choices and rationales
- problems.md: Record unresolved issues, technical debt

You SHOULD append findings to notepad files after completing work.
IMPORTANT: Always APPEND — never overwrite or use Edit tool.

## Plan Location (READ ONLY)
PLAN PATH: .sisyphus/plans/{plan-name}.md
CRITICAL: NEVER MODIFY THE PLAN FILE. Only the Orchestrator manages it.
</Work_Context>
`


// --- 4c. Sisyphus-Junior writes to notepad after completing work ---
// Example: agent discovers a naming convention
Bash(`cat >> .sisyphus/notepads/{plan-name}/learnings.md << 'EOF'
## [2026-03-11T15:00:00Z] Task: 3
All service files use PascalCase and export a singleton instance.
Pattern: src/services/AuthService.ts → export const authService = new AuthService()
EOF`)

// Example: agent hits a gotcha
Bash(`cat >> .sisyphus/notepads/{plan-name}/issues.md << 'EOF'
## [2026-03-11T15:30:00Z] Task: 5
The Zod schema at src/schemas/user.ts uses .transform() which strips extra fields.
New fields must be added to the schema before they appear in API responses.
EOF`)


// --- 4d. Sisyphus-Junior saves QA evidence ---
// After running QA scenarios, evidence saved to .sisyphus/evidence/
Bash("playwright screenshot .sisyphus/evidence/task-3-login-happy.png")
Bash("curl -s http://localhost:3000/api/health > .sisyphus/evidence/task-5-health-check.json")
Bash("tmux capture-pane -p > .sisyphus/evidence/task-7-cli-output.txt")

// FORMAT: Varies by QA tool — .png (screenshots), .json (API responses), .txt (terminal output)
// NAMING: task-{N}-{scenario-slug}.{ext} or task-{N}-{scenario-slug}-error.{ext}
// SUBDIR: .sisyphus/evidence/final-qa/ for final verification wave
// LIFECYCLE: Created during execution, persists as audit trail.
// WRITER: Sisyphus-Junior (during QA scenarios)
// READERS: Atlas (verifies existence after each task), Momus (final audit)


// --- 4e. Atlas toggles checkboxes in plan after verified completion ---
Edit({
  file_path: ".sisyphus/plans/{plan-name}.md",
  old_string: "- [ ] 3. Add Login Endpoint",
  new_string: "- [x] 3. Add Login Endpoint",
})

// --- 4f. Atlas reads notepad before next delegation ---
const learnings = Read(".sisyphus/notepads/{plan-name}/learnings.md")
const issues = Read(".sisyphus/notepads/{plan-name}/issues.md")
// → includes as "Inherited Wisdom" in next task() prompt


// ============================================================
// INFRASTRUCTURE ARTIFACTS (system-level, not agent-generated)
// ============================================================

// --- Run Continuation Marker ---
// Written by: continuation system hooks (todo-continuation-enforcer, stop-continuation-guard)
// Path: .sisyphus/run-continuation/{sessionID}.json
// Schema:
{
  sessionID: "ses_abc123",
  updatedAt: "2026-03-11T14:30:00.000Z",
  sources: {
    todo?: { state: "idle" | "active" | "stopped", reason?: string, updatedAt: string },
    stop?: { state: "idle" | "active" | "stopped", reason?: string, updatedAt: string },
  }
}
// PURPOSE: Tells the CLI whether to auto-continue after agent turn ends.
//   "active" → CLI sends another message to resume work
//   "idle"/"stopped" → CLI stops, returns to user
// READERS: getContinuationState() in continuation-state.ts


// --- Ralph Loop State ---
// Written by: ralph-loop hook (iterative improvement loops)
// Path: .sisyphus/ralph-loop.local.md
// Schema: Markdown with YAML frontmatter
`---
active: true
iteration: 3
max_iterations: 100
completion_promise: "DONE"
started_at: "2026-03-11T14:00:00Z"
session_id: "ses_abc123"
ultrawork: false
strategy: "reset"
---
[The original prompt that started the loop]
`
// PURPOSE: Drives iterative loops where agent keeps working until it outputs
//   the completion_promise tag (default: "DONE"). Used for long-running
//   autonomous work sessions.
// READERS: ralph-loop hook checks state on every agent turn end
```

---

## COMPLETE ARTIFACT TABLE

```
ARTIFACT              PATH                                    FORMAT     SCHEMA
─────────────────────────────────────────────────────────────────────────────────
Boulder State         .sisyphus/boulder.json                  JSON       BoulderState interface
Plan                  .sisyphus/plans/{name}.md               Markdown   Convention (sections + checkboxes)
Draft                 .sisyphus/drafts/{slug}.md              Markdown   Convention (free-form sections)
Notepad: Learnings    .sisyphus/notepads/{name}/learnings.md  Markdown   Append-only, timestamped entries
Notepad: Issues       .sisyphus/notepads/{name}/issues.md     Markdown   Append-only, timestamped entries
Notepad: Decisions    .sisyphus/notepads/{name}/decisions.md  Markdown   Append-only, timestamped entries
Notepad: Problems     .sisyphus/notepads/{name}/problems.md   Markdown   Append-only, timestamped entries
Evidence              .sisyphus/evidence/task-{N}-{slug}.ext  Varies     .png / .json / .txt per QA tool
Evidence (final)      .sisyphus/evidence/final-qa/*           Varies     Same
Continuation Marker   .sisyphus/run-continuation/{sid}.json   JSON       ContinuationMarker interface
Ralph Loop State      .sisyphus/ralph-loop.local.md           MD+YAML   RalphLoopState interface
Rules                 .sisyphus/rules/*.md                    Markdown   Free-form (project config)
```

---

## DATA FLOW: WHO WRITES → WHO READS

```
                 WRITES                              READS
                 ──────                              ─────
Draft            Prometheus ──────────────────────→  Prometheus (own memory)
                                                     User (can review)

Plan             Prometheus ──────────────────────→  Momus (review loop)
                 Atlas (checkbox toggle only) ────→  Atlas (task extraction)
                                                     Sisyphus-Junior (READ-ONLY, via hook)
                                                     /start-work (progress check)

Boulder State    /start-work hook ────────────────→  Atlas (which plan to execute)
                 Atlas (append session_id) ───────→  continuation-state.ts (resume detection)
                                                     /start-work (resume vs new plan)

Notepads         Atlas (creates empty) ───────────→  Atlas (reads before each delegation)
                 Sisyphus-Junior (appends) ───────→  Next Sisyphus-Junior (via Atlas prompt)

Evidence         Sisyphus-Junior (QA artifacts) ──→  Atlas (verifies existence)
                                                     Momus (final audit, F1 task)

Continuation     todo-continuation-enforcer ──────→  CLI runner (auto-continue decision)
Marker           stop-continuation-guard ─────────→  getContinuationState()

Ralph Loop       ralph-loop hook ─────────────────→  ralph-loop hook (iteration control)
State                                                CLI runner (loop continuation)
```

---

## TYPE DEFINITIONS (from source)

```typescript
// src/features/boulder-state/types.ts
interface BoulderState {
  active_plan: string       // Absolute path to plan file
  started_at: string        // ISO 8601 timestamp
  session_ids: string[]     // All sessions that worked on this plan
  plan_name: string         // Derived from filename (basename without .md)
  agent?: string            // e.g., "atlas"
  worktree_path?: string    // Git worktree root (if using worktrees)
}

interface PlanProgress {
  total: number             // Total `- [ ]` + `- [x]` checkboxes
  completed: number         // Total `- [x]` checkboxes
  isComplete: boolean       // total === 0 || completed === total
}

// src/features/run-continuation-state/types.ts
type ContinuationMarkerSource = "todo" | "stop"
type ContinuationMarkerState = "idle" | "active" | "stopped"

interface ContinuationMarkerSourceEntry {
  state: ContinuationMarkerState
  reason?: string
  updatedAt: string         // ISO 8601
}

interface ContinuationMarker {
  sessionID: string
  updatedAt: string         // ISO 8601
  sources: Partial<Record<ContinuationMarkerSource, ContinuationMarkerSourceEntry>>
}

// src/hooks/ralph-loop/types.ts
interface RalphLoopState {
  active: boolean
  iteration: number
  max_iterations: number
  message_count_at_start?: number
  completion_promise: string   // default: "DONE"
  started_at: string           // ISO 8601
  prompt: string               // original loop prompt
  session_id?: string
  ultrawork?: boolean
  strategy?: "reset" | "continue"
}

// src/features/boulder-state/constants.ts
const BOULDER_DIR = ".sisyphus"
const BOULDER_FILE = "boulder.json"
const NOTEPAD_DIR = "notepads"
const NOTEPAD_BASE_PATH = ".sisyphus/notepads"
const PROMETHEUS_PLANS_DIR = ".sisyphus/plans"
```

---

## KEY PARSING FUNCTIONS

```typescript
// Plan progress — checkbox counting (src/features/boulder-state/storage.ts)
function getPlanProgress(planPath: string): PlanProgress {
  const content = readFileSync(planPath, "utf-8")
  const unchecked = content.match(/^\s*[-*]\s*\[\s*\]/gm) || []    // - [ ]
  const checked   = content.match(/^\s*[-*]\s*\[[xX]\]/gm) || []   // - [x] or - [X]
  const total = unchecked.length + checked.length
  return { total, completed: checked.length, isComplete: total === 0 || checked.length === total }
}

// Find plans — directory scan (src/features/boulder-state/storage.ts)
function findPrometheusPlans(directory: string): string[] {
  return readdirSync(join(directory, ".sisyphus/plans"))
    .filter(f => f.endsWith(".md"))
    .map(f => join(directory, ".sisyphus/plans", f))
    .sort((a, b) => statSync(b).mtimeMs - statSync(a).mtimeMs)  // newest first
}

// Boulder state — JSON read/write (src/features/boulder-state/storage.ts)
function readBoulderState(dir: string): BoulderState | null {
  return JSON.parse(readFileSync(join(dir, ".sisyphus/boulder.json"), "utf-8"))
}
function writeBoulderState(dir: string, state: BoulderState): boolean {
  writeFileSync(join(dir, ".sisyphus/boulder.json"), JSON.stringify(state, null, 2))
}

// Continuation state — composite check (src/cli/run/continuation-state.ts)
function getContinuationState(directory: string, sessionID: string): ContinuationState {
  return {
    hasActiveBoulder:     boulder exists && session matches && plan not complete,
    hasActiveRalphLoop:   ralph-loop state active && session matches,
    hasHookMarker:        .sisyphus/run-continuation/{sid}.json exists,
    hasTodoHookMarker:    marker has "todo" source,
    hasActiveHookMarker:  any source in "active" state,
    activeHookMarkerReason: reason string from active source,
  }
}
```
