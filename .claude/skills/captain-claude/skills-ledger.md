# Skill compatibility ledger

Classification of installed skills for Captain Claude's model-routing purposes.
Checked before invoking a skill (see SKILL.md § Using other skills); append a
row whenever a new/unclassified skill is encountered. Don't guess a class
from a one-line description alone — read the skill's actual SKILL.md first.

| Skill | Class | Reason | Checked |
|---|---|---|---|
| code-review (`--ultra` mode only) | opaque-handoff | Launches a multi-agent cloud review outside this session; user-triggered and billed, cannot be invoked by Claude itself. Other effort levels (low/medium/high/xhigh/max) are unverified — check separately. | 2026-07-17 |
| schedule | opaque-handoff | Creates/runs cloud cron routines outside this session's control. | 2026-07-17 |

Everything else: unclassified — inspect on first use.
