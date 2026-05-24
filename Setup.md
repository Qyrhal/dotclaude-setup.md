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
  rules/                ← persistent rule files
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

## Phase 4 — Per-Session Routine

At the **start of every session**, before any task:

1. Read `.claude/kanban.md`.
2. If open items exist, surface them:
   ```
   Open issues from kanban:
   - [ID] [Severity] <description>
   Fix now / defer / dismiss?
   ```
3. On user confirmation, fix items in order of severity (CRITICAL → HIGH → MED → LOW).
4. Append a Fix record to kanban.md for each resolved item.
5. Update status column.

Do not surface improvements.md unless the user asks or there are no kanban items.

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
