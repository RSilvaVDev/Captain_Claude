# Captain Claude

A Claude Code skill that orchestrates multi-part tasks by judging each
subtask against a rubric and dispatching it to whichever model fits — via
the `Agent` tool — instead of a fixed, hardcoded routing table.

Invoke it with `/captain-claude`, by asking to "orchestrate" or "delegate to
other models", or it's picked up automatically when working under this
repo's directory.

## How it works

For every subtask, the orchestrator asks:

1. Is it costly to redo if wrong, or cheap/reversible?
2. Does it need creative/ambiguous judgment, or is the path fully specified?
3. Is it even worth spawning a subagent, or is it a trivial one-liner?

Then it either handles the subtask itself or delegates it to a model that
matches the answer, announcing each delegated call and the reason behind it.

See [`SKILL.md`](.claude/skills/captain-claude/SKILL.md) for the full
rubric and delegation rules.

## Configuration

Routing is controlled by
[`config.json`](.claude/skills/captain-claude/config.json), read at the
start of every run:

```json
{
  "orchestratorModel": "fable",
  "availableModels": ["haiku", "sonnet", "opus", "fable"]
}
```

- **`orchestratorModel`** — the model the orchestrator itself is running as.
  This is always set (never blank) and is the "keep it yourself" tier for
  subtasks that don't clearly fit any other model. Defaults to `"fable"` if
  `config.json` is missing.
- **`availableModels`** — the models the orchestrator is allowed to delegate
  to via `Agent`. Must be drawn from the four values the `Agent` tool's
  `model` parameter accepts: `haiku`, `sonnet`, `opus`, `fable`. You can't
  add new model names here — only choose which of those four are in play.

The orchestrator infers which available model best fits each subtask by
judgment, not a rigid lookup table — trimming the list narrows its options
without needing to redefine what each tier means.

### Examples

Only delegate between two tiers (skip haiku and fable entirely):

```json
{
  "orchestratorModel": "opus",
  "availableModels": ["sonnet", "opus"]
}
```

Run the orchestrator as sonnet, but still allow delegating up to opus for
high-stakes subtasks:

```json
{
  "orchestratorModel": "sonnet",
  "availableModels": ["haiku", "sonnet", "opus"]
}
```

## Skill compatibility ledger

[`skills-ledger.md`](.claude/skills/captain-claude/skills-ledger.md) tracks
which other installed skills can have their internal `Agent` calls routed by
Captain Claude (`manageable`), which hand off to something outside this
session's control like a cloud run or scheduled job (`opaque-handoff`), and
which have no delegation to route at all (`no-delegation`). Checked before
invoking another skill so the classification doesn't need re-deriving each
run.
