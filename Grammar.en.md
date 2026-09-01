# Two Grammars in One Evening

## Prompts and skills: everything you need to get started

[Русская версия](Grammar.md)

Evidence review: September 2026.

> *“A prompt is a conversation with a colleague. A skill is a job description for an employee you may never meet.”*

The original framework and research are by **Yuri Zelenkov**, Doctor of Technical Sciences and a computational linguist who has worked at Yandex for more than 20 years. This public edition was compiled and edited by **Meliksetian Narek**. Published in [Econet](https://t.me/econets) with the research author’s permission.

This edition preserves the original concept and conclusions while distinguishing the author’s heuristics from independently verified guidance. Numerical estimates from the source materials are collected in the [appendix](#appendix-original-quantitative-estimates): the methodology and raw data are unavailable, so the values should not be treated as universal or reproduced results.

---

This guide distills two grammars:

- **prompt grammar** — how to formulate a one-time request;
- **skill grammar** — how to describe a repeatable agent workflow.

After reading it, you will be able to:

- review a prompt using seven possible blocks;
- identify lexical ambiguity, presuppositions, and implicatures;
- turn a recurring request into a compact, verifiable skill;
- distinguish real constraints from unnecessary verbosity.

---

# Part 1. Prompt grammar

## One idea: a prompt is not “just text”

A prompt is an instruction with a goal, a structure, and a space of possible interpretations. Just as a sentence has syntactic positions, a prompt can be decomposed into **seven functional blocks**.

This is not a mandatory form for every request. It is a checklist: include a block when it resolves a real ambiguity or establishes an important boundary.

## The 7 blocks of a good prompt

```text
Underspecified prompt:
  “Write a parser.”
  → A parser for what? In which language? What should it return?

Prompt with a complete contract:
  Role:         Senior Python Developer
  Context:      Python 3.12, stdlib, pytest
  Task:         count_words(text: str) -> dict[str, int]
  Constraints:  No Counter or regex; Unicode-aware
  Format:       two blocks: implementation and tests
  Example:      count_words("cat and cat") -> {"cat": 2, "and": 1}
  Validation:   empty string -> {}; punctuation is not a word
```

| Block | Question | Especially useful when |
|---|---|---|
| Role | Who is answering? | Expertise, perspective, or tone changes the result. |
| Context | In what environment? | The answer depends on a stack, version, data, or existing solution. |
| Task | What observable state should change? | Almost always; describe the outcome rather than an activity. |
| Constraints | Which options are unacceptable? | Real technical or operational boundaries exist. |
| Format | What does a usable answer look like? | A program consumes the result or a short structure is required. |
| Example | What does a correct result resemble? | A boundary or style is difficult to specify in prose. |
| Validation | What proves correctness? | A test, query, schema, or checklist can verify the result. |

A short request may need only a task and a format. A complex engineering task may need all seven blocks. Quality depends on a sufficient contract, not on the number of headings.

## 3 prompt traps

### Trap 1: lexical ambiguity

The original work called this trap homonymy. The broader term is more precise: it includes both homonymy and polysemy.

```text
Weak: “Create a service.”

“Service” may mean:
  - a Python OrderService class;
  - a payments microservice;
  - a systemd unit;
  - a gRPC service.

Better: “Create an OrderService class with create_order().”
```

The model will choose a plausible interpretation, but not necessarily yours. A concrete term narrows the answer space.

### Trap 2: presupposition — a hidden assumption

```text
Weak: “Why is my code slow?”

Hidden assumption: the code is actually slow.
The model may start explaining a cause before checking the premise.

Better: “Is there a measurable performance problem here?
If so, show the evidence and the likely cause.”
```

Check the premise first; ask for an explanation second.

### Trap 3: implicature — an implied requirement

```text
Weak: “Write production-ready code.”

You may imply tests, logging, type safety, error handling,
metrics, and documentation. The model does not know which
parts are mandatory in this task.

Better: “Add error handling, type hints, structured logging,
and five pytest tests for the listed edge cases.”
```

**Author’s rule:** explicit is stronger than implied.

**Editorial qualification:** make only outcome-relevant requirements explicit. Excessive and conflicting rules also reduce reliability.

## How constraints narrow the solution space

```text
“Implement a sort.”
+ “Do not use sorted().”
+ “Do not use recursion.”
+ “Required complexity: O(n log n).”
```

Each real constraint removes a class of unsuitable solutions. A prohibition alone, however, rarely says what to do instead.

Use a pair:

```text
Constraint: do not mutate the input list.
Alternative: return a new list.
```

Reserve absolute `NEVER` and `ALWAYS` language for true invariants. Explicit `if/then` rules are better for conditional behavior.

## Output format affects cost and reliability

The cost of a call depends on the model, pricing tier, input/output volume, caching, and retries:

```text
cost = input_tokens × input_rate
     + cached_input_tokens × cached_rate
     + output_tokens × output_rate
     + tool/runtime costs
```

Output is often more expensive than input, but there is no fixed ratio across all models. JSON does not guarantee savings either: keys, quotes, and repeated structure can be longer than a short natural-language answer.

Choose the **smallest usable format**:

- one value when the consumer needs one value;
- a short table or list for a person;
- JSON/schema for machine processing;
- code or a patch when the artifact must be executable;
- an explanation only when it is part of the result.

If the API supports structured outputs or schema validation, prefer that technical guarantee to a textual request to “respond with strict JSON.”

## PQS — the author’s prompt-quality heuristic

The original work proposed **PQS** as a preflight indicator from 0 to 1 — a kind of linter for prompts.

The idea is useful: before sending a prompt, check whether its task, context, constraints, format, and validation are sufficiently precise. However, no formula, weights, corpus, or data connecting PQS to retry rate have been published. In this edition, PQS is therefore **the name of the author’s checklist, not a validated numerical metric**.

A practical version:

```text
Before sending, ask:
  1. Is the expected outcome unambiguous?
  2. Does the prompt include context that actually changes the answer?
  3. Are real constraints and acceptable alternatives named?
  4. Is the smallest usable output format clear?
  5. Can the result be verified?
```

If you want a numerical PQS, first define its formula and calibrate it on your team’s real tasks.

## 5 efficiency levers

| Lever | Potential benefit | Limitation |
|---|---|---|
| Minimal output format | Less output and easier validation. | JSON is not always shorter. |
| Remove ambiguity | Fewer incorrect interpretations and retries. | Do not explain what is already obvious. |
| Real constraints | Eliminate unacceptable solutions. | A negative rule needs a safe alternative. |
| Runtime controls | May change variance, latency, and price. | Parameters such as temperature depend on the model and API. |
| A short decisive example | Fixes a boundary and may reduce phrasing sensitivity. | A poor or unnecessary example adds noise. |

The largest saving usually comes not from one trick but from shortening the chain `ambiguity → wrong answer → correction → retry`.

## Prompt grammar: summary

```text
7 POSSIBLE BLOCKS
role → context → task → constraints → format → example → validation

3 TRAPS
lexical ambiguity → name the exact object
presupposition → verify the assumption first
implicature → make the important requirement explicit

THE CORE
the smallest sufficient contract
real constraints + the desired alternative
the smallest usable format
a verifiable result
```

---

# Part 2. Skill grammar

## One idea: a skill is not a prompt

```text
Prompt: you see a specific response and can immediately clarify the request.
Skill: a repeatable instruction that may activate in another context.
```

Reuse scales both benefits and defects. An unclear trigger invokes the skill at the wrong time; an ambiguous step produces different actions; an unsafe default repeats without the author present.

A skill therefore deserves stricter treatment than a one-time prompt: define routing, inputs, output, validation, and side-effect boundaries.

## Portable minimum

The open Agent Skills specification requires YAML frontmatter with `name` and `description`. The rest of the structure depends on the task.

```yaml
---
name: review-alert
description: Review Prometheus alert safety and completeness. Use for alert-rule changes; do not use for dashboard-only work.
---
```

The `description` is a routing contract: what the skill does, when it activates, and, when useful, when it must not activate.

## 8 skill sections — the author’s extended template

For a complex workflow, the original work proposes eight sections:

| Section | Function |
|---|---|
| IDENTITY | Role and area of application. |
| CONTEXT | Stack, versions, paths, and inputs. |
| ANTI-PATTERNS | Critical prohibitions and safe alternatives. |
| CAPABILITIES | Concrete supported actions. |
| WORKFLOW | Execution algorithm. |
| OUTPUT FORMAT | Result contract. |
| EXAMPLES | Minimal examples or links to large ones. |
| VALIDATION | Verifiable completion criteria. |

This is a **template, not a standard**. A simple skill does not need eight empty headings. Include a section when it affects routing, execution, or verification.

## Section order

Hiding important information in the middle of a long context is risky: research has found positional effects, although they depend on the model and task.

A practical rule:

- make trigger and scope visible in metadata;
- state dangerous invariants before the corresponding action;
- put workflow steps in execution order;
- keep validation close to the Definition of Done;
- move rare details into external references.

A fixed order of eight sections is not a technical guarantee. Test it with the target agent.

## Each section in 30 seconds

### IDENTITY — who and when

```text
Verbose:
“You are a world-class expert with exceptionally deep knowledge...”

Concrete:
“Database migration specialist for the payments backend.
Use for schema changes and data backfills.”
```

A role is useful when it changes the solution. A trigger is always useful when a skill can activate automatically.

### CONTEXT — where it operates

```text
Py 3.12, FastAPI, SQLAlchemy 2 async, PostgreSQL 16.
Migrations: migrations/; tests: tests/migrations/.
```

Use abbreviations your team understands. Do not sacrifice clarity to save a few tokens.

### ANTI-PATTERNS — critical boundaries

```text
NEVER apply a production migration without explicit approval.
Instead, generate the migration, validate it, and report the exact apply command.
```

Modality:

- `NEVER` / `ALWAYS` — only for an absolute invariant;
- `MUST` — a required criterion or step;
- `SHOULD` — a preference with acceptable exceptions;
- `if/then` — a conditional decision.

“Preferably” is not a forbidden word, but it is too weak for a critical boundary.

### CAPABILITIES — what it can do

```text
1. Create or modify Prometheus alert rules.
2. Review alert safety and completeness.
3. Validate rules with promtool.
```

Name observable actions, not broad areas such as “help with monitoring.”

### WORKFLOW — how it works

```text
1. Inspect existing rules for the service.
2. Create or update YAML using OUTPUT FORMAT.
3. Add or verify the runbook reference.
4. Run `promtool check rules <file>`.
5. Report the diff and validation result.
```

Each step starts with an action and ends with an observable state.

### OUTPUT FORMAT — the result contract

```json
{"issues":[{"line":42,"severity":"warning","fix":"Add a runbook reference"}]}
```

A template is especially useful for machine processing. A Markdown list may be shorter for a person. Do not require JSON by habit.

### EXAMPLES — inline or referenced

One short inline example is appropriate when it fixes a decisive boundary. Move large or rare examples to a file:

```text
For a multi-window burn-rate alert, read examples/burn-rate-alert.yaml.
```

Say **when** to load the reference. A link without a condition does not guarantee progressive disclosure.

### VALIDATION — a verifiable finish

```text
- [ ] Runbook reference exists.
- [ ] `for` is at least 5m, unless the repository documents an exception.
- [ ] `promtool check rules <file>` passes.
```

Validation should use a test, tool, schema, or explicit criterion. Asking the model to “double-check itself” is task-dependent and does not replace external evidence.

## Worked example: a complete skill

````markdown
---
name: manage-alerts
description: Create or review Prometheus alert rules. Use for alert YAML changes; do not use for dashboard-only tasks.
---

# Manage alerts

## IDENTITY
Observability engineer for this repository.

## CONTEXT
Prometheus 2.x and Alertmanager. Rules: `configs/alerts/{service}/`.

## ANTI-PATTERNS
NEVER delete or deploy alerts without explicit approval.
NEVER invent a runbook URL; use an existing repository path or report it missing.
NEVER hardcode an unexplained threshold.

## CAPABILITIES
1. Create or modify Prometheus alert rules.
2. Review rule safety and completeness.

## WORKFLOW
1. Inspect existing service rules and local instructions.
2. Create or update YAML using OUTPUT FORMAT.
3. Add a verified runbook path.
4. Run `promtool check rules <file>`.
5. Report the diff, evidence, and anything not verified.

## OUTPUT FORMAT
```yaml
groups:
  - name: {service}_{metric}
    rules:
      - alert: {Service}{Condition}
        expr: {promql}
        for: 5m
        labels:
          severity: critical|warning
        annotations:
          runbook: docs/runbooks/{alert}.md
```

## EXAMPLES
For a multi-window burn-rate alert, read `examples/burn-rate-alert.yaml`.

## VALIDATION
- [ ] Runbook path exists.
- [ ] Threshold has a documented rationale.
- [ ] `promtool check rules <file>` passes.
````

This example intentionally contains all eight sections because the workflow changes configuration and may have side effects. A simpler task can omit some sections.

## 6 techniques for reducing skill context

| Technique | Benefit | Risk |
|---|---|---|
| The team’s working language | Fewer comprehension and maintenance errors. | English often tokenizes shorter, but the ratio depends on the model. |
| Remove filler | Higher signal density. | Over-compression creates ambiguous abbreviations. |
| References for large examples | Rare details are not always loaded. | A runtime may not fetch a link automatically; routing is required. |
| Modularity | Local instructions activate when needed. | Excessive fragmentation makes discovery harder. |
| Templates and schemas | Easier parsing and validation. | A schema can be longer than a human-facing answer. |
| Deterministic scripts | More reliable repeatable transformation and validation. | Scripts require tests, security, and maintenance. |

Do not optimize input tokens alone. Measure the complete loop: input, output, tools, retries, latency, and human corrections.

## Skill grammar: summary

```text
PORTABLE MINIMUM
name + description with a precise trigger

THE AUTHOR’S TEMPLATE FOR A COMPLEX WORKFLOW
IDENTITY → CONTEXT → ANTI-PATTERNS → CAPABILITIES
→ WORKFLOW → OUTPUT FORMAT → EXAMPLES → VALIDATION

THE CORE
one repeatable job
explicit inputs and output
safe side-effect boundaries
a verifiable result
realistic evals for trigger and output
```

---

# Part 3. Connecting the two grammars

## 7 prompt blocks → 8 possible skill sections

| Prompt | Skill | What reuse adds |
|---|---|---|
| Role | IDENTITY + metadata | Trigger and area of application. |
| Context | CONTEXT | Stable paths, versions, and inputs. |
| Task | CAPABILITIES + WORKFLOW | A separation between “what” and “how.” |
| Constraints | ANTI-PATTERNS | Invariants, approvals, and safe alternatives. |
| Format | OUTPUT FORMAT | A stable result contract. |
| Example | EXAMPLES | Routing to large examples. |
| Validation | VALIDATION | Definition of Done and external checks. |

When a prompt becomes a skill, every important element becomes not necessarily stricter, but more **operational**:

- implied context becomes explicit;
- one-time context becomes a trigger and scope;
- a request becomes a repeatable workflow;
- a preference is separated from an invariant;
- an answer becomes a verifiable artifact;
- a dangerous action receives an approval boundary.

---

# Part 4. The three traps — amplified in skills

One defect in a skill can recur across many invocations:

| Trap | In a one-time prompt | In a repeatable skill |
|---|---|---|
| Lexical ambiguity | One answer may choose the wrong object. | Different users receive different interpretations of the same workflow. |
| Presupposition | The model may confirm an unchecked assumption. | The workflow systematically produces false findings or unnecessary actions. |
| Implicature | An implied requirement is omitted. | Incomplete output becomes stable behavior. |

The solution is to make the **relevant** parts explicit: the exact trigger, object, boundary, alternative, and validation. Do not preload everything else “just in case.”

---

# Part 5. One-page checklist

## For a prompt

- [ ] Is the expected outcome observable?
- [ ] Does the context actually change the answer?
- [ ] Are ambiguous terms resolved?
- [ ] Have hidden assumptions been checked?
- [ ] Are important implied requirements explicit?
- [ ] Do real constraints include a desired alternative?
- [ ] Is the format minimal and usable by the next consumer?
- [ ] Is an example included only when it closes a real gap?
- [ ] Are there verifiable completion criteria?

## For a skill

- [ ] Do `name` and `description` follow the Agent Skills specification?
- [ ] Does the description say when to use and not use the skill?
- [ ] Does the skill have one coherent repeatable job?
- [ ] Are inputs, output, and preconditions clear?
- [ ] Are absolute prohibitions true invariants?
- [ ] Do dangerous actions have an approval boundary?
- [ ] Are large rare details moved into routed references?
- [ ] Does validation rely on verifiable evidence?
- [ ] Have trigger behavior and task output been tested separately in fresh sessions?
- [ ] Is cost measured together with retries and corrections?

---

# Final: the author’s 10 rules — with scope

1. **Explicit is stronger than implied.** Make requirements that change the outcome explicit.
2. **A constraint narrows the space.** Add a safe or desired alternative.
3. **NEVER is stronger than MUST; MUST is stronger than SHOULD.** Use absolute modality only for absolute rules.
4. **A template is stronger than a description when repeatable structure matters.** A schema may be unnecessary for a free-form human answer.
5. **Concrete is stronger than abstract.** Name the exact object, action, and boundary.
6. **Order controls salience.** Do not hide critical information, but do not treat position as a technical guarantee.
7. **Language affects cost and reliability.** Measure with the target tokenizer; maintain the document in its users’ language.
8. **A reference saves context for large, rare examples.** One decisive inline example may be more useful than a link.
9. **150 tokens is a discipline, not a universal limit.** Minimize until sufficient, not until ambiguous.
10. **A reusable skill scales a defect.** Test its trigger, output, safety, and failure behavior.

**Two grammars. One goal: a precise, verifiable result without unnecessary context.**

---

# Appendix: original quantitative estimates

The original work presented these rules of thumb:

| Technique | Original estimate |
|---|---:|
| JSON instead of free text | −45% output |
| Remove homonymy | −37% |
| Add prohibitions | −17% |
| `temperature = 0` | −15% |
| Short example | −15% |
| Combined application | −73% cost |
| English instead of Russian | −40% tokens |
| Remove filler | −50–70% |
| References instead of inline examples | −97% |
| Modularity | −60% |
| Templates instead of descriptions | −30% |
| Abbreviations | −20% |
| Self-check | +10–20% quality |

The original PQS proposal also associated ranges from `0.00–1.00` with retry rates from `50–90%` down to `<5%`, and gave illustrative scores of `0.15` for “Write a parser” and `0.87` for a detailed Python prompt.

**Status of these numbers:** they are the author’s illustrative estimates. The raw data, PQS formula, models, tokenizer, pricing, run count, and definitions of “quality” and retry are unavailable. The values have not been independently reproduced. Percentages cannot be added: effects may use different baselines and interact. For practical use, measure the complete loop on your own models and tasks.

---

# Sources and further reading

- [OpenAI: Prompting](https://learn.chatgpt.com/docs/prompting) — prompt components as a flexible set rather than a mandatory formula.
- [Anthropic: Prompt engineering best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices) — positive instructions, examples, structure, and model-specific caveats.
- [Agent Skills specification](https://agentskills.io/specification) — the portable `name`/`description` minimum and progressive disclosure.
- [PromptPrism: A Linguistically-Inspired Taxonomy for Prompts](https://aclanthology.org/2026.findings-eacl.61/) — linguistic decomposition of prompt structure, semantics, and syntax.
- [POSIX: A Prompt Sensitivity Index](https://aclanthology.org/2024.findings-emnlp.852/) — sensitivity to paraphrases and the effect of few-shot examples in the studied tasks.
- [Flaw or Artifact? Rethinking Prompt Sensitivity](https://aclanthology.org/2025.emnlp-main.1006/) — some measured sensitivity depends on the evaluation method.
- [Lost in the Middle](https://aclanthology.org/2024.tacl-1.9/) — positional effects in long-context retrieval and QA tasks.
- [Language Model Tokenizers Introduce Unfairness Between Languages](https://arxiv.org/abs/2305.15425) — cross-language tokenization differences without a universal Russian-to-English ratio.
- [When Does Intrinsic Self-Correction Help?](https://arxiv.org/abs/2606.23196) — self-correction as a task-dependent strategy rather than a universal guarantee.
