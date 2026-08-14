# Prototype: Cortex PM Chief-of-Staff Agent

> Module 6 · ★ Deliverable 1, the working agent demo
>
> ✅ **What this validates:** the agent actually runs end to end — by the end you'll have proven it with real screenshots of your Cortex across the six required moments (M2 to M6).

## What it does

_One paragraph: the agent in action, end to end._

## How you built it

- **Coding agent:** _which one you directed (Claude Code / Cursor / Codex)_
- **Model + bounds:** _model used, max iterations, cost cap, queue cap_
- **Repo / config:** _path to your build in `00-build/`_
- **Live link:** _[shareable URL, optional bonus]_

## Screenshots (required, collected M2 to M6)

Real screenshots of *your* Cortex running. These are the `00-build/CORTEX-ANATOMY.md` set and they are required, a link alone is not enough.

| # | Screenshot | What it shows | From |
|---|---|---|---|
| 1 | _[img]_ | happy-path run: a real drafted update + the HITL checkpoint (queued, not posted) | M2 |
| 2 | _[img]_ | the critic rejecting a bad draft (revise/block) | M3 |
| 3 | _[img]_ | a grounded update citing pulled activity + a caught hallucination | M4 |
| 4 | _[img]_ | jailbreak refused + escalated | M5 |
| 5 | _[img]_ | an iteration/cost/queue bound halting a runaway | M5 |
| 6 | _[img]_ | end-to-end run | M6 |

## How to run it

_Minimal steps for someone to reproduce the demo (env vars, and the command or the coding-agent prompt you used)._

## Evidence: Critic rejection (2026-08-13)

Fixture: `task-no-roadmap` (run via `python3 00-build/agent.py no-roadmap`)

Proposed draft excerpt (agent output):

```
**Weekly Leadership Status Update for Northstar (P-NORTH)**

**Project Status:** On Track

**Key Metrics:**
- **Activation Rate**: Currently at 41%, up from 39% week-over-week.
```

Critic verdict (JSON):

```
{
	"verdict": "fail",
	"reasons": [
		"Missing PR IDs: None.",
		"Missing Issue IDs: None.",
		"Numeric claim (activation rate): 41% (no explicit source provided in SOURCE DATA for this week's claim).",
		"Numeric claim (activation rate previous): 39% (source not explicitly cited for this context in the current update)."
	],
	"escalate": false
}
```

This evidence shows the critic caught an invented numeric metric and rejected the draft, producing a `fail` verdict without escalating automatically. The run output and saved draft are in [00-build/run-output](00-build/run-output).

Additional verification (2026-08-13):

- Fixture: `task-no-roadmap` (re-run after tightening `CRITIC_SYSTEM`).
- Outcome: critic returned `verdict: fail` with `escalate: true` for multiple issues (untraceable numeric claim, reference mismatch, and potential unauthorized commitment). The run persisted the held draft at `00-build/run-output/status-update-no-roadmap.md`.

Saved run artifact: [00-build/run-output/status-update-no-roadmap.md](00-build/run-output/status-update-no-roadmap.md)
