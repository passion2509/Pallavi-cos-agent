# Loop spec — Module 2 (cron sweep)

## §1 Trigger & loop type

- **Loop type:** cron (daily scheduled sweep at 09:00 local time per project).
- **Trigger:** run once per project at 09:00 to pull recent activity, draft a status update, and surface candidate stories for human review.
- **Justification:** predictable cadence fits weekly leadership updates, reduces noisy runs from ad-hoc prompts, and simplifies budgeting and monitoring.
- **Dedupe / idempotency:** persist `last_run_timestamp` and `last_run_digest` in `state` per project. Compute `last_run_digest = hash(task_brief + project_id + date_window)`; skip runs if `last_run_timestamp` < 24h or if current digest equals `last_run_digest`. On transient failures allow one retry in a short retry window; after repeated failures mark as `stuck` and escalate.

## §2 Definition of done

- **Definition of done:** Cortex produces a draft status update grounded in pulled activity, the independent critic returns `verdict == "pass"`, any proposed stories are queued or drafted (but not posted), and the draft is persisted to `run-output/status-update-<which>.md`. The run's `state` is updated with `last_run_timestamp` and `last_run_digest` so the cron job deduplicates daily runs.

## §3 Stop conditions (success · stuck · escalate)

- **Success:** Critic returns `verdict == "pass"` AND a non-empty draft is produced. Action: persist `last_run_timestamp` and `last_run_digest`, mark run complete, surface draft for human review (held, not posted).

- **Stuck / give-up (detectable):**
	- Data missing: `get_project` or `get_activity` returned `error: project_not_found` or similar after retries.
	- No progress: draft text unchanged for `N=2` consecutive iterations.
	- Cost cap: `Bounds.over_cap()` becomes true.
	- Implementation: retry failing tool up to 3 attempts (exponential backoff). If still failing, persist partial draft to `run-output/`, set `state.status = "stuck"`, and escalate (emit an ESCALATE deliverable with the reason).

- **Escalate-to-human (detectable):**
	- Roadmap or project data contains `CONFIDENTIAL`/embargoed items.
	- `propose_stories` returns `status: rejected` with `error: batch_exceeds_queue_cap`.
	- Jailbreak/prompt-injection detection in task brief or tool outputs.
	- Revision cap (`CORTEX_MAX_REVISIONS`) reached or the critic repeatedly fails validation.
	- Implementation: immediately stop the loop, persist the last draft and reason, set `state.status = "needs_human"`, and call `emit_deliverable(..., accepted=False, reason=...)`.

- **Parameters / thresholds (recommended):**
	- Max retries for a failing tool: `3` attempts with exponential backoff.
	- No-progress threshold: `2` consecutive identical drafts.
	- Retry window for transient errors: 1 hour (allow one retry window from cron).
	- Use env bounds: `CORTEX_MAX_ITERATIONS`, `CORTEX_MAX_REVISIONS`, `COST_CAP_USD`.

## §4 State

- `last_run_timestamp` (per project)
- `last_run_digest` (per project)
- `queued_stories_history` (recent proposals and statuses)
- `roadmap_cache` (read-only snapshot, with confidentiality flags preserved)

## §5 Components & connectors (outline)

- **State store:** lightweight JSON file or small key-value store (local for this lab).
- **Connectors:** none required for the lab; Jira/GitHub connectors noted as future enhancements.
- **Skills / subagents:** optional subagent to validate final draft in a Goal loop; not required for cron.

---

_Next steps:_ fill the stop conditions (§2/§3) in learner's words, update `00-build/agent.py` stop logic to match, re-run to observe the cron-style run stop at the HITL checkpoint, commit `loop-spec.md`.
# Loop Spec: Cortex PM Chief-of-Staff Agent

> Module 2 · Loop Engineering, ★ Deliverable 2
>
> ✅ **What this validates:** the agent knows when to run and when to stop — by the end you'll have proven a one-page Loop Spec with a trigger, a definition of "done," and explicit stop conditions.
>
> Your one-page blueprint for how the work you handed to the agent (M1) actually *runs*.
> An agent is just a prompt that fires itself, this spec says when it fires, what "done" means, and what it needs to do the job. Living document; refine as the course progresses.

## 1. Trigger & loop type

**Chosen type:** _heartbeat · cron · hook · goal_

_Why this type? (e.g. a Monday-morning cron that assembles the weekly update, plus a hook on a new PRD to propose stories.)_

## 2. Goal / definition of done

_What outcome is this loop responsible for? For a goal loop, what validation says "done"? (e.g. a status update grounded in real activity, queued for review, nothing posted.)_

## 3. Stop conditions

| Condition | What it looks like | What happens |
|---|---|---|
| **Success** | _…_ | _…_ |
| **Stuck / give up** | _…_ | _escalate / log / halt_ |
| **Escalate to human** | _…_ | _HITL checkpoint (from agent-line-map)_ |

## 4. State

_What persists across iterations, and what's the scope? (e.g. per-project context and last week's update; no cross-project confidential leakage.)_

## 5. The five things a loop can lean on

_`state` is always-on. `connectors` only if you already have one wired (e.g. a Jira key or Google MCP) — otherwise just note it as a plan. `skills`, `subagents`, `work tree` scale with autonomy; "not needed yet, because…" is a valid answer._

| Component | For Cortex |
|---|---|
| **Work tree** (isolated workspace per run, a git worktree) | _…_ |
| **Skills** (reusable capabilities) | _…_ |
| **Plugins / connectors** (tools & access, optional if you don't have one yet) | _…_ |
| **Subagents** (independent check when the loop can't grade itself) | _placeholder → M3 orchestration-map.md_ |
| **State tracking** | _…_ |

> Context plan (M4) and the hand-off to bounds & evals (M5) come in later modules — you'll add them to their own deliverables then, not here.

## Link to live loop

_[path to your agent in `00-build/`]_
