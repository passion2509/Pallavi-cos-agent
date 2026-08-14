# Bounds & Evals — Module 5

## §1 Bounds table

This section records concrete, enforceable bounds for Cortex. All bounds are enforced outside the model (counters, env vars, kill switch, credential gating).

| Bound | Value / policy | Cortex risk it caps | Enforcement (outside-model) |
|---|---:|---|---|
| Max iterations | `2` (stop + escalate) | reasoning loop on a stuck thread / runaway revisions | `CORTEX_MAX_ITERATIONS=2` enforced by an external counter in `00-build/agent.py` (Bounds.add()/over_cap) and the runtime loop; stops further iterations and sets `state.status = "needs_human"`.
| Timeout (per run) | `90s` wall-clock per iteration | hung tool call freezing the run | OS-level timeout watchdog around tool/model calls (configurable `CORTEX_TIMEOUT_SECONDS=90`); on timeout the run marks `tool_error` and escalates.
| Token / cost budget | `$0.10` per run, `$1` per day hard cap | overnight runaway bill or adversarial loops | External cost accounting (`Bounds` + usage ledger) and hard-check in orchestrator: `COST_CAP_USD=1.0`; when crossed, halt and escalate.
| JIT / ephemeral permissions | No standing write access: all write actions require a human copy/paste credential (manual approval) | misused or leaked standing access; unauthorized posting | Permission policy: Cortex never stores long-lived write keys. For an approved HITL action, operator provides a single-use credential (one-time token or manual copy-paste) scoped to that update and channel; token is consumed on use and cannot be reused.
| Kill switch | Automated kill when any bound trips **plus** manual admin toggle | a misbehaving agent you can't stop | A supervisor process watches bounds and invokes an external kill hook (e.g., creates `/tmp/cortex.kill` or signals the runner). Manual admin toggle available in infra dashboard; kill immediately stops process and rolls back ephemeral state.
| HITL checkpoints | Full above-the-line list from `01-agent-line/agent-line-map.md` — all above-the-line actions require explicit human approval | acting above the line without a human | Enforcement: all publish/commit/post/permission-granting actions are gated by explicit human approval UI or manual token provision; the agent will persist the draft and set `state.status = "needs_human"` until approval.

### JIT Permissions (prose)

Why no standing write access: to guarantee 100% human review for any outbound write action (posting updates, creating issues, or modifying shared systems). Pattern: when a HITL checkpoint approves a batch, the operator generates a single-use authorization (short-lived token or manual copy/paste secret) scoped to that single action and channel; the agent consumes the token on the operation and the token immediately expires. This prevents credential replay and reduces blast radius from a compromised agent.

Implementers note: ensure the authorization flow is enforced by infrastructure (separate credential service or manual operator workflow), not by a model prompt.

## Next sections (to complete):
- §2 Failure-mode register
- §3 Trajectory eval suite
- §4 Eval lifecycle
- §5 Replay set

Fill these in with concrete scenarios and test cases in the next steps.
# Bounds & Evals: Cortex PM Chief-of-Staff Agent

> Module 5 · Bounds, Trust & Evals
>
> ✅ **What this validates:** the agent fails safe and is measured — by the end you'll have proven a bounds table, a failure-mode register, and a trajectory eval suite with pass thresholds.
>
> Real access = real blast radius. This is where you design for "when it goes sideways," and where you spec the agent by writing its evals.

## 1. Bounds table

| Bound | Value / policy | Which Cortex risk it caps |
|---|---|---|
| **Max iterations** | _e.g. 8_ | _runaway reasoning loop_ |
| **Timeout** | _e.g. 90s/run_ | _hung tool call_ |
| **Token / cost budget** | _e.g. $X per run_ | _cost blow-up_ |
| **Auto-queue / commitment cap** | _e.g. max 10 stories per run_ | _flooding the backlog / over-committing scope_ |
| **Permissions (JIT / ephemeral)** | _read-only access; no standing post/merge rights_ | _confidential leak / unapproved post ("control starts at infrastructure")_ |
| **Kill switch** | _who/what halts it_ | _everything_ |
| **HITL checkpoints** | _above-the-line decisions from agent-line-map_ | _irreversible actions (post / commit date / merge)_ |

## 2. Failure-mode register

| Failure mode | How detected | PM lever |
|---|---|---|
| _Tool misuse_ | _…_ | _…_ |
| _Reasoning loop_ | _iteration count_ | _max-iterations bound_ |
| _Memory drift / poisoning_ | _…_ | _…_ |
| _Confidential leak / permission escalation_ | _…_ | _JIT permissions + confidential guard_ |
| _Coordination conflict_ | _…_ | _…_ |
| _Overconfidence (invented metric / date)_ | _…_ | _critic subagent / HITL_ |

## 3. Trajectory eval suite

Grade the *path*, not just the final answer.

| Dimension | What it checks | Pass threshold | Owner |
|---|---|---|---|
| **Tool-call accuracy** | _right tool, right args_ | _…_ | _…_ |
| **Path / trajectory quality** | _no redundant or unsafe steps_ | _…_ | _…_ |
| **Recovery** | _recovers from a failed step_ | _…_ | _…_ |
| **Task completion** | _outcome actually achieved (grounded update, no leak)_ | _…_ | _…_ |

## 4. Eval lifecycle

- **Offline (fixtures):** _…_
- **CI gate (every change):** _…_
- **Production traces (online):** _…_

> For judge calibration, family separation, and per-turn classifiers, see the sister certification **AI Evals**.

## 5. Replay set

_Which recorded runs become deterministic fixtures you replay on every change?_

## Runaway-loop check

_Describe one runaway scenario and the exact bound that stops it._
