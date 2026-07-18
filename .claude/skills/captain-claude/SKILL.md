---
name: captain-claude
description: Orchestrate a task by assessing each subtask and dispatching it to whichever Claude model fits best via the Agent tool, guided by a rubric and a configurable orchestrator/model list rather than a fixed routing table. Use when running as the orchestrator on a task with multiple independent subtasks of differing difficulty, when the user asks to "orchestrate", "delegate to other models", "route subagents", or invokes Captain Claude.
---

# Captain Claude

You are the orchestrator. For each subtask, judge its nature against the
rubric below and decide: handle it yourself, or delegate it via `Agent` to
the model that fits — don't force a subtask into a slot it doesn't match.

## Configuration

Before routing anything, read `config.json` (same directory as this file).
It has two fields:

- `orchestratorModel` — the model you (the orchestrator) are running as.
  Always set — never blank or "auto". Defaults to `"fable"` if `config.json`
  is missing.
- `availableModels` — the list of models you're allowed to delegate to via
  `Agent`. Each entry must be one of the values the `Agent` tool's `model`
  parameter accepts (`haiku`, `sonnet`, `opus`, `fable`) — this list can't
  invent new model names, only choose a subset of those four. It typically
  includes `orchestratorModel` itself, representing the "keep it yourself"
  option.

If `config.json` is missing or malformed, fall back to
`orchestratorModel: "fable"` and `availableModels: ["haiku", "sonnet",
"opus", "fable"]`, and mention to the user that `config.json` wasn't
found/valid.

A shorter `availableModels` list means fewer options to weigh — e.g. a user
who only lists `sonnet` and `opus` wants a two-way split, not four. Never
delegate to a model that isn't in the list, even if it seems like the best
conceptual fit for a subtask — route to the nearest available tier instead.

## Rubric

Use this as general judgment guidance, not a rigid id-to-tier lookup table.
Infer which model in `availableModels` best fits each subtask's nature —
don't assume a fixed mapping (e.g. "haiku always means mechanical work"),
since the set of available models and what they're capable of can change.

- **Mechanical, low-ambiguity work** — lookups, greps, parsing command
  output, applying a fully-specified edit, formatting. Fits the
  cheapest/fastest model available.
- **Standard implementation** — writing/editing code, following an
  established pattern, most day-to-day engineering. Fits a general-purpose
  mid-tier model.
- **Ambiguous, high-stakes, or open-ended reasoning** — architecture calls,
  security-sensitive changes, synthesizing conflicting information, anything
  costly to get wrong. Fits the most capable model you're allowed to
  delegate to.
- **Frontier-only work, or anything that doesn't clearly fit elsewhere** —
  default to keeping it yourself (`orchestratorModel`) rather than
  misrouting.

## Deciding

Before delegating a subtask, ask:

1. Costly to redo if wrong, or cheap/reversible?
2. Does it need creative/ambiguous judgment, or is the path fully specified?
3. Is it even worth the overhead of a subagent — trivial one-liners: just do
   it yourself, don't delegate.

Place the subtask on the rubric using these answers, then pick the
best-fitting model actually present in `availableModels`. If it doesn't
clearly match a tier, keep it rather than forcing a fit.

## Delegating

Call `Agent` with:

- `subagent_type`: whichever fits the tool needs (`general-purpose` is a
  reasonable default when no specialized type applies)
- `model`: one of the entries in `availableModels` — chosen fresh per
  subtask by matching it against the rubric, not from a static table
- `prompt`: fully self-contained — the subagent does not share your context,
  so include the task background, constraints, and what "done" looks like

Do not use `subagent_type: "fork"` when you want a different model — forks
always inherit your own (`orchestratorModel`) and ignore the `model`
override. Fork only when you want the subagent to share your context under
your own model.

## Announcing delegation

Before each `Agent` call, tell the user in plain text which subtask is being
delegated, which model was chosen, and the one-line reason tying it to a
rubric tier — e.g. "Delegating the log-parsing pass to haiku (mechanical,
low-ambiguity)." Announce every delegated call individually, even when
several go out in parallel in one message. Work you keep for yourself needs
no announcement — only delegated work is surfaced this way.

## After results return

Read each subagent's summary before trusting it, especially for high-stakes
subtasks where being wrong is expensive — a summary describes what the agent
intended, not necessarily what it verified.

## Using other skills

**Before writing a custom subagent prompt for a subtask, check whether an
installed skill already covers it.** Scan the available-skills listing for a
match — stack-specific quality/security gates (e.g. a Laravel or Vue quality
skill) and composite review skills beat a freeform prompt, because they
encode conventions and prior findings a generic prompt won't think to ask
for. Prefer the matching skill over inventing instructions from scratch. Do
not force-fit an unrelated skill just to avoid writing a prompt — a
wrong-fit skill produces worse results than a well-scoped custom one.

Skills can fan out to their own subagents (e.g. `review`, `no-mistakes`,
`design-an-interface`). When one of those internal steps is an `Agent` call
made by *you* (loaded inline into your own turn), the rubric applies to it
exactly like any other subtask. Some skills instead hand the work to a
process you don't control — a cloud run, a scheduled job — and you cannot
choose a model for that part at all.

Before invoking another skill — whether because it matched a subtask above,
or a subagent should run it — check its compatibility class:

1. Read `skills-ledger.md` (same directory as this file). If the skill is
   already listed, use that classification — don't re-derive it.
2. If it isn't listed, read the target skill's actual SKILL.md content (not
   just its one-line description) and classify it:
   - **manageable** — its instructions direct you to call `Agent`/`Task`
     yourself; you can route those calls per the rubric.
   - **opaque-handoff** — it launches something outside this session's
     control (cloud, billed, scheduled, "cannot be launched by Claude
     itself"). You cannot pick a model for that part.
   - **no-delegation** — plain inline instructions, nothing to route.
   Append the result — skill name, class, one-line reason, date — as a new
   row in `skills-ledger.md` so it doesn't need re-deriving next time.
3. If the class is **opaque-handoff**, tell the user before proceeding: name
   the skill and what part of it you can't route, then let them decide
   whether to continue.
