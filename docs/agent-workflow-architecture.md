# Agent Workflow — Complete Function-Call Representation

The entire system is one pipeline: **Interview → Plan → Review → Execute**.
Every step below is an actual function/tool call with its signature.

---

## THE FULL PIPELINE AS PSEUDOCODE

```typescript
// ============================================================
// STAGE 1: INTERVIEW  (Agent: Prometheus)
// ============================================================

function prometheusInterview(userMessage: string): PlanFile {

  // 1a. Classify what the user wants
  const intent: Intent = classifyIntent(userMessage)
  //   → "Trivial" | "Refactoring" | "BuildFromScratch"
  //     | "MidSized" | "Collaborative" | "Architecture" | "Research"

  // 1b. Fire background research agents (parallel, non-blocking)
  //     Only for non-trivial intents.
  if (intent !== "Trivial") {
    const exploreTask = task({
      subagent_type: "explore",           // read-only codebase search
      run_in_background: true,            // async — don't wait
      load_skills: [],
      prompt: `[CONTEXT] what I'm building
               [GOAL] match existing conventions
               [DOWNSTREAM] use for plan structure
               [REQUEST] find similar implementations, return file paths + patterns`
    })
    // → returns: task_id (e.g. "bg_explore_001")
    // → side effects: none (read-only)

    const librarianTask = task({
      subagent_type: "librarian",         // external docs/OSS research
      run_in_background: true,
      load_skills: [],
      prompt: `[CONTEXT] implementing {technology}
               [GOAL] follow best practices
               [DOWNSTREAM] inform plan decisions
               [REQUEST] official docs, production examples, pitfalls`
    })
    // → returns: task_id (e.g. "bg_librarian_001")
    // → side effects: may clone repos to /tmp/ (ephemeral)

    // For Architecture intent, also consult Oracle (synchronous)
    if (intent === "Architecture") {
      const oracleResult = task({
        subagent_type: "oracle",          // strategic advisor, high-IQ reasoning
        run_in_background: false,         // BLOCKS until done
        load_skills: [],
        prompt: "Architecture consultation: {context}, analyze options and trade-offs"
      })
      // → returns: { bottomLine: string, actionPlan: Step[], effort: "Quick"|"Short"|"Medium"|"Large" }
      // → side effects: none (read-only)
    }
  }

  // 1c. Create draft file (external memory for multi-turn interview)
  Write({
    file_path: ".sisyphus/drafts/{topic-slug}.md",
    content: "# Draft: {Topic}\n## Requirements\n## Decisions\n## Research\n## Open Questions\n## Scope"
  })
  // → side effects: creates .sisyphus/drafts/{topic-slug}.md

  // 1d. Interview loop — ask questions, update draft each turn
  while (true) {

    // Collect background results when ready
    const exploreResults = background_output({ task_id: "bg_explore_001" })
    const librarianResults = background_output({ task_id: "bg_librarian_001" })

    // Ask user informed questions
    const answer = Question({
      questions: [{
        question: "Found pattern X in codebase. Should new code follow this?",
        header: "Pattern",
        options: [
          { label: "Follow existing", description: "Match current conventions" },
          { label: "Deviate", description: "Use different approach because..." }
        ]
      }]
    })
    // → returns: user's selection
    // → side effects: none

    // Update draft with new info
    Edit({
      file_path: ".sisyphus/drafts/{topic-slug}.md",
      old_string: "## Open Questions",
      new_string: "## Open Questions\n- {answered question}: {answer}\n"
    })
    // → side effects: modifies draft file

    // 1e. Clearance check — run after EVERY turn
    const clear = checkClearance({
      objectiveDefined: true,        // □ Core objective clear?
      scopeBoundaries: true,         // □ IN/OUT established?
      noAmbiguities: true,           // □ No critical unknowns?
      technicalApproach: true,       // □ Approach decided?
      testStrategy: true,            // □ TDD/tests-after/none?
      noBlockingQuestions: true       // □ Nothing outstanding?
    })
    // → returns: boolean
    // → side effects: none

    if (clear) break  // → proceed to Stage 2
    // else: ask the specific unclear question, loop again
  }

  return generatePlan(/* draft + research results */)
}


// ============================================================
// STAGE 2: PLAN GENERATION  (Agent: Prometheus)
// ============================================================

function generatePlan(interviewData: InterviewData): PlanFile {

  // 2a. Register visible progress tracker
  TodoWrite({ todos: [
    { content: "Consult Metis for gap analysis",          status: "pending", activeForm: "Consulting Metis..." },
    { content: "Generate plan file",                      status: "pending", activeForm: "Generating plan..." },
    { content: "Self-review: classify gaps",              status: "pending", activeForm: "Self-reviewing..." },
    { content: "Present summary to user",                 status: "pending", activeForm: "Presenting summary..." },
    { content: "Handle decisions needed (if any)",        status: "pending", activeForm: "Waiting for decisions..." },
    { content: "Offer high accuracy vs start work",       status: "pending", activeForm: "Offering choice..." },
    { content: "If high accuracy: Momus review loop",     status: "pending", activeForm: "Momus reviewing..." },
    { content: "Cleanup draft, guide to /start-work",     status: "pending", activeForm: "Cleaning up..." },
  ]})
  // → side effects: user sees 8-step checklist in UI

  // 2b. Consult Metis — mandatory gap analysis before plan
  const metisAnalysis = task({
    subagent_type: "metis",             // pre-planning consultant
    run_in_background: false,           // BLOCKS — need results before plan
    load_skills: [],
    prompt: `Review this planning session:
      **User's Goal**: ${interviewData.goal}
      **What We Discussed**: ${interviewData.keyPoints}
      **My Understanding**: ${interviewData.interpretation}
      **Research Findings**: ${interviewData.research}
      Identify: missed questions, guardrails, scope creep,
      unvalidated assumptions, missing acceptance criteria, edge cases`
  })
  // → returns: {
  //     intentType: "Build" | "Refactoring" | ...,
  //     risks: [{ risk: string, mitigation: string }],
  //     directives: { must: string[], mustNot: string[], patterns: string[] },
  //     questionsForUser: string[]
  //   }
  // → side effects: none (read-only)

  // 2c. Write plan skeleton (one Write call)
  Write({
    file_path: ".sisyphus/plans/{plan-name}.md",
    content: `# {Plan Title}
## TL;DR
## Context
## Work Objectives
## Verification Strategy
## Execution Strategy
---
## TODOs
---
## Final Verification Wave
## Success Criteria`
  })
  // → side effects: creates .sisyphus/plans/{plan-name}.md

  // 2d. Append tasks in batches of 2-4 (avoids output token limits)
  for (const batch of taskBatches) {
    Edit({
      file_path: ".sisyphus/plans/{plan-name}.md",
      old_string: "---\n\n## Final Verification Wave",
      new_string: `- [ ] ${batch[0].id}. ${batch[0].title}
  **What to do**: ${batch[0].steps}
  **Must NOT do**: ${batch[0].exclusions}
  **Agent Profile**: ${batch[0].category} + [${batch[0].skills}]
  **Parallelization**: Wave ${batch[0].wave} | Blocked By: ${batch[0].deps}
  **References**: ${batch[0].refs}
  **Acceptance Criteria**: ${batch[0].criteria}
  **QA Scenarios**: ${batch[0].qa}

- [ ] ${batch[1].id}. ${batch[1].title}
  ...

---

## Final Verification Wave`
    })
    // → side effects: appends tasks to plan file
  }

  // 2e. Verify plan completeness
  const planContent = Read({ file_path: ".sisyphus/plans/{plan-name}.md" })
  // → returns: full plan text
  // → side effects: none

  // 2f. Self-review — classify gaps
  const gaps = selfReview(planContent)
  //   CRITICAL gaps  → mark [DECISION NEEDED] in plan, ask user
  //   MINOR gaps     → fix silently in plan
  //   AMBIGUOUS gaps → apply default, disclose in summary

  // 2g. Present summary to user (text output)
  //   "Plan Generated: {name}"
  //   "Key Decisions: ..., Scope: IN/OUT, Guardrails: ..., Auto-Resolved: ..."

  // 2h. Offer choice
  const choice = Question({
    questions: [{
      question: "Plan is ready. How would you like to proceed?",
      header: "Next Step",
      options: [
        { label: "Start Work",           description: "Execute with /start-work" },
        { label: "High Accuracy Review", description: "Momus verifies every detail first" }
      ]
    }]
  })
  // → returns: "Start Work" | "High Accuracy Review"

  if (choice === "High Accuracy Review") {
    momusReviewLoop(".sisyphus/plans/{plan-name}.md")
  }

  // 2i. Cleanup
  Bash({ command: "rm .sisyphus/drafts/{topic-slug}.md" })
  // → side effects: deletes draft

  return ".sisyphus/plans/{plan-name}.md"
}


// ============================================================
// STAGE 3: MOMUS REVIEW LOOP  (Agent: Prometheus invokes Momus)
// ============================================================

function momusReviewLoop(planPath: string): void {

  while (true) {
    const verdict = task({
      subagent_type: "momus",             // plan reviewer — blocker-finder
      run_in_background: false,           // BLOCKS
      load_skills: [],
      prompt: planPath                    // just the file path, nothing else
    })
    // Momus internally does:
    //   1. Read(planPath)
    //   2. For each task: verify file refs exist via Read/Glob
    //   3. Check executability: can a dev start each task?
    //   4. Return "[OKAY]" or "[REJECT]" + max 3 blocking issues
    //
    // → returns: { verdict: "OKAY" | "REJECT", issues?: string[] }
    // → side effects: none (read-only)

    if (verdict.verdict === "OKAY") break

    // Fix every issue Momus raised
    for (const issue of verdict.issues) {
      Edit({
        file_path: planPath,
        old_string: /* problematic section */,
        new_string: /* fixed section */
      })
    }
    // Loop — resubmit to Momus. No max retries.
  }
}


// ============================================================
// STAGE 4: EXECUTION  (User runs /start-work → Atlas takes over)
// ============================================================

// --- 4a. /start-work hook ---

function startWork(planName?: string): void {

  // Find available plans
  const plans: string[] = findPrometheusPlans(directory)
  // → scans .sisyphus/plans/*.md, returns paths sorted by mtime
  // → side effects: none

  // Check if there's already active work
  const boulder: BoulderState | null = readBoulderState(directory)
  // → reads .sisyphus/boulder.json
  // → returns: { active_plan, started_at, session_ids, plan_name, worktree_path } | null

  let selectedPlan: string
  if (boulder && !getPlanProgress(boulder.active_plan).isComplete) {
    // Resume existing work
    selectedPlan = boulder.active_plan
    appendSessionId(directory, currentSessionId)
    // → side effects: updates session_ids in boulder.json
  } else if (plans.length === 1) {
    selectedPlan = plans[0]  // auto-select
  } else {
    selectedPlan = askUserToSelect(plans)
  }

  // Create/update boulder state
  const state = createBoulderState(selectedPlan, currentSessionId, "atlas", worktreePath)
  // → returns: { active_plan, started_at, session_ids: [sid], plan_name, agent: "atlas", worktree_path }
  writeBoulderState(directory, state)
  // → side effects: writes .sisyphus/boulder.json

  // Hand off to Atlas
  atlasOrchestrate(selectedPlan)
}


// --- 4b. Atlas orchestration ---

function atlasOrchestrate(planPath: string): void {

  // Read and parse the plan
  const plan = Read({ file_path: planPath })
  // → returns: full plan markdown
  const tasks = parseTasks(plan)
  //   extracts: [ { id, title, wave, deps, done: boolean, ... } ]
  const waves = buildParallelizationMap(tasks)
  //   groups independent tasks into waves for parallel execution

  // Register tracking
  TodoWrite({ todos: [
    { content: "Complete ALL tasks in work plan", status: "in_progress",
      activeForm: "Orchestrating plan execution..." }
  ]})

  // Initialize notepad (accumulated knowledge between tasks)
  Bash({ command: "mkdir -p .sisyphus/notepads/{plan-name}" })
  // → side effects: creates learnings.md, decisions.md, issues.md, problems.md

  // Execute wave by wave
  for (const wave of waves) {

    // Read notepad before each wave
    const learnings = Read({ file_path: ".sisyphus/notepads/{plan-name}/learnings.md" })
    const issues    = Read({ file_path: ".sisyphus/notepads/{plan-name}/issues.md" })

    // --- DELEGATE: parallel tasks in ONE message ---
    const results = wave.tasks.map(t => task({
      // Option A: category-based (spawns Sisyphus-Junior)
      category: t.category,             // e.g. "quick", "deep", "visual-engineering"
      load_skills: t.skills,            // e.g. ["playwright", "frontend-ui-ux"]
      run_in_background: false,         // ALWAYS sync for execution
      prompt: `
## 1. TASK
${t.exactCheckboxText}

## 2. EXPECTED OUTCOME
- [ ] Files: ${t.filePaths}
- [ ] Behavior: ${t.expectedBehavior}
- [ ] Verify: \`${t.verifyCommand}\` passes

## 3. REQUIRED TOOLS
${t.tools.map(tool => `- ${tool}`).join('\n')}

## 4. MUST DO
- Follow pattern in ${t.referenceFiles}
- Append findings to .sisyphus/notepads/{plan-name}/learnings.md

## 5. MUST NOT DO
- Do NOT modify files outside ${t.scope}
- Do NOT add dependencies
- Do NOT skip verification

## 6. CONTEXT
### Inherited Wisdom
${learnings}
### Known Issues
${issues}
### Dependencies
${t.dependsOn}`
    }))
    // → returns per task: { result: string, session_id: string }
    // → side effects: files created/modified by sub-agent

    // --- OR Option B: specialized agent ---
    // task({ subagent_type: "explore", ... })
    // task({ subagent_type: "oracle", ... })

    // --- VERIFY: mandatory after every delegation ---
    for (const [i, result] of results.entries()) {
      const t = wave.tasks[i]

      // A. Automated checks
      Bash({ command: "lsp_diagnostics ." })         // → must be ZERO errors
      Bash({ command: "bun run build" })              // → must exit 0
      Bash({ command: "bun test" })                   // → must all pass

      // B. Manual code review (read every changed file)
      for (const file of result.changedFiles) {
        const content = Read({ file_path: file })
        // Verify: logic correct? no stubs/TODOs? matches codebase patterns? imports right?
      }

      // C. Hands-on QA (if applicable)
      // Frontend → task({ load_skills: ["playwright"], ... })
      // CLI/TUI  → interactive_bash(...)
      // API      → Bash({ command: "curl ..." })

      // D. Check plan progress
      const planNow = Read({ file_path: planPath })
      // Count remaining `- [ ]` vs `- [x]` checkboxes

      // --- RETRY on failure ---
      if (!verified) {
        // Resume SAME session (preserves full context)
        const fix = task({
          session_id: result.session_id,   // from original task output
          load_skills: t.skills,
          run_in_background: false,
          prompt: `FAILED: ${actualError}. Fix by: ${specificInstruction}`
        })
        // → returns: fixed result (sub-agent retains all prior context)
        // → side effects: files re-modified
        // Max 3 retries. After 3: document in notepad, move on.
      }

      // Mark task complete in plan
      Edit({
        file_path: planPath,
        old_string: `- [ ] ${t.id}. ${t.title}`,
        new_string: `- [x] ${t.id}. ${t.title}`
      })
    }
  }

  // Final report
  // "ORCHESTRATION COMPLETE — N/N tasks done, files modified: [...], wisdom: [...]"
}
```

---

## WHAT EACH AGENT DOES INTERNALLY

```typescript
// ─── EXPLORE ────────────────────────────────────────────────
// Invoked by: Prometheus, Metis, Atlas, Sisyphus
// Mode: subagent (read-only)
// Restrictions: no write, no edit, no task, no call_omo_agent
function explore(prompt: string): SearchResults {
  // 1. Parse intent: literal request → actual need → success criteria
  // 2. Fire 3+ tools in parallel:
  grep({ pattern: "...", path: "src/" })
  glob({ pattern: "**/*.ts" })
  lsp_find_references({ symbol: "..." })
  ast_grep_search({ pattern: "..." })
  // 3. Return structured results:
  //    { files: [{ path, why }], answer: string, nextSteps: string }
}

// ─── LIBRARIAN ──────────────────────────────────────────────
// Invoked by: Prometheus, Metis, Atlas, Sisyphus
// Mode: subagent (read-only)
// Restrictions: no write, no edit, no task, no call_omo_agent
function librarian(prompt: string): ResearchResults {
  // 1. Classify: TYPE A (conceptual) | B (implementation) | C (context) | D (comprehensive)
  // 2. Doc discovery phase (sequential):
  websearch("library-name official docs")
  webfetch(docsUrl + "/sitemap.xml")
  webfetch(specificDocPage)
  // 3. Main research phase (parallel):
  context7_resolve_library_id("library")
  context7_query_docs({ libraryId, query: "topic" })
  grep_app_searchGitHub({ query: "pattern", language: ["TypeScript"] })
  Bash({ command: `gh repo clone owner/repo /tmp/repo -- --depth 1` })
  // 4. Return: claims with GitHub permalinks, code snippets, explanations
}

// ─── ORACLE ─────────────────────────────────────────────────
// Invoked by: Prometheus (architecture), Atlas (after failures), Sisyphus (hard bugs)
// Mode: subagent (read-only)
// Restrictions: no write, no edit, no task
// Config: thinking.budgetTokens = 32000 (Claude) | reasoningEffort = "medium" (GPT)
function oracle(prompt: string): Consultation {
  // Extended thinking → deep analysis
  // Returns:
  //   bottomLine: "2-3 sentences"
  //   actionPlan: Step[]  (max 7 steps, each ≤2 sentences)
  //   effort: "Quick" | "Short" | "Medium" | "Large"
  //   watchOutFor?: string[]  (max 3 bullets)
}

// ─── METIS ──────────────────────────────────────────────────
// Invoked by: Prometheus (mandatory before plan generation)
// Mode: subagent (read-only)
// Restrictions: no write, no edit
// Config: temperature = 0.3, thinking.budgetTokens = 32000
function metis(prompt: string): GapAnalysis {
  // 1. Classify intent (same taxonomy as Prometheus)
  // 2. May fire explore/librarian for BUILD/RESEARCH intents
  // 3. Analyze for: missed questions, guardrails, scope creep, AI-slop
  // Returns:
  //   intentType: string
  //   risks: [{ risk, mitigation }]
  //   questionsForUser: string[]
  //   directives: { must: string[], mustNot: string[], patterns: string[] }
}

// ─── MOMUS ──────────────────────────────────────────────────
// Invoked by: Prometheus (high accuracy mode only)
// Mode: subagent (read-only)
// Restrictions: no write, no edit, no task
// Config: temperature = 0.1, thinking.budgetTokens = 32000
function momus(planPath: string): ReviewVerdict {
  // 1. Read(planPath)
  // 2. For each task: Read/Glob to verify file references exist
  // 3. Check: can a developer START each task?
  // 4. Approval bias: when in doubt, APPROVE
  // Returns:
  //   verdict: "OKAY" | "REJECT"
  //   issues?: string[]  (max 3 blocking issues if REJECT)
}

// ─── SISYPHUS ───────────────────────────────────────────────
// The main agent for direct work (non-planning mode)
// Mode: all (primary + subagent)
// Permissions: full (write, edit, task, bash, webfetch, question)
function sisyphus(userMessage: string): void {
  // Phase 0: Intent gate (every message)
  //   Verbalize intent → classify → check for delegation triggers
  // Phase 1: Codebase assessment (open-ended tasks)
  //   Check configs, sample files, classify discipline level
  // Phase 2A: Exploration (fire explore/librarian in background)
  // Phase 2B: Implementation (delegate via task() or do directly)
  //   Uses: task({ category: "...", load_skills: [...], prompt: "6-section" })
  //   Verifies: lsp_diagnostics, build, tests
  // Phase 3: Completion (all todos done, diagnostics clean)
}

// ─── ATLAS ──────────────────────────────────────────────────
// Plan execution orchestrator
// Mode: all
// Permissions: task (yes), write/edit (delegates, never does itself)
function atlas(planPath: string): void {
  // See atlasOrchestrate() above — that IS Atlas's full behavior.
  // Key rules:
  //   - Never writes code itself (always delegates)
  //   - 6-section prompt mandatory for every delegation
  //   - Verify after EVERY delegation (automated + manual + hands-on)
  //   - Resume failed tasks via session_id (max 3 retries)
  //   - Notepad read before every delegation, write after every completion
}

// ─── SISYPHUS-JUNIOR ────────────────────────────────────────
// Spawned by task(category="...") — the actual worker
// Mode: all
// Restrictions: no task (prevents infinite recursion)
// Gets: skills injected, category-specific model, category prompt append
function sisyphusJunior(prompt: string): void {
  // Receives 6-section prompt from Atlas or Sisyphus
  // Has full read/write/edit/bash access
  // Executes the actual code changes
  // Cannot spawn further sub-agents (task tool denied)
}
```

---

## FILE ARTIFACTS PRODUCED

```
.sisyphus/
├── drafts/
│   └── {topic-slug}.md          # Created: Stage 1 (interview)
│                                 # Deleted: Stage 2 (after plan generated)
├── plans/
│   └── {plan-name}.md           # Created: Stage 2 (plan generation)
│                                 # Read by: Momus (Stage 3), Atlas (Stage 4)
│                                 # Updated: Atlas marks [x] as tasks complete
├── boulder.json                  # Created: Stage 4 (/start-work)
│                                 # Contains: active_plan, session_ids, worktree_path
├── notepads/
│   └── {plan-name}/
│       ├── learnings.md          # Created: Stage 4 (Atlas init)
│       ├── decisions.md          # Append-only during execution
│       ├── issues.md             # Shared across all task delegations
│       └── problems.md
└── evidence/
    └── task-{N}-{slug}.{ext}    # Screenshots, terminal output, API responses
```

---

## INVOCATION SUMMARY (who calls whom)

```
Prometheus
 ├── task(explore)        ×N  background   "find codebase patterns"
 ├── task(librarian)      ×N  background   "find external docs"
 ├── task(oracle)         ×1  sync         "architecture consultation"  (Architecture intent only)
 ├── task(metis)          ×1  sync         "gap analysis before plan"   (MANDATORY)
 └── task(momus)          ×N  sync/loop    "review plan until OKAY"     (high accuracy only)

Atlas
 ├── task(category=...)   ×N  sync         "execute plan task"          (spawns Sisyphus-Junior)
 │    └── session resume  ×3  sync         "fix failed task"            (same session_id)
 ├── task(explore)        ×N  background   "research for delegation"
 └── task(oracle)         ×1  sync         "consult after 2+ failures"

Sisyphus (direct work, no planning)
 ├── task(explore)        ×N  background   "contextual grep"
 ├── task(librarian)      ×N  background   "external research"
 ├── task(oracle)         ×1  sync/bg      "hard debugging / architecture"
 └── task(category=...)   ×N  sync         "delegate implementation"
```
