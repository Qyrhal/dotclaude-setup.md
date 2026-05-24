# setup.md

<!--
  PURPOSE: Agent-executable runbook for bootstrapping the .claude directory.
  Scope is strictly limited to .claude/ and the root CLAUDE.md file.
  Nothing outside that boundary is read, written, or inferred.
  Ref: https://code.claude.com/docs/en/claude-directory
-->

---

## Scope

Touches **only**:
```
CLAUDE.md               ← project root
.claude/
  settings.json
  settings.local.json   ← gitignored, created if user requests local overrides
  CLAUDE.local.md       ← gitignored, personal notes
  commands/             ← slash-command stubs
  rules/
    maintain.md         ← self-maintenance contract, loaded every session
    coding.md           ← behavioural guardrails for all coding tasks
  memory/               ← structured memory files
  kanban.md             ← live issue/fix tracker (session-persistent)
  improvements.md       ← forward-looking feature/quality backlog
```

**Does not touch source code, CI config, dependencies, or any file outside .claude/.**

---

## Phase 1 — Scan

Perform the following reads silently before asking the user anything.

| Check | What to look for |
|---|---|
| Language / runtime | `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `*.csproj`, etc. |
| Test runner | test scripts in package.json, pytest config, `Makefile` targets |
| Build commands | build scripts, Makefile, Dockerfile |
| Lint / format | `.eslintrc`, `.prettierrc`, `ruff.toml`, `.golangci.yml`, etc. |
| Secrets exposure | `.env` committed, hardcoded tokens, API keys in source |
| Dependency hygiene | lockfile present, `node_modules` committed, known CVE patterns |
| Structure clarity | deep nesting, missing README, no entrypoint obvious |
| Permissions / auth | middleware patterns, auth guards, RBAC hints |
| Logging | structured vs. ad-hoc, sensitive data in logs |
| Error handling | bare `catch {}`, unhandled promise rejections, panic without recovery |

Log findings internally. Do not output them yet.

---

## Phase 2 — Questions

Ask **only what cannot be inferred** from the scan. Batch into a single prompt; do not ask one at a time.

```
I've scanned the repo. A few things I couldn't determine:

1. [Only if runtime is ambiguous] What language/runtime should I assume as primary?
2. [Only if no test runner found] What command runs the test suite?
3. [Only if no build script found] What is the build command?
4. What is the primary purpose of this project? (one sentence)
5. Are there security or compliance requirements I should know about?
   (e.g. HIPAA, PCI-DSS, GDPR, SOC 2, internal policy)
6. Any team conventions not obvious from code? (style, PR rules, etc.)
7. Should I create settings.local.json for personal overrides? (yes/no)

Skip any question that is already answered by the repo.
```

---

## Phase 3 — Generate Files

### CLAUDE.md (project root)

```markdown
# <project-name>

<!--
  FILE: CLAUDE.md
  PURPOSE: Primary memory file loaded at every Claude Code session start.
  Keeps context compact; links to .claude/memory/ for extended detail.
  Target: ≤ 80 lines. Trim aggressively before adding.
  Ref: https://code.claude.com/docs/en/memory
-->

## Project
<one-sentence description from Phase 2>

**Stack:** <inferred from scan>
**Entry:** <main entrypoint file or command>

## Commands
```bash
build:  <command>
test:   <command>
lint:   <command>
run:    <command>
```

## Structure
<3–6 line directory map — only key folders>

## Standards
- <code style rule 1>
- <naming convention>
- <any compliance requirement>

## Memory index
@.claude/memory/architecture.md
@.claude/memory/decisions.md
@.claude/rules/maintain.md
@.claude/rules/coding.md
```

---

### .claude/settings.json

```json
{
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(<build-command>)",
      "Bash(<test-command>)",
      "Bash(<lint-command>)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl * | bash)",
      "Bash(wget * | sh)",
      "WebFetch(<any non-whitelisted domain>)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"[AUDIT] $(date -u +%Y-%m-%dT%H:%M:%SZ) BASH: $CLAUDE_TOOL_INPUT\" >> .claude/audit.log"
          }
        ]
      }
    ]
  },
  "env": {
    "NODE_ENV": "development"
  }
}
```

> Populate allow/deny based on scan findings. Add only commands the project actually uses. Audit log provides traceability consistent with DO-178B traceability requirements.

---

### .claude/settings.local.json (if user opted in — gitignored)

```json
{
  "env": {
    "LOCAL_OVERRIDE": "true"
  }
}
```

---

### .claude/CLAUDE.local.md (gitignored)

```markdown
<!--
  FILE: CLAUDE.local.md
  PURPOSE: Personal session notes not shared with team.
  Gitignored. Override or extend CLAUDE.md conventions here.
-->

## My overrides
<!-- Add personal preferences below -->
```

---

### .claude/memory/architecture.md

```markdown
<!--
  FILE: .claude/memory/architecture.md
  PURPOSE: Structural map of the codebase for agent navigation.
  Updated when significant refactors occur.
-->

## Architecture
<inferred from scan: service boundaries, data flow, key modules>
```

---

### .claude/memory/decisions.md

```markdown
<!--
  FILE: .claude/memory/decisions.md
  PURPOSE: Log of non-obvious design decisions and their rationale.
  Prevents re-litigating settled choices.
  Format: decision | reason | date
-->

## Decision log
| Decision | Reason | Date |
|---|---|---|
| | | |
```

---

### .claude/kanban.md

```markdown
<!--
  FILE: .claude/kanban.md
  PURPOSE: Session-persistent issue tracker for the agent.
           At session start, agent reads this file, presents open items,
           and asks: "These issues were found — fix now, defer, or dismiss?"
           Agent updates status after each action.
  Quality bar: DO-178B-aligned — every fix must be traceable to a symptom,
               a root cause, and a verification step.
  DO NOT auto-fix without user confirmation on items marked [RISK: HIGH].
-->

## Backlog

| ID | Category | Severity | Item | Status |
|---|---|---|---|---|
<!-- Agent populates from Phase 1 scan findings -->
<!-- Severity: LOW / MED / HIGH / CRITICAL -->
<!-- Status: OPEN / IN-PROGRESS / FIXED / DEFERRED / DISMISSED -->

## Fix record

<!--
  When a fix is applied, append here:

  ### FIX-<ID>
  **File:** <path>
  **Symptom:** <what was wrong>
  **Root cause:** <why it existed>
  **Change:** <what was done>
  **Verification:** <how to confirm it is resolved — test command or manual step>
  **Date:** <ISO 8601>
-->
```

---

### .claude/improvements.md

```markdown
<!--
  FILE: .claude/improvements.md
  PURPOSE: Forward-looking backlog for maintainability, performance,
           security hardening, and new features.
           Not session-critical; reviewed periodically, not every session.
  Columns: priority (P1–P3), effort (S/M/L), and a clear acceptance criterion.
-->

## Improvements

| ID | Priority | Effort | Category | Description | Acceptance criterion |
|---|---|---|---|---|---|
<!-- Agent pre-populates from scan; user adds freely -->
<!-- Category: maintainability | performance | security | feature -->
<!-- Priority: P1=do soon, P2=next quarter, P3=someday -->
```

---

### .claude/rules/maintain.md

This file is loaded every session via the `rules/` directory. It is the self-maintenance contract — the agent reads and obeys it continuously, not just at setup time.

```markdown
<!--
  FILE: .claude/rules/maintain.md
  PURPOSE: Instructs the agent to actively maintain the .claude directory
           throughout every session. Loaded automatically by Claude Code
           from the rules/ directory on each session start.
  DO NOT DELETE — removing this file breaks session continuity.
-->

# .claude Maintenance Rules

## Session start (always, before any task)

1. Read kanban.md. If OPEN or IN-PROGRESS items exist, present them:
   ```
   Open issues:
   - [ID] [SEVERITY] <description>
   Fix now / defer / dismiss?
   ```
2. Fix confirmed items highest-severity first. Append a Fix record. Update status.
3. If no kanban items, check improvements.md and ask if user wants to action any P1 items.

## During work (continuous)

### kanban.md
- **Add** a new OPEN item when you discover a bug, security gap, or quality issue
  not already listed. Do not wait for session end.
- **Update** status to IN-PROGRESS when you start a fix.
- **Update** status to FIXED and append a Fix record when done.
- **Update** status to DEFERRED or DISMISSED if the user says so.
- Never delete rows — only change status.

### improvements.md
- **Add** a row when you identify a meaningful improvement (perf, security,
  maintainability, feature) that is out of scope for the current task.
- **Remove** a row only when the improvement has been fully implemented and verified.

### memory/architecture.md
- **Update** when files, services, or data flows are added, removed, or renamed.

### memory/decisions.md
- **Append** a row when a non-obvious design decision is made during the session.

### CLAUDE.md
- **Update** commands, stack, or structure section if the project changes.
- Keep ≤ 80 lines. Move overflow to memory/.

### settings.json
- **Add** to allow list if a new project command is introduced.
- **Add** to deny list if a new shell-injection risk is found (log to kanban first).

## Session end (always, after last task)

1. Confirm all in-progress kanban items are either FIXED or explicitly DEFERRED.
2. Confirm any decisions made this session are logged in decisions.md.
3. Confirm architecture.md reflects current state.
4. Do not summarise aloud unless the user asks — just write the files.

## Hard rules

- Never auto-fix [RISK: HIGH] kanban items without user confirmation.
- kanban Fix records are append-only. Never edit or delete a past record.
- audit.log is append-only. Never truncate.
- CLAUDE.md must remain loadable in < 5 seconds (≤ 80 lines, no binary content).
```

---

### .claude/rules/coding.md

Loaded every session alongside `maintain.md`. Contains behavioural guardrails that reduce common LLM coding mistakes.

```markdown
<!--
  FILE: .claude/rules/coding.md
  PURPOSE: Behavioural guardrails applied to every coding task in this project.
           Biases toward caution over speed — use judgment on trivial tasks.
           Loaded automatically from rules/ each session.
  DO NOT DELETE — removing this degrades output quality and traceability.
-->

# Coding Behaviour

## 1. Think Before Coding
Before implementing anything:
- State assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First
Minimum code that solves the problem. Nothing speculative.
- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Heuristic: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes
Touch only what you must. Clean up only your own mess.

When editing existing code:
- Don't improve adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it — don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

Test: every changed line must trace directly to the user's request.

## 4. Goal-Driven Execution
Transform tasks into verifiable goals before starting:
- "Add validation" → "Write tests for invalid inputs, then make them pass."
- "Fix the bug" → "Write a test that reproduces it, then make it pass."
- "Refactor X" → "Ensure tests pass before and after."

For multi-step tasks, state a plan first:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Weak success criteria ("make it work") require constant clarification.
Strong criteria let you loop independently.

## Health check
These rules are working if:
- Diffs contain fewer unnecessary changes.
- Fewer rewrites due to overcomplication.
- Clarifying questions come before implementation, not after mistakes.
```

---

## Phase 4 — Per-Session Routine

The agent follows `.claude/rules/maintain.md` automatically each session. No manual intervention needed. The rules file is the routine.

The only manual step: user responds to the opening kanban prompt (fix / defer / dismiss). Everything else the agent handles.

---

## Phase 5 — Documentation Standard

Every function or method added or modified in the project must include a docblock. Minimum required fields:

```
Purpose  — what it does, in one sentence
Params   — name, type, allowed values / constraints
Returns  — type and meaning
Raises   — exception/error types and when they occur
Issues   — open known issues with this function (link to kanban ID)
Fixed    — closed issues (kanban ID, fix summary, date)
```

**File-level header** (every file agent creates or substantively edits):

```
Purpose  — why this file exists
Owner    — module or domain it belongs to
Deps     — direct dependencies (imports / services)
Issues   — open kanban IDs relevant to this file
```

Keep docblocks factual. No filler phrases. One sentence per field unless unavoidable.

---

## Constraints

- Never write outside `.claude/` or the root `CLAUDE.md`.
- Never commit `settings.local.json` or `CLAUDE.local.md` (both gitignored).
- Never auto-fix a `[RISK: HIGH]` item without explicit user confirmation.
- CLAUDE.md stays ≤ 80 lines. Overflow goes to `.claude/memory/`.
- Audit log entries are append-only; never delete `.claude/audit.log`.
- All allow-listed Bash commands must match actual project usage found in scan.
