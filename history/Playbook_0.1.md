# AI Agent Setup Playbook

**Vendor-neutral edition** - for sharing outside a single company. Org-specific forges, CLIs, and internal MCP names are generalized here. If you maintain an internal fork, keep company-only runbooks there.

This document is addressed to an AI agent tasked with setting up the AI-agent infrastructure of a new project. It is self-contained: the reader does not need any background from a prior conversation.

**Scope:** Sections **1-15** and **Recipes** are **tool-agnostic** (any coding agent). **Section 19 (Security)** is tool-agnostic policy for secrets, MCP, and publishing. Sections **16-18** are short **adapter notes** for Claude Code, Cursor, and Codex only where behavior differs; the same metaprinciples apply to all.

Sources: [Habr - CLAUDE.md / context engineering](https://habr.com/ru/articles/972308/) (translation of HumanLayer ideas), **Bring Your Own Agent (BYOA)** pattern (`.ai/` SSOT, bootstrap, skills - see section 2), Anthropic [Claude Code hooks](https://docs.claude.com/en/docs/claude-code/hooks), [AGENTS.md](https://agents.md), Cursor [agent best practices](https://cursor.com/blog/agent-best-practices), OpenAI [Codex best practices](https://developers.openai.com/codex/learn/best-practices), and field experience.

---

## 0. How to use this document

1. Read **Metaprinciples** first - they are the filter for every decision.
2. Survey the project (structure, stack, team size, types of changes) - the right recipe depends on the project type.
3. Walk through the **New-project setup checklist** step by step.
4. For each decision, look at **Recipes by project type** first, then the general sections.
5. Before committing the setup, scan the **Antipatterns** section to confirm you are not falling into one.
6. Read **Security (section 19)** when wiring permissions, MCP, or anything that touches credentials or publishing.

This is not "run N steps." It is a **set of principles plus concrete recipes**. Apply with understanding of context, not mechanically.

### 0.1 Mapping: Habr (972308) and BYOA

| Idea in the source | Where it lives in this playbook |
| --- | --- |
| Stateless LLM; onboarding file | P2-P3, section 3, P11 |
| Few lines; progressive disclosure; relevance | P2-P3, section 3 "Relevance" |
| Linter vs LLM; `/init` noise; antipatterns | Sections 3, 8, 14, 18 |
| BYOA `.ai/`, bootstrap, memory | Sections 2, 5-7, 12 |
| Permissions, MCP, private forges / SSO | Sections 9-10, checklist stage 1 |
| Secrets, publishing, chat hygiene | **Section 19**, A11 |
| Global autonomy defaults | P6 examples in `~/.claude/CLAUDE.md` (or equivalent) |

If something is missing for your org, default to **P1** and **P6** before inventing new file types.

---

## 1. Metaprinciples

These **11** principles (P1-P11) override everything else in this document. If a specific recipe conflicts with a principle, follow the principle and ignore the recipe.

### P1. Single Source of Truth

All rules and context for agents live **in one place**. Native files (`CLAUDE.md`, `.cursor/rules/*.mdc`, `AGENTS.md` for Codex) are either **the same file** (via a symlink) or **auto-generated** from templates. Never edit two files with the same content by hand - that is the road to drift.

### P2. Minimum context, maximum signal

Most agents already have around 50 reasonable rules baked into the system prompt ("do not write redundant comments", "respond concisely", "edit existing files instead of creating new ones", etc.). **Do not duplicate them.** Only write what:

- the agent cannot infer from reading the code;
- the agent cannot infer from file names and structure;
- contradicts the agent's default behavior.

AGENTS.md size: **under 300 lines is good, under 60 is great**.

**Instruction budget (qualitative):** many harnesses already ship a large system prompt. Every extra line in `AGENTS.md` competes for attention. Empirical write-ups (e.g. [Habr 972308](https://habr.com/ru/articles/972308/)) note that **too many narrow or rarely relevant rules** make the model attend to them **less** - treat that as another reason to keep the root file **universally applicable** and push edge cases to linked docs, modules, or skills.

### P3. Progressive Disclosure

Do not stuff the whole project documentation into AGENTS.md. Instead, **link to files**: "testing rules in `docs/testing.md`", "build commands in [section]". The agent will read what it needs for a specific task. This saves context on 90% of tasks and still provides full material on the remaining 10%.

### P4. Universality via AGENTS.md

[AGENTS.md](https://agents.md) is supported natively by OpenAI Codex CLI, Aider, OpenCode, Cursor, and others listed on the spec site. The format is an open convention (stewarded by the Agentic AI Foundation under the Linux Foundation). **Use AGENTS.md as the canonical format.** For Claude Code, which only reads `CLAUDE.md`, create a symlink. This gives portability across tools without duplication.

**Nested files:** per the AGENTS.md FAQ, the **closest** `AGENTS.md` to the file or working context usually wins; explicit user prompts override file guidance. Align this with your hierarchy: child files own local rules; the root file is the global map and shared invariants (see section 4).

### P5. Adapters are not source of truth

If a project auto-generates files (`CLAUDE.md`, `.cursor/rules/*.mdc`, skill wrappers), they **must** start with:

```
# AUTO-GENERATED FILE. DO NOT EDIT MANUALLY.
# Source of truth: <path>
# Regenerate: <command>
```

This prevents the "coworker edited CLAUDE.md and the next regen wiped it" situation.

### P6. Knowledge is classified

Every statement that emerges while working with an agent belongs to one of these categories:

| Category | Where it goes |
|---|---|
| Universal user preference | `~/.claude/CLAUDE.md` (or equivalent at the user level) |
| Whole-project rule | `<repo>/AGENTS.md` |
| Single-module rule | `<module>/AGENTS.md` |
| Repeatable 3+ step workflow | Slash command / skill |
| Local-only facts (stand names, machine paths) | gitignored memory (see Memory section) |
| Command the user constantly approves by hand | `permissions.allow` |

**If a piece of knowledge is useful to everyone, memory is the wrong place.** This rule comes from the `learn`-skill pattern and must be enforced.

#### Examples: what often lives in `~/.claude/CLAUDE.md` (or other user-global agent prefs)

Personal **cross-repo** defaults the harness would not infer:

- **Response shape:** terse vs verbose, whether preambles or closing recap blocks are wanted by default.
- **Autonomy boundaries** many engineers set globally unless a repo overrides them: e.g. do not run **heavy** local test suites without being asked; do not **VCS commit or push** without an explicit instruction; do **not** publish **issue tracker** or **code review** comments via integrations unless the user clearly asked to publish (read-only tools are fine).
- **Planning gate:** for multi-file or public-API changes, show a short plan and wait; for a single obvious edit or an explicit "just do X", execute immediately.

Chat language defaults belong with **P10** (same file is fine). If a rule applies to **one** product or repo only, it belongs in that repo's `AGENTS.md`, not in the global file (see also A10).

### P7. Do not proliferate files

Before creating a new file, ask: "can this live in an existing one?". Four short notes in one AGENTS.md beat four separate files. Files should be few and meaningful.

### P8. ASCII typography in files

Default to ASCII (plus Cyrillic for Russian prose). No non-ASCII typography in configs, code, AGENTS.md, or slash commands: em-dash (U+2014), en-dash (U+2013), arrows U+2190..U+21FF, curly quotes (U+201C/201D/2018/2019), guillemets, ellipsis U+2026, emoji. They are hard to type and grep. ASCII equivalents: ` - ` or `:`, `->` / `<-`, plain `"` and `'`, `...`.

Simple ASCII markers `+`, `-`, `?`, `*` in tables and lists are fine alongside `yes`/`no`/`maybe`. The ban is on non-ASCII only.

Exception: use a non-ASCII character when it is genuinely warranted (it is the subject of discussion, or omitting it loses meaning).

### P9. Self-updating context

Configuration must stay current. After a complex task or important finding, the agent **proposes** (does not silently do) an update to the relevant config when the knowledge applies to future sessions.

Where things go: see P6. Triggers:

- The user corrected the agent's behavior ("don't do X", "do Y instead").
- A non-trivial infrastructure detail surfaced (workaround for a misleading error, a non-obvious build step, hidden coupling between modules).
- A task finished with a pitfall that should not be hit again.

Do **not** propose updates after every task - only when there is something real to record and it does not duplicate what is already there. Keep configs current and compact at the same time - **one precise bullet beats a paragraph** of narrative nobody will trim later.

See also the `Memory` section on the `learn`-skill, which automates this loop.

### P10. Language: Russian and English (chat vs durable config)

**Preferred human languages:** Russian and English. No hard rule that the whole repo must be one language.

- **Live work (chat, reviews, explanations):** follow the **user's and team's priority** - often **Russian** (or another language) for language-first teams. The agent should not force English in conversation. **Mixed** languages in one message: follow the **dominant** language.
- **Project prose (`README`, design docs):** match the **dominant documentation language**. If the team writes docs in Russian, a Russian `AGENTS.md` can be easier for humans to maintain and still work well for models.
- **Durable machine-oriented text** (`AGENTS.md` invariants, rule bodies, skill specs, frontmatter `description` fields, auto-generated headers): **English is often a good default** - dense keywords, shared examples online, and slightly lower ambiguity for instruction-following on frontier models. Switch to Russian for those files when **human maintenance** or **strict alignment** with Russian-only docs clearly wins.
- **Cross-project templates** (shared playbooks, skill templates that ship to many repos): keep **one** language per file - usually English unless the template is explicitly for Russian-only consumers.

**Heuristic:** Russian where people **read and edit** daily together; English where the text is mostly **stable instructions for the model** and the team is comfortable editing English. Avoid duplicating the same policy in two languages in the same file - pick one primary language per artifact.

**Short version:** chat = user's language; committed agent instructions = the language the team will keep **accurate** (Russian or English per above, not both duplicated in one place).

### P11. Shape task prompts, not only static repo rules

`AGENTS.md` and rules reduce repeated mistakes; they do not replace clear **per-task** instructions. A practical structure (used across Codex guidance) is:

- **Goal** - what to change or build.
- **Context** - which files, errors, docs, or examples matter (`@` / mentions where the tool supports them).
- **Constraints** - architecture, safety, style of review, or team conventions beyond defaults.
- **Done when** - observable completion (tests pass, behavior fixed, diff scope bounded).

Match **reasoning / effort level** to task difficulty (fast for small edits; higher for debugging or long agentic runs). Teach the team this shape in onboarding, not only the contents of `AGENTS.md`.

---

## 2. Architecture: Bring Your Own Agent (BYOA)

BYOA (**Bring Your Own Agent**) is a pattern for the case where a team uses **different AI agents** (Claude Code, Cursor, Codex, etc.) on **one project**. A typical layout you can copy: **committed** `.ai/` (AGENTS text, skill sources, templates), **generated** per-tool adapters, **gitignored** local memory, **CI** running bootstrap `--check`, and thin **wrapper skills** so Claude finds canonical `SKILL.md` paths.

### One-line idea

```
.ai/ - source of truth (committed)
CLAUDE.md, .cursor/rules/*.mdc, .claude/skills/** - auto-generated adapters
```

### When full BYOA is appropriate

- Team size > 3 people.
- Two or more different AI agents in use across the team.
- Monorepo or middle-to-large project (5 or more modules).
- The project has dedicated **skills with tools** (see Skills section).

### When BYOA is overkill

- Single developer using a single agent.
- Sandbox or personal project.
- Fewer than 10 files in the project.
- No skills with tools (only a plain text AGENTS.md).

In those cases use a **simplified scheme**: one `AGENTS.md` in the root plus `CLAUDE.md` as a symlink to it. No `.ai/`, no bootstrap script, no templates.

### Full BYOA structure

```
<repo>/
|-- AGENTS.md -> .ai/AGENTS.md           (symlink OR auto-generated)
|-- CLAUDE.md -> .ai/AGENTS.md           (symlink OR auto-generated)
|-- .cursor/rules/ai-router.mdc          (auto-generated, alwaysApply: true)
|-- .claude/skills/<wrapper>/SKILL.md    (auto-generated per .ai/skills/*)
|-- .ai/
|   |-- AGENTS.md                        (root context and routing)
|   |-- bootstrap/
|   |   +-- adapters/
|   |       |-- claude-root.md.tmpl
|   |       |-- claude-skill-wrapper.md.tmpl
|   |       +-- cursor-ai-router.mdc.tmpl
|   |-- skills/
|   |   +-- <skill_name>/
|   |       |-- SKILL.md
|   |       +-- tools/                   (optional skill scripts)
|   |-- tools/                           (shared AI tools, not skill-scoped)
|   +-- memory/                          (gitignored, local-only)
|-- tools/
|   +-- ai/
|       |-- bring_your_own_agent_bootstrap.py
|       +-- bring_your_own_agent_bootstrap.sh
+-- <module>/
    +-- .ai/
        |-- AGENTS.md                    (module context)
        |-- tools/                       (module-specific)
        +-- skills/                      (optional)
```

### What to put in `.gitignore`

```
# Adapters are regenerated locally
**/CLAUDE.md
.cursor/rules/ai-router.mdc
.claude/
.junie/
.opencode/

# Local memory stays local
**/.ai/memory/**
```

Adapters are **not committed** because each developer runs bootstrap locally - no merge conflicts on autogen files. Only `.ai/` source files are committed.

Alternative for small projects: commit `CLAUDE.md` as a symlink to `AGENTS.md`. No bootstrap script needed. Simplicity beats "drift protection" when there is nothing to regenerate.

---

## 3. AGENTS.md: contents and rules

### File structure

The root `AGENTS.md` answers three questions:

1. **WHAT** - what this project is and what its boundaries are.
2. **WHY** - what problem it solves and what its invariants are.
3. **HOW** - how to work with it: build/test commands (or links), rules, gotchas.

### What to write

- **Architectural invariants** the agent cannot infer from code: "all IPC over gRPC; REST is health-check only".
- **Prohibitions with rationale**: "do not use lombok - we tried, it conflicts with kotlinx.serialization". Without rationale the rule risks being overridden in edge cases.
- **Frequently needed commands**: build, test, lint, run. With exact syntax.
- **Routing** to child AGENTS.md files (see Hierarchy).
- **Contracts with external systems**: which APIs must not break, which schemas are versioned.
- **Non-trivial stack quirks**: "Gradle must be run from the repo root only", "DB migrations through flyway, never by hand".

### What NOT to write

- Code style: variable naming, formatting. That is the linter/formatter's job.
- Anything obvious from file names: "`users_repository.go` deals with users". Useless.
- Project history, postmortems. Belongs in a wiki, not an agent file.
- Full dependency lists. The agent will read `go.mod` / `package.json` / `pyproject.toml` itself.
- Long framework tutorials. If a developer does not know Spring, the agent does. Extra explanation is noise.
- Things already in the agent's system prompt ("respond concisely", "do not write comments").

### Relevance beats volume (why "fat" root files underperform)

Some coding harnesses inject `CLAUDE.md` / project context with a **relevance hint** - the model is allowed to treat long, task-specific blobs as **low priority** when they do not match the current task (see [Habr 972308](https://habr.com/ru/articles/972308/) and upstream HumanLayer material). Regardless of exact wording, the lesson is the same as P2 and P3: **narrow or rare rules in the always-loaded file** compete with global invariants. Keep the root **universally useful**; push edge cases to linked docs, module `AGENTS.md`, scoped rules, or skills.

### Size (rule of thumb)

- Root `AGENTS.md`: **< 60 lines** ideal, **< 300** hard maximum.
- Module `AGENTS.md`: **< 30 lines** ideal.
- If you exceed this, that is a signal to move material into `docs/<topic>.md` files and link from AGENTS.md.

### Root AGENTS.md template

```markdown
# <Project name>

## WHAT

<1-3 sentences: what the project is, who it serves, where it is deployed.>

## WHY

<1-2 sentences: the business or technical problem it solves.>

## HOW to work

- <Top-level invariant 1.>
- <Top-level invariant 2.>
- <Exact build command.>
- <Exact test command.>

## Hierarchy

- `<module-1>/**` -> `<module-1>/AGENTS.md`
  (one line: what the module owns and when to drill in)
- `<module-2>/**` -> `<module-2>/AGENTS.md`

## Details

- Test strategy: `docs/testing.md`
- Deploy: `docs/deploy.md`
- External contracts: `docs/api-contracts.md`
- Security / MCP publishing policy (if any): `docs/security-agents.md` or one line in HOW per playbook section 19
```

### Module AGENTS.md template

```markdown
# <Module name>

Use this file for tasks under `<module>/**`.

## Scope

- <1-3 sentences on what the module does.>

## Work rules

- <Only rules specific to this module.>
- <Module-specific build/test commands.>

## Parent context

Also read `/AGENTS.md` for global routing.
```

---

## 4. AGENTS.md hierarchy

In a monorepo or multi-module project: **one root file plus one per module**.

### Rules

- The root AGENTS.md contains a **module map** with one line of description per module. The agent uses this line to decide whether to drill down.
- Each module AGENTS.md ends with a **"Parent context: also read /AGENTS.md"** section. Explicit two-way linkage.
- Module AGENTS.md does **not** repeat the root file. Only module-specific content.
- Depth: typically 1 level. 2 levels (root -> module -> submodule) is acceptable for very large monorepos.

**Precedence vs tools:** many agents resolve **nearest** `AGENTS.md` automatically. Your "also read parent" line helps when the harness does not merge parent + child in one shot, or when a human needs a clear drill-down path. If a tool documents different merge rules, follow the tool and keep files consistent so any merge order stays sane.

### When hierarchy is needed

- The project has 3 or more independent modules.
- Modules have **different** build/test/deploy rules.
- The team is divided by module.

### When hierarchy is not needed

- A single-build project with one command.
- All modules build and test with the same command.
- Fewer than 5 rule files.

---

## 5. Skills

A skill is a reusable procedure with its own tools. Clear distinction from AGENTS.md: AGENTS.md is **context** (what-where-how); a skill is **procedure** (a step sequence).

### When to extract a skill

- The procedure repeats (3 or more applications in the foreseeable future).
- The procedure has non-trivial steps: confirmation gates, tool calls whose output you must parse, permission checks.
- The procedure outlives a single agent (logic, not a one-off instruction).

### Skill structure (BYOA)

```
.ai/skills/<name>/
|-- SKILL.md          # canonical definition with frontmatter
+-- tools/            # scripts referenced from SKILL.md (optional)
```

### SKILL.md frontmatter

```yaml
---
name: <name>
description: <one line for the agent on what the skill does>
user_invocable: true   # set if the user can call it as /<name>
---
```

`description` matters - the agent uses it to decide whether to activate the skill. It should contain **trigger words** (e.g. "redeploy", "deploy", "production") and be concrete.

### SKILL.md content

- **Goal**: one sentence.
- **Parameters**: required inputs (host, component, flags).
- **Required context**: files to read first (e.g. memory with environment params).
- **Steps**: numbered, with exact commands.
- **Decision policies**: for interactive parts, **do not hardcode** "ask in chat". Write: "if runtime supports interactive tool approval, use it; otherwise ask in text" (see P5).
- **Output contracts**: if the skill consumes tool output, describe the format (e.g. "last non-empty stdout line: `IMAGE_REF=<repo:tag>`").

### A skill must not duplicate AGENTS.md

If `core/AGENTS.md` has the build command for `core`, a `redeploy` skill must **reference it**: "use the corresponding module AGENTS.md as source of truth for build commands. Do not duplicate." This is critical for drift prevention.

### Wrapper for Claude Code

Claude Code only reads `~/.claude/skills/<name>/SKILL.md` or `<repo>/.claude/skills/<name>/SKILL.md`. In BYOA, the bootstrap auto-generates a wrapper that points to the canonical skill:

```markdown
---
name: <wrapper_name>
description: Auto-generated wrapper for canonical .ai skill.
user_invocable: true
---

# AUTO-GENERATED FILE. DO NOT EDIT MANUALLY.
# Source of truth: .ai/bootstrap/adapters/claude-skill-wrapper.md.tmpl
# Regenerate: python3 tools/ai/bring_your_own_agent_bootstrap.py --write

Use canonical skill definition: `<.ai/skills/.../SKILL.md>`

Canonical helper tools:
- `<path/to/tool1>`
- `<path/to/tool2>`
```

`wrapper_name` must be unique: a skill in `core/.ai/skills/redeploy/` becomes `core--redeploy`.

---

## 6. Slash commands

Slash commands are **prompt shortcuts**: the user types `/<name>` and the agent runs the prompt from a markdown file. Difference from a skill: a slash command is a **prompt recipe**, not a procedure with tools.

### Where they live (Claude Code)

- Global: `~/.claude/commands/<name>.md` - available in every project.
- Project: `<repo>/.claude/commands/<name>.md` - only in this project.

### Format

```markdown
---
description: <one line on what the command does>
argument-hint: [<hint for the arguments>]
---

<Prompt body. $ARGUMENTS is substituted with what the user typed after the command name.>

<Clear steps. Clear output format. What not to do.>
```

### When to extract a slash command

- A prompt repeats (you would type it 3+ times).
- A prompt is more than 5 lines and you would rather not retype it.
- It is useful in any project (then make it global).

### Universal global slash commands

Useful templates for `~/.claude/commands/`:

- `/explain [file/symbol]` - explain code, focusing on non-obvious bits.
- `/diff [focus]` - analyze uncommitted changes (works with `git`, `svn`, or your team's primary VCS CLI).
- `/learn [focus]` - extract reusable knowledge from recent context (see Memory section).
- `/ticket <id>` - read a tracker ticket through MCP.

### Slash command antipatterns

- Slash command for a one-off operation - just write the prompt.
- Slash command duplicating a built-in skill (`/security-review` is already a Claude Code skill - do not redo it).
- Slash command without `description` - the agent cannot suggest it correctly.

---

## 7. Memory

Memory is a **local, non-committed** layer of knowledge visible to the agent.

### Two memory layers

1. **Agent auto-memory** (Claude Code: `~/.claude/projects/<encoded-path>/memory/`). The agent reads and writes by type:
   - `user` - who the user is, role and preferences.
   - `feedback` - corrections and confirmations from the user.
   - `project` - current initiatives, deadlines, who is doing what.
   - `reference` - pointers to external systems.

2. **Structured memory in `.ai/memory/`** (the BYOA PR pattern). Holds **typed data**: stand configurations, key path names, service names. The schema is defined in AGENTS.md.

### When to use which

- Free-form textual observations -> agent auto-memory.
- Structured data that skills reference -> `.ai/memory/*.md` with a YAML schema.
- Credentials, tokens, keys -> **neither**, leave them in Keychain or a secrets store. Memory is not for secrets.

### Memory content rules

(From the BYOA PR, "Rules for this memory file":)

- Only actual local environment records for the current user.
- Do not store instructions, templates, or workflow explanations.
- If the schema/rules change, update AGENTS.md, not the memory file.

### `learn`-skill: feedback loop

After a task or a non-trivial session, extract knowledge and classify it. See the full template in the global `/learn` slash command (when configured) or inside the project at `.ai/skills/learn/SKILL.md`.

Signal words for the context scan: `unexpected`, `non-obvious`, `pitfall`, `workaround`, `must`, `always`, `never`, `failed`, `regression`, user corrections.

Each accepted finding is 3 lines:

1. **Trigger / Context** - when it applies.
2. **Rule / Action** - what to do.
3. **Why** *(optional)* - the reason.

### Example memory schema (deploy environments)

In AGENTS.md:

```yaml
environments:
  - name: <unique>
    type: docker-compose | kubernetes | bare-metal
    host: <fqdn>
    login_user: <ssh user>
    deploy_user: <effective user after sudo>
    docker_compose_path: <path>
    services:
      <component>: <docker service name>
    health_probes:
      <component>: <url>
```

In `.ai/memory/environments.md`: real values for this machine.

---

## 8. Hooks (Claude Code specific, idea is universal)

Hooks are external commands the agent harness runs in response to events. **Behavior that cannot be expressed via memory or preferences.**

### Types (Claude Code) - common subset

The product defines **many** hook events (session, turn, tool loop, compaction, worktrees, MCP elicitation, config changes, etc.). Start with these when designing policies:

- `SessionStart` - session begins or resumes (inject context; cannot block the same way as tool hooks).
- `UserPromptSubmit` - user submitted a prompt (validation / extra context; can block).
- `PreToolUse` - before a tool call (can block or adjust input).
- `PostToolUse` / `PostToolUseFailure` - after tool success or failure (formatters, logging).
- `Stop` / `SubagentStop` - end of assistant turn or subagent (task enforcement loops, summaries).
- `PreCompact` / `PostCompact` - around context compaction (backups, transcripts).

**Authoritative list and JSON schemas:** [Claude Code hooks reference](https://docs.claude.com/en/docs/claude-code/hooks). Event names and behavior change with releases; link the team to that page instead of copying a stale table into many repos.

### Typical use cases

- **Auto-lint after edit**: `PostToolUse` on `Edit|Write` -> run a formatter/linter on the changed file. E.g. `prettier --write`, `gofmt`, `rustfmt`, or your project's formatter command.
- **Block dangerous commands**: `PreToolUse` on `Bash` -> refuse if the command contains `rm -rf` or `--no-verify`.
- **Telemetry / session log**: `SessionStart` -> write metadata.
- **Notify on finish**: `Stop` -> `say "done"` / a desktop notification.

### Where they live

- Global: `~/.claude/settings.json` -> `hooks` section.
- Project: `<repo>/.claude/settings.json`.
- Local user: `<repo>/.claude/settings.local.json` (gitignored).

### Format

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "<shell command>"
          }
        ]
      }
    ]
  }
}
```

### When NOT to set a linter hook

- The project is **mixed-stack** (Python + Go + Shell + SQL in one folder). One linter does not fit. Use a linter in the submodule with a homogeneous stack instead.
- The linter is slow (> 2s). It will ruin UX. Use `--check` in CI instead.
- The linter needs whole-project context (e.g. ESLint with types). Use a git pre-commit hook, not a Claude hook.

---

## 9. Permissions

Products such as **Claude Code** use `permissions.allow` (and related fields) for actions that **skip approval prompts**. Treat that list as a **security boundary**, not a way to silence every prompt.

### Rules of thumb

- Allow only **read-only** or trivially reversible work.
- Never allow **mutating** VCS (`push`, `commit`, `--force`, mass delete) for whichever CLI the repo actually uses; mirror the same idea if you use a second CLI alongside `git`.
- For **shell**: prefer explicit `Bash(<command> *)` entries, not broad wildcards.
- For **MCP and other integrated tools**: classify each operation as **read** vs **write**; auto-approve only reads your team is comfortable with. **Tool identifiers are product- and server-specific** - copy exact strings from your agent's permission or tool inspector; do not rely on a universal naming pattern from examples online.
- **Secrets in JSON:** see **section 19** (canonical policy).

### Example: read-only `git` shell (illustrative)

```json
{
  "permissions": {
    "allow": [
      "Bash(ls)", "Bash(ls *)",
      "Bash(pwd)", "Bash(whoami)", "Bash(date)",
      "Bash(cat *)", "Bash(head *)", "Bash(tail *)", "Bash(wc *)", "Bash(file *)",
      "Bash(rg *)", "Bash(grep *)", "Bash(find . *)",
      "Bash(jq *)", "Bash(which *)",
      "Bash(git status)", "Bash(git status *)",
      "Bash(git log)", "Bash(git log *)",
      "Bash(git diff)", "Bash(git diff *)",
      "Bash(git branch)", "Bash(git branch --show-current)",
      "Bash(git show *)", "Bash(git blame *)"
    ]
  }
}
```

If the team uses **another read-only CLI** in parallel, add the same *class* of entries for that tool (status/diff/log/show - exact flags vary).

### Merge order (Claude Code)

Most local wins for scalars; `allow`/`deny` lists **merge** and **`deny` wins** - confirm in current product docs. Typical stack: `<repo>/.claude/settings.local.json` (gitignored) -> `<repo>/.claude/settings.json` -> `~/.claude/settings.json`.

### Optional knobs (no secrets in repo)

`env` for API base URL, `effortLevel`, `statusLine`, `hooks.SessionStart` for telemetry - describe **mechanisms** only; values that authenticate belong in gitignored locals, env, or vendor token-helper hooks (**section 19**).

### Filling the list

Use transcript-driven helpers when your product ships them (e.g. Claude Code **`fewer-permission-prompts`** skill) and **human review** before widening write access.

---

## 10. MCP

[MCP](https://modelcontextprotocol.io/) connects agents to other systems (tracker, forge, search, DB). Put **non-secret** server definitions in committed **`.mcp.json`**; keep tokens in gitignored paths or environment (**section 19**).

```json
{
  "mcpServers": {
    "<server-name>": {
      "command": "<path/to/server>",
      "args": ["<arg1>", "<arg2>"]
    }
  }
}
```

### When it pays off

Repeated access to authenticated APIs, internal search, or structured data where **public WebFetch** is wrong or blocked.

### What to capture in `AGENTS.md` (private / large org)

Keep it short: real **build and test** commands; how **reviews and tickets** are referenced; that **internal SSO URLs** usually need MCP (or org-approved auth), not public web tools; prefer **scoped search** over whole-tree `find` on huge repos; **read vs write** policy for automation (same intent as section 9). Mention extra ignore files only if they change what the agent should read.

---

## 11. Multi-agent: one setup for Claude / Cursor / Codex

The **strategy below is the same for every tool**: one canonical `AGENTS.md` (plus module files), optional BYOA adapters, same MCP and permission philosophy. Tool-specific file names in sections **16-18** are implementation details only.

### Strategy

- **Source of truth**: `AGENTS.md` in the root (and in each module).
- **Codex CLI**: reads `AGENTS.md` natively. Nothing to do.
- **OpenCode, Aider**: read `AGENTS.md` natively. Nothing to do.
- **Cursor**: recent versions read `AGENTS.md` natively. As a fallback, add `.cursor/rules/agents.mdc` that points to `@AGENTS.md` with `alwaysApply: true`.
- **Claude Code**: only reads `CLAUDE.md`. Create a **symlink** `CLAUDE.md -> AGENTS.md`. Skills and slash commands stay in `.claude/`.
- **Some vendor IDEs** ship their own project-rules format and may not read `AGENTS.md` - support them separately if your team uses them.

### Symlink command (UNIX)

```bash
ln -sf AGENTS.md CLAUDE.md
# If Cursor is recent, that is enough. Otherwise:
mkdir -p .cursor/rules && cat > .cursor/rules/agents.mdc <<'EOF'
---
alwaysApply: true
---
See AGENTS.md for primary context.
EOF
```

### Alternative to symlinks: auto-generation

If symlinks are inconvenient (e.g. Windows contributors in the team), the bootstrap script copies AGENTS.md to CLAUDE.md with an AUTO-GENERATED header. See section 12.

---

## 12. Bootstrap script (for BYOA)

The script syncs adapters with templates. It runs:

- Locally before working: `python3 tools/ai/bring_your_own_agent_bootstrap.py --write`.
- In CI: `--check` (exit 1 on drift).

### Minimum responsibilities

1. Render templates into native files (`CLAUDE.md`, `.cursor/rules/ai-router.mdc`).
2. Scan `**/.ai/skills/*/SKILL.md` and render wrappers into `.claude/skills/`.
3. Remove stale wrappers (by the `AUTO-GENERATED` marker).
4. `--check` mode: diff and exit 1 on drift.

### Idiomatic implementation (Python)

See your own bootstrap script or an internal template repo for a full example. Key pieces:

- `STATIC_MAPPINGS = [(template, target), ...]` - static template-to-adapter pairs.
- `discover_ai_skills()` - `Path.rglob(".ai")` then `glob("skills/*/SKILL.md")`.
- `wrapper_dir_name_for()` - unique wrapper directory name per skill (derived from module path).
- `AUTO_GENERATED_MARKER` - the header string used to find stale wrappers.

### Running it from the agent: skill `aiimport`

So users do not have to remember the command:

```markdown
---
name: aiimport
description: Regenerate native Cursor/Claude adapters from .ai source-of-truth templates.
user_invocable: true
---

# AI Import

Run from the repository root:

\`\`\`bash
python3 tools/ai/bring_your_own_agent_bootstrap.py --write
\`\`\`

Then verify:

\`\`\`bash
python3 tools/ai/bring_your_own_agent_bootstrap.py --check
\`\`\`
```

---

## 13. New-project setup checklist

### Stage 1. Survey the project

- [ ] Read `README.md` / `Makefile` / `package.json` / `pyproject.toml` / build files your stack uses (`CMakeLists.txt`, `BUILD`, `go.mod`, etc.) to understand the stack.
- [ ] Map top-level directories (`ls`, IDE tree, or repo-specific search). On **very large monorepos**, prefer **scoped search** (ripgrep, IDE, or internal search MCP) over `find` across the whole tree.
- [ ] Check existing AI configs: search for `CLAUDE.md`, `AGENTS.md`, `.cursor`, `.claude`, `.codeassistant`, `.codex`, `.agents` (use your VCS or `rg --files -g 'CLAUDE.md'` style queries - avoid giant `find` on huge trees).
- [ ] Determine project type (see **Recipes**).
- [ ] Identify which agents the team uses (ask, or look for committed `.cursor/`, `.claude/`, `.opencode/`, `.codex/`, etc.).
- [ ] If **private forge / SSO monorepo**: document primary VCS CLI, review URLs, issue tracker MCP, and internal search policy (see section 10, "Private corporate stack").

### Stage 2. Baseline setup (always)

- [ ] Create `AGENTS.md` in the root (see Root AGENTS.md template).
- [ ] If the team uses Claude Code: `ln -sf AGENTS.md CLAUDE.md`.
- [ ] If Cursor: check whether the installed version reads AGENTS.md natively. If not, add `.cursor/rules/agents.mdc`.
- [ ] Verify `.gitignore` either commits adapters (symlinks) or ignores them (BYOA).

### Stage 3. If monorepo / many modules

- [ ] One `<module>/AGENTS.md` per module (see Module template).
- [ ] In the root AGENTS.md add a Hierarchy section with one line per module.
- [ ] Every module AGENTS.md ends with "Parent context: also read /AGENTS.md".

### Stage 4. If skills are needed

- [ ] Identify which procedures repeat (3+ applications).
- [ ] For each: create `<module>/.ai/skills/<name>/SKILL.md` with all sections (Goal, Parameters, Steps, Decision policies, Output contracts).
- [ ] If a skill has its own scripts, put them in `<module>/.ai/skills/<name>/tools/`.

### Stage 5. If BYOA (2+ agents, 5+ modules)

- [ ] Create `tools/ai/bring_your_own_agent_bootstrap.py` (see section 12).
- [ ] Adapter templates -> `.ai/bootstrap/adapters/*.tmpl`.
- [ ] Update `.gitignore`: exclude adapters (`**/CLAUDE.md`, `.cursor/rules/ai-router.mdc`, `.claude/skills/`).
- [ ] Add an `aiimport`-skill for one-button regeneration.
- [ ] Add a CI step: `python3 tools/ai/bring_your_own_agent_bootstrap.py --check`.

### Stage 6. Permissions, MCP, hooks (as needed)

- [ ] `<repo>/.claude/settings.json` with the baseline `permissions.allow` (see section 9).
- [ ] If external systems are involved: `.mcp.json` with the relevant servers.
- [ ] If the stack is homogeneous: a `PostToolUse` hook for the linter.

### Stage 7. Memory

- [ ] If there is structured data (stands, configs), define the schema in AGENTS.md.
- [ ] Create `.ai/memory/` and add it to `.gitignore`.
- [ ] **Do not** populate memory yourself - let the user decide what belongs there.

### Stage 8. Review

- [ ] Walk through **Antipatterns** (section 14).
- [ ] **Security (section 19):** no secrets in tracked agent config; `permissions.allow` / MCP match read-vs-publish policy; redact before pasting configs into chats or tickets.
- [ ] Confirm the root AGENTS.md is at most 60 lines (soft) / 300 lines (hard).
- [ ] Read AGENTS.md through the eyes of a **new developer**. Is anything redundant? Obvious? Duplicating the system prompt?

---

## 14. Antipatterns

### A1. Uncurated auto-generated agent docs (`/init`, "scan the repo" dumps)

Tools that scaffold `AGENTS.md` or `CLAUDE.md` from a template often produce **long, low-signal** text (obvious directory listings, generic advice). That dilutes what the model attends to. **Curate:** keep WHAT/WHY/HOW, real commands, real invariants; delete obvious filler.

**Codex exception:** `/init` in the Codex CLI is an intentional **starting point** - still trim the output to match how the team actually builds, tests, and reviews (per OpenAI's own guidance).

### A2. CLAUDE.md / AGENTS.md over 500 lines

A sign that documentation was dumped where agent guidance belongs. Documentation goes in `docs/`; AGENTS.md keeps only what changes the agent's behavior.

### A3. Code style in AGENTS.md

"Use camelCase for functions" is a linter's job, not an agent's. If the linter is configured, the agent applies it. If it is not, configure the linter instead of writing a rule into AGENTS.md.

### A4. Duplicating commands between AGENTS.md and skills

If `core/AGENTS.md` has `./gradlew :core:build` and `redeploy/SKILL.md` has `cd core && ./gradlew build`, that is drift. The skill must **reference**: "use core/AGENTS.md as source of truth".

### A5. Memory as a replacement for AGENTS.md

"I'll just put it in memory, let it remember" is often laziness. If the knowledge is useful to the team, it belongs in AGENTS.md or a skill, not in personal memory.

### A6. A slash command for every little thing

`/build`, `/test`, `/run` - usually redundant. If `./gradlew build` is documented in AGENTS.md, the agent will run it.

### A7. Hooks on a slow linter

A `PostToolUse` hook running a linter that takes > 2 seconds makes work miserable. Use `--check` in CI.

### A8. Permissions with `Bash(*)`

Approving everything is the opposite of protection. Approval prompts for destructive operations are a feature, not a bug.

### A9. Mixing source of truth and adapters into one file

`CLAUDE.md` that is simultaneously hand-written source and an adapter for another tool is a drift recipe. Clearly separate what is source and what is adapter.

### A10. Global rules specific to one project

`~/.claude/CLAUDE.md` saying "always use Spring Boot" leaks one project's stack into all projects. Global CLAUDE.md is for universal preferences only (response style, broad prohibitions).

### A11. Secrets in tracked agent config

API keys, OAuth client secrets, or long-lived tokens in committed `.claude/settings.json`, `.mcp.json`, or env blocks are a credential leak and a compliance incident. Use gitignored locals, `apiKeyHelper`, or host env - see **section 19** (and section 9 for where it applies in Claude Code).

---

## 15. Recipes by project type

### R1. Single-package monolith (one build, one stack)

**Minimal setup.**

- `AGENTS.md` in the root (~30-50 lines): WHAT/WHY/HOW plus exact build/test commands.
- `CLAUDE.md -> AGENTS.md` (symlink).
- If the linter is automatable: `<repo>/.claude/settings.json` with a `PostToolUse` hook.
- No `.ai/`, no skills, no bootstrap.

### R2. Monorepo with multiple modules (3-8 modules)

- Root `AGENTS.md` with a module map.
- One `<module>/AGENTS.md` per module.
- `CLAUDE.md -> AGENTS.md` symlink in the root.
- If multiple agents are used across the team, consider BYOA with `.ai/`.

### R3. Large monorepo with 2+ agents in the team (BYOA case)

- Full BYOA structure from section 2.
- `tools/ai/bring_your_own_agent_bootstrap.py` with `--write` / `--check`.
- CI sync check.
- `aiimport` and `learn` skills are required.

### R4. Sandbox / personal junk folder

**Minimum minimum.** The goal is to mark the folder as a **sandbox** so the agent stops applying production standards.

- `AGENTS.md` of ~20-30 lines: "WHAT - sandbox. WHY - experiments. HOW - each subfolder is independent, do not refactor, do not suggest tests, do not proliferate files". A **Russian** WHAT/WHY/HOW is fine when the author and docs are Russian-first (P10).
- `CLAUDE.md -> AGENTS.md` symlink.
- No hooks, no skills, no `.ai/`. Overkill.

### R5. Python project

- Exact build/test commands with virtualenv/poetry/uv.
- If `ruff` / `black` / `mypy` are configured, mention them in AGENTS.md (but do **not** duplicate config).
- A `PostToolUse` hook with `ruff check --fix` is justified (fast).

### R6. Go project

- `go.mod` plus `go test ./...`, `go vet`.
- Package conventions are usually obvious from structure - do not duplicate.
- A `gofmt -w` hook is justified.

### R7. JavaScript / TypeScript project

- `package.json` scripts referenced from AGENTS.md.
- An ESLint hook is **dangerous** if type-checking is enabled (slow). Prefer `prettier --write` only.

### R8. Kotlin / Java + Gradle

- Emphasize: "Run Gradle from the repo root only".
- If `ktlint` / `spotless` is configured, name the exact module command: `./gradlew :module:ktlintFormat`.

### R9. SQL / data analytics

- Queries are usually for manual runs, not CI. State this explicitly.
- Do not propose SQL formatting (developers often write SQL for their own reading style).

---

## 16. Claude Code technical details (for the agent that uses it)

### File hierarchy

```
~/.claude/
|-- CLAUDE.md                 # global system context
|-- settings.json             # global settings
|-- commands/                 # global slash commands
|   +-- <name>.md
|-- skills/                   # global skills
|   +-- <name>/
|       +-- SKILL.md
|-- memory/                   # auto-memory (path-encoded)
+-- statusline.sh             # user statusLine

<repo>/
|-- CLAUDE.md -> AGENTS.md    # symlink (or auto-generated)
|-- AGENTS.md                 # source of truth
|-- .mcp.json                 # MCP servers
+-- .claude/
    |-- settings.json         # project settings, committed
    |-- settings.local.json   # local settings, gitignored
    |-- commands/             # project slash commands
    +-- skills/               # project skills (or BYOA wrappers)
```

### Source priority

CLAUDE.md (which AGENTS.md feeds via symlink):

1. `<repo>/CLAUDE.md` - project.
2. `~/.claude/CLAUDE.md` - user.

`settings.json` is merged: `<repo>/.claude/settings.local.json` -> `<repo>/.claude/settings.json` -> `~/.claude/settings.json`. Lists (`allow`, `deny`) are **merged**. Scalars (`theme`, `effortLevel`) - most-local wins.

### Slash command frontmatter

```yaml
---
description: <required - for autocomplete>
argument-hint: [<hint for arguments>]
---
```

`$ARGUMENTS` in the body is substituted with whatever the user typed after the command name.

### Skill frontmatter

```yaml
---
name: <required - kebab-case>
description: <required - contains trigger words>
user_invocable: true | false   # whether the user can invoke as /<name>
---
```

### Hook events (authoritative source)

Claude Code defines **many** hook events; names and schemas change between releases. Use `/hooks` in the product and the official **[Hooks reference](https://docs.claude.com/en/docs/claude-code/hooks)** instead of copying a long table here. Common starting points: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`.

### Permissions syntax (Claude Code)

- `Bash(<cmd>)` - exact match; `Bash(<cmd> *)` - with arguments.
- MCP and other tools - **exact id string per product docs**, not a portable template.
- `Read(<path>)`, `Write(<path>)`, `Edit(<path>)` - file ops where supported.

### StatusLine

```json
{
  "statusLine": {
    "type": "command",
    "command": "/path/to/script.sh"
  }
}
```

The script receives JSON on stdin (fields `model.display_name`, `workspace.current_dir`, `output_style.name`, etc.) and emits a single line on stdout. Useful to display: model, directory, branch, output style.

---

## 17. Cursor

Cursor combines **project rules**, optional **team rules** (Team / Enterprise), **user rules**, and **AGENTS.md**. Rough precedence: team policies override project defaults; project overrides user for shared repo behavior. Confirm in [Rules](https://cursor.com/docs/rules) for your plan and version.

### AGENTS.md

Cursor reads root `AGENTS.md` as project context for the agent (same open format as Codex and others). Prefer keeping it short and linking to `docs/` for depth (see P2, P3).

### Project rules: `.cursor/rules/*.mdc`

- Use **`.mdc`** files with YAML **frontmatter** for metadata: `description`, `globs`, `alwaysApply`, and rule activation mode (always vs intelligent vs path-scoped vs manual `@rule` - see docs).
- **Split by topic** and use **globs** so only relevant rules load for a given file. Cursor documents a **~500 line** soft ceiling per rule file; split rather than grow one mega-file.
- **`alwaysApply: true`** puts the rule in every request - use sparingly (high signal only). Prefer globs or agent-selected rules for large repos.
- Legacy **`.cursorrules`** may still work but treat it as deprecated; migrate to `.cursor/rules/` or `AGENTS.md`.

### Commands (slash recipes)

Team-shared prompt shortcuts live under **`.cursor/commands/`** as Markdown (similar spirit to Claude Code's `.claude/commands/`). Check in git so everyone shares the same workflows.

### Plan mode and saved plans

- **Plan Mode** (toggle in the agent input, e.g. **Shift+Tab**): the agent researches the codebase, asks clarifying questions, and proposes a plan before large edits.
- **Save to workspace** stores plans under **`.cursor/plans/`** - useful for resuming work and for the next agent session on the same feature.

### Context hygiene (from Cursor's agent guidance)

- **Let the agent search** - do not paste the whole repo; tag files only when you know exactly which matter.
- **New chat** when switching features or when quality drops after many turns (long threads accumulate noise).
- **`@Past Chats`** to pull selective prior context instead of duplicating full transcripts.

### Skills and hooks (Cursor-native)

Cursor is adding **Agent Skills** (`SKILL.md`, optional hooks under **`.cursor/hooks.json`**) for dynamic workflows. Capabilities and file locations evolve; see [Skills](https://cursor.com/docs/context/skills) and [Agent hooks](https://cursor.com/docs/agent/hooks). Some skill features have shipped behind **Nightly** - verify channel in settings before documenting "required" team workflows on them.

### Parallelism

Cursor supports **git worktrees** and **multi-model** runs for hard tasks. If you standardize on parallel agents, document branch / worktree conventions in `AGENTS.md` so automated runs do not collide.

---

## 18. OpenAI Codex CLI

Codex reads **`AGENTS.md` automatically** as durable repo guidance. Official guide: [AGENTS.md in Codex](https://developers.openai.com/codex/guides/agents-md).

### Nesting and personal defaults

- **Repo root** `AGENTS.md` for team standards.
- **Nested** `AGENTS.md` in subdirectories for package-local rules - **more specific path wins** over parent (same mental model as https://agents.md).
- **User global** `AGENTS.md` under **`~/.codex`** for personal defaults that apply across repos (per Codex docs).

### Configuration layers

- **`~/.codex/config.toml`** - personal defaults (model, reasoning, sandbox, MCP, profiles).
- **`.codex/config.toml`** in repo - shared team behavior.
- CLI flags for one-off overrides.

Codex documents **approval mode** and **sandbox mode** separately (what runs vs what touches the filesystem). Start strict; widen only for trusted repos or known workflows.

### Scaffolding vs curation

The Codex CLI **`/init`** command scaffolds a starter `AGENTS.md`. Treat that as a **first draft** you trim to real commands and invariants - same anti-noise goal as section 14 (auto-generated walls of text). Prefer a short accurate file over a long generic one.

### Session management (CLI)

Use thread controls when work gets long: e.g. **`/compact`**, **`/fork`**, **`/resume`**, **`/agent`** for parallel threads - see [slash commands](https://developers.openai.com/codex/cli/slash-commands). Aligns with "one thread per coherent task" from Codex best practices.

### Skills (Codex)

Team skills can live under **`.agents/skills`** in the repo; personal skills under **`$HOME/.agents/skills`**. Use the **`$skill-creator`** flow in Codex docs to bootstrap. Same principle as elsewhere: one job per skill, description with **trigger phrases**.

---

## 19. Security for AI agent setups

Canonical policy for **secrets**, **MCP scope**, and **safe sharing**. Sections **9-10** implement the same rules inside permission and MCP config.

- **No long-lived secrets** in committed agent files (`.claude/settings.json`, `.mcp.json`, `AGENTS.md`, rules, skills, tracked scripts). Use gitignored locals, host env, or vendor **short-lived token** helpers (e.g. Claude Code `apiKeyHelper`). **Rotate** anything pasted into chat, tickets, or screenshots.
- **MCP and hooks run code** - pin trusted servers/commands; prefer **read** auto-approval over **write**; document publish rules in `AGENTS.md`.
- **Chats and cloud sessions** may be retained - no production secrets or unsanitized PII unless policy allows.
- **Subprocess hygiene:** hooks inherit env; avoid secrets on argv; prefer narrow Bash allows (A8).

### Optional one-liner for root `AGENTS.md`

`Security: no secrets in repo; automation is read-first; publishing to trackers/reviews/VCS only when explicitly requested - see <org policy URL>.`

---

## 20. Glossary

- **AGENTS.md** - open standard markdown file containing context for AI agents. See https://agents.md.
- **CLAUDE.md** - Claude Code's format, content-equivalent to AGENTS.md.
- **BYOA** (Bring Your Own Agent) - pattern in which `.ai/` is the source of truth and native agent files are auto-generated.
- **Skill** - reusable procedure with its own tools, defined by a SKILL.md.
- **Slash command** - prompt template invoked as `/<name>`.
- **Hook** - external command run by the agent harness in response to an event.
- **MCP** (Model Context Protocol) - standard for connecting external tool servers to agents.
- **Adapter** - agent file auto-generated from a source-of-truth template.
- **Memory** - local-only, non-committed knowledge layer for the agent.
- **Bootstrap** - script that syncs adapters with templates.
- **Forge** - your org's code hosting and review system (GitHub, GitLab, self-hosted, etc.).

---

## 21. References

Primary specs (bookmark upstream docs; they change faster than this playbook):

- Model Context Protocol: https://modelcontextprotocol.io/introduction
- AGENTS.md format and FAQ: https://agents.md/
- Claude Code overview: https://docs.claude.com/en/docs/claude-code/overview
- Claude Code hooks: https://docs.claude.com/en/docs/claude-code/hooks (quickstart: https://docs.claude.com/en/hooks-guide)
- Cursor agent best practices: https://cursor.com/blog/agent-best-practices
- Cursor Rules: https://cursor.com/docs/rules
- Cursor Agent Skills: https://cursor.com/docs/context/skills
- Cursor Agent hooks: https://cursor.com/docs/agent/hooks
- OpenAI Codex hub: https://developers.openai.com/codex/
- OpenAI Codex best practices: https://developers.openai.com/codex/learn/best-practices
- OpenAI Codex AGENTS.md guide: https://developers.openai.com/codex/guides/agents-md
- Example AGENTS.md repos: https://github.com/search?q=path%3AAGENTS.md+NOT+is%3Afork&type=code (filter before copying)

Further reading: [Habr / HumanLayer on CLAUDE.md](https://habr.com/ru/articles/972308/) (also in **Sources** above); maintain your own BYOA template repo - patterns in sections 2 and 12.

---

## 22. Versioning this playbook

This document is a living one. When you find a new practice or antipattern, add it to the relevant section. When something becomes outdated (e.g. Cursor changes its rules format), update it with a date stamp.

Current revision reflects practices as of **May 2026** (primary links in **References** spot-checked). **Section 19 (Security)** is the canonical security policy. **Paired file:** `ai-agent-setup-playbook.md` (Yandex Arcadia / Arcanum / internal BYOA specifics).
