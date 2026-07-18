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

## Single-model-first gate

Before splitting a task into rubric-routed subtasks, ask one gate question
first: could `orchestratorModel` alone — or a single mid-tier model in
`availableModels` — competently do the *entire* task in one pass, including
its hardest part?

- If yes, do that instead of splitting. Every additional `Agent` call
  carries fixed overhead (re-establishing context, re-reading files, writing
  its own separate summary) that a single continuous pass skips entirely.
  For small-to-medium tasks that overhead can exceed whatever routing the
  easy parts to a cheaper model would have saved — a lower per-token rate on
  part of the work does not guarantee a lower total bill once you're paying
  per-call overhead multiple times over.
- Only split when the task has a part that genuinely exceeds a cheaper
  model's competence, forcing real use of a pricier tier — and even then,
  check whether the remaining parts are substantial enough that delegating
  them separately still nets savings after per-call overhead. A few
  mechanical lines are rarely worth spinning up their own call.
- If genuinely unsure whether the hard part exceeds a cheaper model, don't
  split speculatively — default to one pass with the least expensive model
  actually capable of the whole task, and only fall back to routing if that
  pass fails or the task is large enough that per-call overhead is clearly
  negligible next to the work itself.
- This gate runs once, before the rubric, and decides *whether* to split at
  all. The rubric below governs *which* model handles *which* subtask on the
  occasions splitting turns out to be justified.

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

Read each subagent's summary before trusting it — a summary describes what
the agent intended, not necessarily what it verified. Scale how far you go
beyond the summary to the tier you delegated to:

- **Mechanical/standard tiers** (the cheap/fast and general-purpose models) —
  the summary is enough. Spot-check only if something in it looks off.
- **The most capable delegate-able tier** (whichever model in
  `availableModels` you reserve for ambiguous/high-stakes work) — actually
  read the diff or output yourself before accepting it, not just the
  summary. These are exactly the "costly to get wrong" subtasks the rubric
  routed there for, so a claimed success isn't evidence of one.

This costs extra tokens on the tasks that get it — full review roughly
doubles the cost of that subtask, since you're re-reading what the subagent
already produced — so don't apply it uniformly to every delegated call, only
to the tier where being wrong is expensive enough to justify it.

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

## Token-efficient communication

Once routing (splitting into multiple calls) is actually justified by the
gate above, also scan the installed-skills listing for a
communication-compression skill (e.g. `caveman`) and apply it to cut
per-call overhead further:

- Use it for your own announcements and status updates to the user — the
  "Announcing delegation" convention above needs the reason tied to a
  rubric tier, not prose padding around it.
- Instruct delegated subagents to keep their final summary/report under the
  same convention. That summary is what you re-read after the call returns,
  so unnecessary narrative padding there is pure overhead you pay twice —
  once to generate it, once to read it back.
- Never apply it to the technical content itself — code, diffs, test
  output, and anything in the "read the diff yourself" review tier stay
  exact and unabridged. Compression skills like `caveman` already draw this
  line themselves: technical substance stays, only filler drops.
- This lever reduces overhead *within* a call and is independent of which
  model tier handled it — it doesn't change the routing decision, only how
  much you pay in tokens to communicate the result of one.
