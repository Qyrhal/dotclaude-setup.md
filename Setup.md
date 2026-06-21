# setup.md

<!--
  PURPOSE: Agent-executable runbook — bootstraps agent configuration files only.
  Scope: files listed below. Never touches source code, CI config, or lockfiles.
  Refs:
    https://code.claude.com/docs/en/claude-directory
    https://agents.md/
-->

---

## Scope

| File | When created |
|---|---|
| `AGENTS.md` | AGENTS-only or Both modes |
| `CLAUDE.md` | Claude or Both modes |
| `.claude/settings.json` | Claude or Both modes |
| `.claude/settings.local.json` | Claude or Both modes, if user opts in |
| `.claude/CLAUDE.local.md` | Claude or Both modes |
| `.claude/commands/` | Claude or Both modes |
| `.claude/rules/maintain.md` | Claude or Both modes |
| `.claude/rules/coding.md` | Claude or Both modes |
| `.claude/memory/architecture.md` | Claude or Both modes |
| `.claude/memory/decisions.md` | Claude or Both modes |
| `kanban.md` / `.claude/kanban.md` | Always (location depends on mode) |
| `improvements.md` / `.claude/improvements.md` | Always (location depends on mode) |

**gitignore entries to add (always):**
```
.claude/settings.local.json
.claude/CLAUDE.local.md
.claude/audit.log
```

---

## Phase 1 — Scan

Read silently. Do not output findings yet.

| Check | What to look for |
|---|---|
| Language / runtime | `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `*.csproj` |
| Test runner | test scripts, pytest config, Makefile targets |
| Build / lint commands | build scripts, Makefile, `.eslintrc`, `.prettierrc`, `ruff.toml` |
| Secrets exposure | `.env` committed, hardcoded tokens, API keys in source |
| Dependency hygiene | lockfile present, `node_modules` committed |
| Error handling | bare `catch {}`, unhandled promise rejections, unrecovered panics |
| Existing agent files | `AGENTS.md`, `CLAUDE.md`, `.claude/`, `.cursor/`, `.aider.conf.yml` |

---

## Phase 2 — Questions

Ask all of the following in a single prompt. Omit any question the scan already answered with certainty.

```
I've scanned the repo. A few things I need to confirm:

0. Which agent configuration do you want?
   a) AGENTS.md only  — universal, works across all agents
   b) .claude/ only   — Claude Code-specific, full feature set
   c) Both            — AGENTS.md as shared base + .claude/ as Claude layer [recommended]

1. What is the primary purpose of this project? (one sentence)
2. What command runs tests?
3. What is the build command?
4. Any security or compliance requirements? (HIPAA, PCI-DSS, GDPR, SOC 2, etc.)
5. Any team conventions not obvious from code? (style, PR rules, commit format, etc.)
6. Should I create settings.local.json for personal local overrides? (yes/no)
7. [If existing agent files found] Existing files detected: <list>. Merge or replace?
```

**After Q0:**
- Mode **a** → generate AGENTS.md + kanban.md + improvements.md at project root only.
- Mode **b** → generate all `.claude/` files + CLAUDE.md. Skip AGENTS.md.
- Mode **c** → generate everything. CLAUDE.md defers to AGENTS.md; no duplication.

---

## Phase 3 — Generate Files

### AGENTS.md  _(modes: a, c)_

Place at project root. Keep agent-agnostic — no Claude-specific syntax.
If merging with an existing file, extend rather than overwrite.

~~~markdown
<!--
  FILE: AGENTS.md
  PURPOSE: Universal agent instructions. Works across Claude Code, OpenAI Codex,
           Cursor, Windsurf, Gemini CLI, Aider, GitHub Copilot, Amp, and others.
  MAINTENANCE: Update commands and conventions when the project changes.
               For monorepos, place a child AGENTS.md in each package directory.
  Ref: https://agents.md/
-->

# <project-name>

## Overview
<one-sentence description>

## Stack
<language, framework, runtime>

## Commands
    build: <command>
    test:  <command>
    lint:  <command>
    run:   <command>

## Structure
<3–6 line directory map — key folders only>

## Code style
- <inferred style rule or from Q5>
- <naming convention>
- <compliance requirement from Q4, if any>

## Testing
- Run the full test suite before any commit.
- Write or update tests for every change, even if not asked.
- A task is not done until tests pass.

## Security
- <inferred from scan or Q4 — e.g. "never log auth tokens", "validate all inputs">

## Conventions
- <commit / PR format from Q5, if any>
- Clarify ambiguities before implementing.
- Prefer the simplest solution that works.
- Touch only what the task requires.

## Agent session routine
At session start (AGENTS.md-only mode), before any task:
1. Read kanban.md. Present any OPEN or IN-PROGRESS items: fix / defer / dismiss?
2. Fix confirmed items highest-severity first. Append Fix record. Update status.
3. If no kanban items, check improvements.md — ask if user wants to action P1 items.
~~~

---

### kanban.md  _(all modes)_

Location: `.claude/kanban.md` in Claude/Both modes; project root in AGENTS-only mode.

~~~markdown
<!--
  FILE: kanban.md
  PURPOSE: Session-persistent issue tracker. Agent reads this at session start,
           surfaces open items, and asks: fix / defer / dismiss?
           Agent updates status after each action.
  RULE: DO NOT auto-fix [SEVERITY: HIGH] items without explicit user confirmation.
        Fix records are append-only — never edit or delete past records.
-->

## Backlog

| ID | Category | Severity | Item | Status |
|---|---|---|---|---|
<!-- Populate from Phase 1 scan findings -->
<!-- Severity: LOW / MED / HIGH / CRITICAL -->
<!-- Status: OPEN / IN-PROGRESS / FIXED / DEFERRED / DISMISSED -->

## Fix record

<!--
  Append for every resolved item. Never edit or delete past records.

  ### FIX-<ID>
  **File:** <path>
  **Symptom:** <what was wrong>
  **Root cause:** <why it existed>
  **Change:** <what was done>
  **Verification:** <test command or manual step>
  **Date:** <ISO 8601>
-->
~~~

---

### improvements.md  _(all modes)_

Location: `.claude/improvements.md` in Claude/Both modes; project root in AGENTS-only mode.

~~~markdown
<!--
  FILE: improvements.md
  PURPOSE: Forward-looking backlog — maintainability, performance, security,
           and new features. Reviewed periodically, not every session.
-->

## Improvements

| ID | Priority | Effort | Category | Description | Acceptance criterion |
|---|---|---|---|---|---|
<!-- Category: maintainability | performance | security | feature -->
<!-- Priority: P1=do soon  P2=next quarter  P3=someday -->
<!-- Effort: S / M / L -->
~~~

---

### CLAUDE.md  _(modes: b, c)_

Place at project root. Keep ≤ 80 lines; overflow to `.claude/memory/`.

In **Both** mode — reference AGENTS.md instead of duplicating content:

~~~markdown
<!--
  FILE: CLAUDE.md
  PURPOSE: Claude Code primary memory. Loaded at every session start.
  RULE: Keep ≤ 80 lines. Overflow to .claude/memory/.
-->

# <project-name>

> See AGENTS.md for full project context, commands, and conventions.

## Claude-specific notes
<anything that needs Claude-specific framing or overrides AGENTS.md for Claude sessions>

## Memory index
@AGENTS.md
@.claude/memory/architecture.md
@.claude/memory/decisions.md
@.claude/rules/maintain.md
@.claude/rules/coding.md
~~~

In **Claude-only** mode — CLAUDE.md must be self-contained. Use the same sections
as the AGENTS.md template above (Overview, Stack, Commands, Structure, Code style,
Testing, Security, Conventions), then append the Memory index block.

---

### .claude/settings.json  _(modes: b, c)_

Populate `allow` with only commands confirmed by the scan.

~~~json
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
      "Bash(wget * | sh)"
    ]
  },
  "env": {
    "NODE_ENV": "development"
  }
}
~~~

> **Note:** Claude Code hooks do not inject tool input as environment variables,
> so shell-based audit logging of tool arguments is not reliably available.
> Omit the `hooks` block; rely on Claude Code's native session transcript instead.

---

### .claude/settings.local.json  _(modes: b, c — if Q6 = yes)_

~~~json
{
  "env": {
    "LOCAL_OVERRIDE": "true"
  }
}
~~~

---

### .claude/CLAUDE.local.md  _(modes: b, c)_

~~~markdown
<!--
  FILE: CLAUDE.local.md
  PURPOSE: Personal session notes. Gitignored. Overrides CLAUDE.md locally.
-->

## My overrides
~~~

---

### .claude/memory/architecture.md  _(modes: b, c)_

~~~markdown
<!--
  FILE: .claude/memory/architecture.md
  PURPOSE: Structural map of the codebase. Update on significant refactors.
-->

## Architecture
<service boundaries, data flow, key modules — inferred from scan>
~~~

---

### .claude/memory/decisions.md  _(modes: b, c)_

~~~markdown
<!--
  FILE: .claude/memory/decisions.md
  PURPOSE: Log of non-obvious design decisions. Prevents re-litigating choices.
-->

## Decision log

| Decision | Reason | Date |
|---|---|---|
~~~

---

### .claude/rules/maintain.md  _(modes: b, c)_

~~~markdown
<!--
  FILE: .claude/rules/maintain.md
  PURPOSE: Self-maintenance contract for all agent config files.
           Loaded automatically each session. DO NOT DELETE.
-->

# Maintenance Rules

## Session start — before any task

1. Read `.claude/kanban.md`. If OPEN or IN-PROGRESS items exist, present them:
       Open issues:
       - [ID] [SEVERITY] <description>
       Fix now / defer / dismiss?
2. Fix confirmed items highest-severity first. Append Fix record. Update status.
3. If no kanban items, check `improvements.md` — ask if user wants to action P1 items.

## During work

**kanban.md**
- Add OPEN item immediately when you find a bug, security gap, or quality issue.
- Update to IN-PROGRESS when starting a fix; FIXED when done with Fix record appended.
- Update to DEFERRED or DISMISSED on user instruction.
- Never delete rows — only change status.

**improvements.md**
- Add a row for meaningful out-of-scope improvements found during work.
- Remove a row only when fully implemented and verified.

**AGENTS.md** (if present)
- Update commands, stack, structure, and conventions when the project changes.
- Keep it agent-agnostic — no Claude-specific syntax.

**memory/architecture.md**
- Update when files, services, or data flows change.

**memory/decisions.md**
- Append a row for every non-obvious design decision made this session.

**CLAUDE.md**
- Update if project context changes. Keep ≤ 80 lines.

**settings.json**
- Add to `allow` when a new project command is introduced.
- Add to `deny` when a shell-injection risk is found (log to kanban first).

## Session end — after last task

1. All in-progress kanban items are FIXED or DEFERRED.
2. Decisions made this session are logged in `decisions.md`.
3. `architecture.md` reflects current state.
4. AGENTS.md reflects any changed commands or conventions.
5. Write files silently. Do not summarise unless asked.

## Hard rules

- Never auto-fix [SEVERITY: HIGH] kanban items without user confirmation.
- kanban Fix records are append-only. Never edit or delete past records.
- CLAUDE.md must stay ≤ 80 lines.
- AGENTS.md is the cross-agent source of truth; CLAUDE.md defers to it per session.
~~~

---

### .claude/rules/coding.md  _(modes: b, c)_

~~~markdown
<!--
  FILE: .claude/rules/coding.md
  PURPOSE: Behavioural guardrails for every coding task. DO NOT DELETE.
-->

# Coding Behaviour

## 1. Think before coding
- State assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so.
- If something is unclear, name what's confusing and ask.

## 2. Simplicity first
Write the minimum code that solves the problem. Nothing speculative.
- No features beyond what was asked.
- No abstractions for single-use code.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

## 3. Surgical changes
Touch only what you must.

When editing existing code:
- Don't improve adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it — don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

## 4. Goal-driven execution
Before starting any task, restate it as a verifiable goal:
- "Add validation" → "Write tests for invalid inputs, then make them pass."
- "Fix the bug" → "Write a test that reproduces it, then make it pass."

For multi-step tasks, state a plan first:
    1. [Step] → verify: [check]
    2. [Step] → verify: [check]

## 5. Documentation standard
Every function added or modified gets a docblock:
    Purpose  — what it does, one sentence
    Params   — name, type, constraints
    Returns  — type and meaning
    Raises   — error types and when
    Issues   — open kanban IDs affecting this function

Every file created or substantively edited gets a header:
    Purpose  — why this file exists
    Owner    — module or domain
    Deps     — direct dependencies
    Issues   — open kanban IDs

Factual only. No filler. One sentence per field unless unavoidable.
~~~

---

## Constraints

| Rule | Detail |
|---|---|
| Scope | Only files listed in the Scope table. Never source code, CI config, or lockfiles. |
| Gitignore | Always add the three entries listed in the Scope section. |
| CLAUDE.md length | ≤ 80 lines. Overflow to `.claude/memory/`. |
| HIGH severity items | Never auto-fix without explicit user confirmation. |
| Fix records | Append-only. Never edit or delete. |
| AGENTS.md | Agent-agnostic. No Claude-specific syntax inside it. |
| Conflict resolution | If AGENTS.md and CLAUDE.md conflict, CLAUDE.md governs for Claude sessions only. |
