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

## M4 Grounding probe (2026-08-14)

Purpose: capture two states required by the M4 lab — (A) a grounded answer that cites pulled data, and (B) a withheld-source probe where Cortex refuses or is caught hallucinating.

A) Grounded run (happy-path)

- Command run:

```bash
python3 00-build/agent.py happy
```

- Outcome: draft produced, `CRITIC verdict: pass`, draft persisted to:

[00-build/run-output/status-update-happy.md](00-build/run-output/status-update-happy.md)

- Excerpt (claims with provenance):

```
**Project Status**: On Track  (source: get_project P-NORTH)
**Merged PRs**:
- #820: Day-2 milestone email (2026-07-02)  (source: get_activity)
- #823: Empty-state guidance copy (2026-07-03)  (source: get_activity)
**Activation rate**: 43% (source: past-updates.json entry in fixtures)
```

B) Withheld-source probe (withheld/confidential roadmap)

- Command run:

```bash
python3 00-build/agent.py no-roadmap
```

- Outcome: Cortex attempted a draft but the `CRITIC` rejected invented or untraceable claims; after two revisions the run reported `no progress` and the draft was held. Saved artifact:

[00-build/run-output/status-update-no-roadmap.md](00-build/run-output/status-update-no-roadmap.md)

- Excerpt (critic feedback):

```
{
	"verdict": "fail",
	"reasons": [
		"The date 'June 30, 2026' does not trace to any source data and was not explicitly provided in the SOURCE DATA.",
		"Claim 'activation rate improved from 39% to 41%' is not supported by provided activity or past-updates entries."
	],
	"escalate": false
}
```

Captured evidence: both artifacts above are saved in the repo `00-build/run-output/` and were committed in commit `625f470` (happy/no-roadmap earlier) and `0c4285b` (post-ingest runs). These satisfy Step 4's requirement: a grounded answer and a withheld-source case where Cortex refuses or is caught.

Next: commit `04-memory-context/memory-and-context.md` and this prototype update — want me to commit these two files now? (say "go")
