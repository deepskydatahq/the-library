# Execute Mission

Full autopilot from mission alignment to PRs. Breaks down the mission into epics and stories, triages each story, then launches parallel sub-agents to implement each epic in an isolated worktree. Independent evaluator agents verify each epic before PR creation. One PR per epic.

## Arguments

- Mission ID (required): `/execute-mission M014`

## Instructions

### Phase 0: NFTOS Setup

Check if `nftos` is available:

```bash
which nftos 2>/dev/null
```

If available, NFTOS posting is enabled for this mission. You'll post at key milestones throughout execution. If not available, skip all `nftos` commands silently.

### Phase 1: Breakdown + Triage

#### 1.1 Read the Mission

Read the mission TOML file:

```bash
ls product/missions/{mission_id}-*.toml
```

Read the file. Extract: `id`, `title`, `outcome.description`, `scope.in_scope`, `scope.out_of_scope`, `context.relevant_paths`, `testing.criteria`.

Also read `CLAUDE.md` for project conventions.

#### 1.2 Mission → Epics

**Check for existing epics first:**

```bash
ls product/epics/{mission_id}-E*.toml 2>/dev/null
```

**If epics already exist:** Read them and proceed to step 1.3.

**If no epics exist:** Create them.

Review the files listed in `relevant_paths` to understand current code state. Identify 3-6 epic boundaries based on:
- Backend vs frontend concerns
- Data layer vs presentation layer
- Core functionality vs supporting features
- Independent workstreams

For each epic, create `product/epics/{mission_id}-E{NNN}-{slug}.toml`:

```toml
id = "{mission_id}-E{NNN}"
parent = "{mission_id}"
title = "Epic title"
status = "active"
created = {today}
depends_on = []  # Other epic IDs if sequential

[outcome]
description = """What this epic delivers."""

[job_story]
description = """When..., I want..., so that..."""

[testing]
approach = "agent-judgment"
criteria = ["Verifiable outcome 1", "Verifiable outcome 2"]
validator_context = ["relevant/paths/"]

[context]
relevant_paths = ["scoped/paths/"]
dependencies = []

[notes]
considerations = """Context for story breakdown."""

estimated_stories = 0
```

Verify: all mission outcomes covered, no gaps, no overlaps, dependencies noted.

#### 1.2.1 NFTOS: Mission Kickoff

If nftos is available, post the mission kickoff:

```bash
nftos "Breaking down mission {mission_id} into {N} epics — {mission_title}" --type status
```

#### 1.3 Epics → Stories

For each epic:

**Check for existing stories first:**

```bash
ls product/stories/{epic_id}-S*.toml 2>/dev/null
```

**If stories already exist:** Read them and proceed to step 1.4.

**If no stories exist:** Create them.

Read the epic's `relevant_paths` to understand what exists. Break the epic into implementation units. Each story should:
- Take one implementation session
- Have a single clear purpose
- Be testable in isolation
- Result in working code

For each story, create `product/stories/{epic_id}-S{NNN}-{slug}.toml`:

```toml
id = "{epic_id}-S{NNN}"
parent = "{epic_id}"
title = "Story title"
status = "ready"
created = {today}

[outcome]
description = """One sentence: what this implements."""

[acceptance_criteria]
executable = true

[[acceptance_criteria.criteria]]
test = "unit"
description = "Specific testable behavior"

[context]
relevant_paths = ["src/specific/file.ts"]
input_fixtures = []
depends_on = []  # Other story IDs

[handoff]
implementation_hints = """Approach hints if not self-explanatory."""
reference_files = []
```

Write acceptance criteria that are specific and executable:
- Good: `calculateMRR([]) returns 0`
- Bad: `MRR calculation works correctly`

#### 1.4 LLM Triage

For each story, evaluate and assign a triage label. Read the story TOML and assess:

**READY** (skip brainstorm + plan) — assign when ALL true:
- Acceptance criteria reference specific file paths
- Changes are mechanical or well-defined (add field, wire function, update test)
- Scope is a single file or small set of files
- No design decisions needed
- Implementation approach is obvious from the story context

**PLAN** (skip brainstorm) — assign when:
- Clear goal and acceptance criteria
- Needs codebase exploration to identify exact files
- Implementation approach is obvious but details need working out
- May touch multiple files but the pattern is clear

**BRAINSTORM** (full pipeline) — assign when ANY true:
- Multiple valid approaches exist
- Architectural decisions needed
- Scope is unclear or cross-cutting
- New patterns or abstractions required
- Story notes contain open questions

Output a triage table:

```
## Triage Results

| Story | Title | Label | Justification |
|-------|-------|-------|---------------|
| M015-E001-S001 | Add parser | ready | Clear ACs, single file, mechanical |
| M015-E001-S002 | Wire into pipeline | plan | Needs exploration to identify integration points |
| M015-E002-S001 | Design metrics | brainstorm | Multiple approaches, needs design decision |
```

#### 1.5 Create Beads Tasks

For each story, create a Beads task:

```bash
bd create "{story_id}: {story_title}" --labels {triage_label} -d "{context}" --silent
```

The task description should include:
- Mission context (title + outcome, 2 sentences)
- Epic context (title + outcome, 1 sentence)
- Story outcome
- Acceptance criteria (copied from TOML)
- Relevant paths
- Implementation hints (if any)

Capture the task ID from each `bd create` output.

Register story dependencies as Beads task dependencies:

```bash
bd dep add {dependent_task_id} {blocking_task_id}
```

Record the mapping: `story_id → beads_task_id` — you'll need this for the sub-agent prompts.

#### 1.6 Build Dependency Graph

From the epic TOMLs, identify epic-level dependencies (`depends_on` field).

Classify epics:
- **Independent:** no `depends_on`, or all dependencies are outside this mission (already complete)
- **Dependent:** has `depends_on` referencing other epics in this mission

---

### Phase 2: Epic Sub-Agents

Launch one sub-agent per epic using the Agent tool with `isolation: "worktree"`.

**Dependency-aware launch algorithm:**

1. Start with all epics in a `remaining` set
2. Find `launchable` = epics in `remaining` whose `depends_on` are all completed
3. Launch all `launchable` epics in parallel (multiple Agent tool calls in ONE message)
4. When results return:
   - **Success:** move to `completed`, record branch name and worktree path
   - **Failed:** move to `failed`, also mark any epics that depend on it as `cascade-failed`
5. Repeat from step 2 until `remaining` is empty
6. If `launchable` is empty but `remaining` is not — deadlock (all remaining depend on failures). Report and stop.

**For each epic, launch an Agent with this prompt:**

Use `subagent_type: "general-purpose"` and `isolation: "worktree"`.

The prompt template (fill in the `{placeholders}` for each epic):

---

```
You are implementing epic {epic_id} for this project.

## Project Context

{Paste the relevant section from CLAUDE.md — build commands, test commands, project structure, conventions.}

## Mission: {mission_title}

{mission_outcome — 2-3 sentences}

## Epic: {epic_id} — {epic_title}

{epic_outcome_description}

## Stories to Implement (in order)

{For each story in this epic, numbered:}

### Story {N}: {story_id} — {story_title}
**Triage:** {ready|plan|brainstorm}
**Outcome:** {story_outcome}
**Beads Task:** {beads_task_id}
**Acceptance Criteria:**
{list each criterion with test type}
**Relevant Paths:** {paths}
**Implementation Hints:** {hints or "none"}
**Depends On:** {story dependencies or "none"}

{...repeat for each story...}

## Your Process

For each story, in order:

1. **Claim the task:**
   bd update {beads_task_id} --status in_progress

2. **Handle based on triage label:**

   If BRAINSTORM:
   - Read the relevant code paths
   - Consider 2-3 approaches, pick the simplest one
   - Note your design decision in a brief comment in the code or commit message
   - Then proceed to plan and implement

   If PLAN:
   - Read the relevant code paths
   - Identify the specific files to create or modify
   - Then implement

   If READY:
   - Go straight to implementation

3. **Implement:**
   - Write the code changes
   - Write tests for each acceptance criterion
   - Run tests
   - Fix any test failures
   - Commit with a descriptive message: feat: {description}

4. **Close the task:**
   bd close {beads_task_id}

5. Move to the next story.

## After All Stories

1. Run the full test suite
2. Run the build (if applicable)
3. Fix any issues
4. Push the branch: git push -u origin HEAD
5. Do NOT create the PR — that happens after evaluation.

## Rules

- Do NOT ask questions. Make autonomous decisions — pick the simpler option when unsure.
- If you hit a blocker on one story, document it as a comment and move to the next.
- Commit after each story — incremental commits, not one big commit.
- All tests MUST pass before pushing.
- If tests fail after all attempts, push anyway but note failures in the output.

## Output

When complete, output exactly this block:

EPIC_RESULT: {epic_id}
STATUS: success | partial | failed
BRANCH: {branch name}
WORKTREE: {worktree path}
STORIES_COMPLETED: {comma-separated story_ids}
STORIES_FAILED: {comma-separated story_ids with brief reasons, or "none"}
TASKS_CLOSED: {comma-separated beads_task_ids}
```

---

### Phase 2b: Evaluator Agents

After each epic sub-agent completes successfully, launch an evaluator agent in the **same worktree**. The evaluator is a separate agent with a fresh context — it assesses the output without seeing the generator's reasoning.

**Why a separate agent?** Self-evaluation is biased. Agents tend to praise their own work. An independent evaluator with only the contract and the code provides honest assessment.

For each completed epic, launch an evaluator Agent with `subagent_type: "general-purpose"` (no worktree isolation — use the epic's existing worktree by setting the working directory).

**Evaluator prompt template:**

```
You are an independent evaluator for epic {epic_id}.

Your job is to verify whether the implementation meets the agreed testing criteria. You are NOT the author of this code — you are a skeptical reviewer. Do not assume anything works; verify it.

## Project Context

{Paste the relevant section from CLAUDE.md — build commands, test commands.}
- Working directory: {worktree_path}

## Testing Contract

These criteria were agreed BEFORE implementation. Every criterion must pass.

### Mission Criteria (from {mission_id})

{Paste the full testing.criteria array from the mission TOML}

### Epic Criteria (from {epic_id})

{Paste the full testing.criteria array from the epic TOML}

## Your Process

1. **Run tests:**
   {project test command}
   Record: pass count, fail count, any failures.

2. **Verify each criterion independently:**
   For each criterion in the contract:
   - Read the relevant code
   - Check whether the criterion is actually met (not just whether a test exists)
   - Look for edge cases the author might have missed
   - Score: PASS or FAIL
   - Write a specific finding (what you checked and what you found)

3. **Check for regressions:**
   - git diff main...HEAD --stat to see all changed files
   - Verify no unrelated files were modified
   - Check that existing functionality wasn't broken

## Rules

- Be skeptical. "Tests pass" does not mean criteria are met — tests might be weak.
- Check the actual behavior, not just that code exists.
- FAIL means the criterion is not met. Be specific about what's wrong.
- PASS means you verified it works. Say what you checked.
- Do NOT fix anything. You are an evaluator, not a fixer.
- Do NOT add new requirements beyond the contract.

## Output

Output exactly this block:

EVAL_RESULT: {epic_id}
TESTS: {pass_count} passed, {fail_count} failed
VERDICT: pass | fail

CRITERIA:
- [PASS] "{criterion text}" — {what you verified}
- [FAIL] "{criterion text}" — {what's wrong and where}
{...one line per criterion...}

FINDINGS:
{Any additional observations — regressions, code quality concerns, edge cases.
Or "None" if everything looks good.}
```

**Processing evaluator results:**

Parse the evaluator output. Extract the verdict and any FAIL criteria.

**If VERDICT is "pass":** Post to NFTOS (if available) and create the PR:

```bash
nftos "Epic {epic_id} passed evaluation — all criteria met, creating PR" --type milestone
```

```bash
cd {worktree_path}
gh pr create --base main --title "feat({epic_id}): {epic_title}" --body "$(cat <<'EOF'
## Summary
{epic_outcome}

## Stories Implemented
{list of story_ids and titles}

## Evaluation
All {N} criteria passed.

---
Automated via /execute-mission
EOF
)"
```

**If VERDICT is "fail":** Launch a rework agent. Use the Agent tool with `subagent_type: "general-purpose"` (no worktree isolation — work in the same worktree).

**Rework agent prompt:**

```
You are fixing issues found by an independent evaluator for epic {epic_id}.

Working directory: {worktree_path}

## Failed Criteria

{Paste each FAIL line from the evaluator output}

## Additional Findings

{Paste the FINDINGS section}

## Instructions

1. For each failed criterion, read the finding and fix the issue.
2. Run tests
3. Commit fixes: git commit -m "fix: address evaluator findings for {epic_id}"
4. Push: git push

Do NOT ask questions. Fix the specific issues identified.

Output: REWORK_DONE: {epic_id}
```

After the rework agent completes, run the evaluator again (same prompt). Max 2 rework rounds total.

After 2 rounds, if still failing: create the PR anyway with unresolved findings in the body:

```bash
gh pr create --base main --title "feat({epic_id}): {epic_title}" --body "$(cat <<'EOF'
## Summary
{epic_outcome}

## Stories Implemented
{list}

## Evaluation — Unresolved Findings
{paste FAIL criteria that remain after 2 rework rounds}

**Human review required for unresolved items.**

---
Automated via /execute-mission
EOF
)"
```

---

### Phase 3: PR Feedback

After all epic sub-agents complete, collect the PR URLs from their results.

Wait 60 seconds for automated reviewers (Graptile, CodeRabbit) to process the PRs:

```bash
sleep 60
```

For each PR that was created successfully, launch a sub-agent to fix feedback:

Use the Agent tool with `subagent_type: "general-purpose"` (no worktree isolation needed — it works on the existing PR branch).

Prompt: `"Run /fix-pr-feedback {pr_number} — check out the PR branch, read review comments, fix actionable issues, push fixes. Max 2 rounds."`

Launch all PR feedback agents in parallel (multiple Agent calls in one message, use `run_in_background: true`).

---

### Phase 4: TOML Commit

Commit the mission, epic, and story TOML files on a separate branch and create a PR:

```bash
git checkout -b bd/{mission_id}-tomls main
git add product/missions/{mission_id}-*.toml product/epics/{mission_id}-*.toml product/stories/{mission_id}-*.toml
git commit -m "bd: add {mission_id} mission, epic, and story TOMLs"
git push -u origin HEAD
gh pr create --base main --title "bd: {mission_id} mission, epic, and story TOMLs" --body "Product layer TOMLs for {mission_id}."
```

Do NOT merge — leave for human review alongside the epic PRs.

---

### Phase 5: Report

#### 5.0 NFTOS: Mission Complete

If nftos is available, post the mission result:

**If all epics succeeded:**
```bash
nftos "Mission {mission_id} complete — {N} epics, {N} stories, {N} PRs created. {mission_title}" --type milestone
```

**If some epics failed:**
```bash
nftos "Mission {mission_id} partial — {completed}/{total} epics shipped, {failed} need attention. {mission_title}" --type error
```

#### 5.1 Report

Collect results from all agents and output:

```
## Mission Execution Report: {mission_id} — {mission_title}

### Epics

| Epic | Status | Eval | PR | Stories |
|------|--------|------|-----|---------|
| E001 | success | pass (1st) | #{pr} | 3/3 completed |
| E002 | partial | pass (2nd — 1 rework) | #{pr} | 2/3 completed |
| E003 | cascade-failed | — | — | blocked by E001 |

### Evaluation Summary
- E001: 5/5 criteria passed on first evaluation
- E002: 3/5 passed first eval → rework → 5/5 passed second eval
- E003: not evaluated (blocked)

### PRs for Review
- #{pr}: feat(E001): {title} — {url}
- #{pr}: feat(E002): {title} — {url}
- #{pr}: bd: {mission_id} TOMLs — {url}

**ACTION REQUIRED: Please review and merge these PRs.**
For sequential epics, merge in order (E001 first, then E002, etc.).

### Tasks Closed
{list of beads task IDs}

### Failures Requiring Attention
{list of failed stories with reasons, or "None — all stories completed successfully."}

### Unresolved Evaluation Findings
{list of criteria that failed after 2 rework rounds, or "None — all criteria met."}

### PR Feedback
- #{pr}: {N} issues fixed, {N} unresolved
- #{pr}: No actionable feedback

### Summary
- Epics: {completed}/{total} succeeded
- Stories: {completed}/{total} implemented
- Evaluation: {pass_count}/{total} passed, {rework_count} required rework
- PRs: {count} created (awaiting review)
- Tasks: {count} closed
```
