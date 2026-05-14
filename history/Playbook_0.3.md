# AI Agent Configuration Playbook

Vendor-neutral reference and bootstrap guide for configuring AI coding agents across projects and teams.

Current revision: May 2026.

This document is designed to be:
- portable across tools and vendors;
- compact enough for both humans and models;
- operational rather than theoretical;
- safe to share publicly;
- maintainable over time.

Supported ecosystems include:
- Claude Code
- Cursor
- OpenAI Codex
- AGENTS.md-compatible tools
- MCP-based tooling
- custom agent runtimes

The principles are tool-agnostic unless explicitly noted otherwise.

---

# 0. Bootstrap protocol for the reading AI

If you are an AI model reading this document in order to configure a new project, follow this sequence:

1. Analyze the repository structure, stack, build system, and existing workflows.
2. Detect existing agent configuration files (`AGENTS.md`, `CLAUDE.md`, `.cursor/`, `.github/copilot-instructions.md`, `.windsurfrules`, `.mcp.json`, etc.).
3. Create or update the canonical `AGENTS.md` (or `.ai/AGENTS.md`) using the principles from this playbook.
4. Generate tool-specific adapters if needed.
5. Configure indexing hygiene (`.cursorignore`, `.aignore`, generated-file exclusions).
6. Add exact build, test, lint, and verification commands.
7. Document only non-obvious architectural invariants and operational constraints.
8. Report the generated setup to the user and request any missing project-specific invariants.

Do not generate giant boilerplate rule files.

---

# 1. Core philosophy

AI-agent configuration is not primarily prompt engineering.

It is:
- information architecture;
- retrieval architecture;
- context management;
- workflow orchestration;
- verification design;
- permission design;
- operational boundary management.

The goal is not to maximize instructions.

The goal is to maximize:
- signal quality;
- maintainability;
- reproducibility;
- retrieval precision;
- operational safety.

---

# 2. Metaprinciples

## P1. Single Source of Truth

All durable agent guidance should live in one canonical place.

Prefer:
- `AGENTS.md`
- or `.ai/AGENTS.md`

Tool-specific files (`CLAUDE.md`, `.cursorrules`, Copilot instructions, etc.) should be generated adapters or thin wrappers.

Never maintain duplicated rule sets manually.

---

## P2. Minimum context, maximum signal

Do not teach the agent general programming knowledge.

Modern coding agents already contain extensive default coding guidance in system prompts and runtime policies.

Avoid:
- style-guide dumps;
- repeated linter rules;
- framework tutorials;
- obvious repository descriptions;
- repeated system-prompt advice;
- giant architectural walls of text.

Only include:
- non-obvious invariants;
- repository-specific constraints;
- operational rules;
- exact commands;
- workflow boundaries.

Good heuristic:
- root `AGENTS.md` under 100-200 lines.

Smaller high-signal context usually outperforms large static context.

---

## P3. Progressive disclosure

Do not preload all documentation into agent context.

Prefer:
- references;
- routing;
- targeted retrieval;
- scoped instructions.

Example:

Bad:

```md
Paste the entire testing handbook into AGENTS.md
```

Good:

```md
Testing strategy: docs/testing.md
```

The agent should retrieve details only when relevant.

---

## P4. Context economics

Context is a limited resource.

Every always-loaded instruction competes with:
- user requests;
- retrieved code;
- tool outputs;
- planning state;
- execution history.

Prefer:
- retrieval over preload;
- scoped rules over global rules;
- references over duplication;
- task-local instructions over universal constraints.

---

## P5. ASCII-first text

Prefer ASCII in:
- configs;
- code;
- AGENTS.md;
- slash commands;
- generated prompts.

Avoid:
- smart quotes;
- em/en dashes;
- unicode arrows;
- unicode ellipsis;
- emoji;
- invisible unicode characters.

Reason:
- easier grep/search;
- easier typing;
- safer shell usage;
- cleaner diffs;
- fewer encoding/tooling issues.

Exception: use non-ASCII only when it is semantically necessary.

---

## P6. Structured formatting

Clear structure improves retrieval and instruction segmentation.

Structured formatting improves retrieval, segmentation, and instruction parsing.

Useful formats include:
- Markdown headings;
- fenced sections;
- YAML frontmatter;
- XML-style delimiters.

Example:

```xml
<architectural_rules>
  <rule>All service communication uses gRPC.</rule>
</architectural_rules>
```

Use whichever format the team can maintain consistently.

---

## P7. Knowledge classification

Classify knowledge by scope.

| Knowledge type | Location |
|---|---|
| User-global preference | `~/.claude/CLAUDE.md` or equivalent |
| Project-wide invariant | `<repo>/AGENTS.md` |
| Module-specific rule | `<module>/AGENTS.md` |
| Reusable workflow | skill / command |
| Local machine facts | gitignored memory |
| Secrets | secret manager / environment |

If knowledge is useful to the entire team, memory is usually the wrong place.

---

## P8. Self-updating context

After meaningful work, agents should propose updates when:
- a non-obvious pitfall was discovered;
- the user corrected behavior;
- hidden infrastructure details surfaced;
- repeated friction emerged.

Do not update documentation after every task.

Prefer:
- compactness;
- precision;
- future usefulness.

One precise bullet usually beats a paragraph of narrative.

---

# 3. Verification-first workflow

Do not trust code generation blindly.

Preferred execution loop:

1. Analyze existing implementation
2. Form hypothesis
3. Propose minimal plan
4. Implement atomic changes
5. Verify using tools
6. Summarize evidence

Verification is mandatory for:
- refactors;
- migrations;
- concurrency changes;
- infrastructure automation;
- public API modifications.

Do not report success without verification artifacts.

Formatting, naming, and style enforcement should primarily come from:
- linters;
- formatters;
- typecheckers;
- CI validation.

Avoid pushing style micromanagement into AGENTS.md.

---

# 4. Context and indexing hygiene

Modern coding agents rely heavily on retrieval and indexing.

If indexing is polluted, retrieval quality degrades.

Always maintain ignore rules such as:
- `.cursorignore`
- `.aignore`
- repository-specific indexing exclusions

Exclude aggressively:
- `node_modules/`
- `dist/`
- `build/`
- `.git/`
- generated assets
- logs
- minified files
- large lockfiles if unnecessary for retrieval

Prefer targeted retrieval over whole-repository scans.

---

# 5. Thread hygiene

Long noisy sessions reduce reliability.

Start a new session when:
- the task changes substantially;
- exploratory noise accumulates;
- multiple abandoned approaches exist;
- the agent repeats failed strategies;
- context becomes incoherent.

Prefer concise summaries over continuing extremely long transcripts.

Avoid combining:
- exploration;
- implementation;
- debugging;
- review;
- deployment;
inside one uncontrolled thread.

---

# 6. Runtime topology

Complex tasks work better when responsibilities are separated.

Common runtime roles:
- orchestrator agent;
- research subagent;
- implementation subagent;
- verification/review agent;
- deployment agent.

Useful patterns:
- isolated worktrees;
- bounded contexts;
- parallel branches;
- scoped retrieval;
- review-only agents.

Avoid overloading one long-lived context with every responsibility simultaneously.

---

# 7. Diff discipline

Prefer:
- small atomic changes;
- bounded scope;
- reviewable diffs;
- incremental refactors.

Avoid:
- giant multi-concern diffs;
- mixing formatting with logic;
- unnecessary rewrites;
- ghost refactoring.

Ghost refactoring = rewriting adjacent code unrelated to the task.

Preserve unrelated code and surrounding architecture unless modification is required for correctness, compatibility, or maintainability directly tied to the requested change.

Do not modify nearby code unless required for correctness or maintainability directly related to the requested work.

---

# 8. AGENTS.md design

The root `AGENTS.md` should answer:

1. WHAT is this project?
2. WHY does it exist?
3. HOW should agents work with it?

Recommended contents:
- architectural invariants;
- prohibitions with rationale;
- exact build/test/lint commands;
- deployment constraints;
- routing to submodules;
- external API constraints;
- operational pitfalls.

Avoid:
- style-guide duplication;
- giant tutorials;
- obvious descriptions;
- dependency dumps.

Recommended structure:

```md
# Project

## WHAT

## WHY

## HOW

## Hierarchy

## Details
```

Example:

```xml
<project_context>
  <what>Backend analytics platform.</what>
  <why>Processes high-volume events.</why>
</project_context>

<workflow_commands>
  <command purpose="test">make test</command>
  <command purpose="lint">make lint</command>
</workflow_commands>
```

---

# 9. Hierarchical configuration

Use hierarchical `AGENTS.md` files for:
- monorepos;
- independent modules;
- mixed stacks;
- different workflows.

Example:

```txt
root/AGENTS.md
services/auth/AGENTS.md
frontend/AGENTS.md
```

Guidelines:
- root file contains global invariants;
- child files contain only local specifics;
- avoid duplication;
- keep hierarchy shallow.

Most tools prioritize the closest relevant `AGENTS.md`, though exact precedence differs by implementation.

---

# 10. BYOA (Bring Your Own Agent)

Purpose:
- one canonical source;
- many agent runtimes;
- minimal duplication.

Recommended structure:

```txt
.ai/
  AGENTS.md
  skills/
  bootstrap/
  memory/

CLAUDE.md -> generated or symlinked
.cursor/rules/ -> generated
```

Use full BYOA only when complexity justifies it.

Good candidates:
- multi-agent teams;
- large monorepos;
- reusable skills;
- CI validation.

Avoid overengineering tiny repositories.

The setup complexity should scale with repository complexity.

---

# 11. Skills and slash commands

## Skills

Skills are reusable workflows.

Difference:
- `AGENTS.md` = context;
- skill = procedure.

Extract a skill when:
- the workflow repeats;
- multiple steps exist;
- permissions matter;
- tool orchestration is required.

Minimal structure:

```txt
.ai/skills/<name>/SKILL.md
```

Recommended sections:
- Goal
- Parameters
- Required context
- Steps
- Output contracts

Avoid duplicating AGENTS.md inside skills.

Skills should reference canonical project rules instead of re-declaring them.

---

## Slash commands

Slash commands are reusable prompt shortcuts.

Good use cases:
- explain code;
- summarize diff;
- review changes;
- extract learnings.

Avoid:
- trivial wrappers around obvious commands;
- one-off prompts;
- replacing proper skills.

---

# 12. Memory

Memory is local and non-committed.

If knowledge is useful to the whole team, memory is usually the wrong place.

Use memory for:
- machine-local paths;
- temporary environment details;
- user-specific workflows;
- structured operational metadata.

Do not store:
- secrets;
- long-lived credentials;
- reusable team guidance.

Prefer structured schemas for automation-related memory.

---

# 13. Hooks and automation

Hooks execute logic around runtime events.

Hooks are for behavior that cannot be expressed cleanly through:
- static instructions;
- memory;
- normal workflows;
- permissions.

Avoid turning hooks into hidden orchestration layers.

Typical uses:
- formatting;
- lightweight linting;
- notifications;
- telemetry;
- safety checks.

Avoid:
- slow hooks;
- hidden heavy automation;
- stack-wide hooks in heterogeneous repositories.

Hooks should remain:
- predictable;
- observable;
- lightweight.

---

# 14. Permissions and MCP

Permissions are a security boundary.

Do not treat them as a way to suppress prompts.

Good candidates for auto-approval:
- read-only commands;
- harmless filesystem inspection;
- reversible formatting.

Avoid auto-approving:
- destructive shell operations;
- production deployment;
- unrestricted Bash;
- credential operations.

Bad:

```json
"Bash(*)"
```

Good:

```json
"Bash(git status *)"
```

---

## MCP

MCP connects agents to external systems.

Examples:
- issue trackers;
- repositories;
- internal search;
- deployment systems;
- databases.

Guidelines:
- commit only non-secret configuration;
- default to read-first integrations;
- separate read vs write capabilities;
- prefer read-only defaults;
- require explicit approval for mutating operations.

---

# 15. Capability isolation

Do not expose broad production capabilities to general-purpose coding agents by default.

Avoid unrestricted access to:
- production shells;
- cloud admin credentials;
- deployment systems;
- infrastructure mutation tooling.

Prefer:
- dedicated deployment skills;
- constrained scopes;
- explicit approval;
- isolated runtime identities.

---

# 16. Observability

For autonomous or long-running workflows, capture:
- executed commands;
- changed files;
- retries;
- tool failures;
- verification outputs;
- permission escalations.

Without observability, debugging agent failures becomes difficult.

Prefer:
- append-only logs;
- structured outputs;
- explicit verification summaries.

---

# 17. Antipatterns

## A1. Giant rulesets

Large low-signal files dilute attention.

---

## A2. Auto-generated noise

Uncurated `/init` dumps often hurt more than help.

Curate aggressively.

---

## A3. Repeating system prompts

Do not waste context on:
- “You are an expert programmer”;
- “Think step by step”;
- generic coding advice.

Modern coding tools already inject these behaviors.

---

## A4. Workflow duplication

Do not duplicate commands across:
- AGENTS.md;
- README;
- skills;
- adapters.

---

## A5. Memory abuse

Reusable team knowledge does not belong in personal memory.

---

## A6. Slow hooks

Heavy hooks destroy UX.

---

## A7. Overbroad permissions

`Bash(*)` effectively disables the safety model.

---

## A8. Retrieval pollution

Do not preload massive documentation blobs instead of using retrieval.

---

## A9. Infinite-thread syndrome

Very long uncontrolled sessions often degrade quality.

Prefer clean session boundaries and concise handoff summaries.

---

# 18. Project setup checklist

## Stage 1. Analyze

- Inspect repository structure
- Detect stacks and build systems
- Detect existing agent configs
- Identify workflows and constraints

## Stage 2. Initialize

- Create canonical `AGENTS.md`
- Configure hierarchy if needed
- Add exact commands

## Stage 3. Configure retrieval

- Create `.cursorignore` / `.aignore`
- Exclude generated artifacts
- Reduce retrieval pollution

## Stage 4. Generate adapters

- Claude Code
- Cursor
- Copilot
- other team tools

## Stage 5. Add workflows

- Extract reusable skills
- Configure hooks carefully
- Add verification workflows

## Stage 6. Secure the setup

- Configure permissions
- Configure MCP
- Verify no secrets are tracked
- Validate capability boundaries

## Stage 7. Review

- Remove duplication
- Remove obvious filler
- Verify compactness
- Verify operational clarity

---

# 19. Tool-specific notes

## Claude Code

Typically uses:
- `CLAUDE.md`
- hooks
- slash commands
- skills
- permissions

Prefer symlink or generated adapter from `AGENTS.md`.

---

## Cursor

Supports:
- `AGENTS.md`
- rules
- plans
- hooks
- retrieval indexing

Prefer:
- compact scoped rules;
- strong indexing hygiene;
- minimal always-loaded context.

---

## OpenAI Codex

Supports:
- nested `AGENTS.md`;
- repository-level guidance;
- user-global guidance.

Keep instructions concise and operational.

---

# 20. Security policy

Core rules:
- no secrets in tracked config;
- no unrestricted production access;
- automation should default to read-first;
- publishing should require explicit intent;
- hooks and MCP are executable code.

Chat systems may retain transcripts.

Never paste:
- production credentials;
- sensitive customer data;
- internal tokens;
- infrastructure secrets.

Rotate any secret accidentally exposed.

---

# 21. References

AGENTS.md
https://agents.md/

Model Context Protocol
https://modelcontextprotocol.io/

Claude Code docs
https://docs.claude.com/

Cursor docs
https://cursor.com/docs

OpenAI Codex docs
https://developers.openai.com/codex/

---

# 22. Final guidance

The best AI-agent setups are:
- compact;
- explicit;
- verifiable;
- observable;
- minimally duplicated;
- operationally safe.

Good agent systems are designed like good software systems:
- clear interfaces;
- bounded responsibility;
- low coupling;
- reproducible behavior;
- controlled capabilities.

Treat agent configuration as engineering infrastructure, not prompt decoration.
