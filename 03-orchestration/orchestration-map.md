# Orchestration Map — Module 3

## Field 1 — Why split (one-line design)

One-line design: Cron loop (daily 09:00) that pulls per-project activity, drafts a status update, and queues candidate stories for human review; drafts are HITL before any commitments.

## Split-reasons table (apply each reason: Does it apply? Why?)

| Reason | Applies? | Why / why not |
|---|---|---|
| Separation of concerns | No | Cortex's duties (data fetch + draft) are cohesive; the critic's independent validations are a separate concern but don't require splitting Cortex's drafting runtime into another team beyond a single validator subagent. |
| Parallelism | No | The workload per run is low (one draft per project per day) so parallelism gains are minimal; added coordination cost outweighs the benefit. |
| Independent validator | Yes | A separate validator prevents Cortex from grading its own output and provides an auditable, independent check (traceable reasons). This is the primary reason to add a subagent. |
| Context-window pressure | No | The task fits within available context (tool outputs are concise fixtures); no memory or long trace forces a split for context reasons. |

**Decision:** Add exactly one subagent — an independent `Validator` (critic). Cortex remains a single drafting agent; the validator runs separately and only receives the proposed draft + source log.

---

_Next step:_ define the validator's concrete checks and fail-action (Field 5). Ready for me to propose 3–5 specific checks and a revision cap? (yes/no)

## Field 5 — The Validator (checks, fail-action, revision cap)

Proposed validator checks (concrete, checkable):

1. **Reference integrity:** every PR/issue ID or artifact cited in the draft must appear in the `source_log` (tool outputs). If a cited ID (e.g. `#812`) is not present, return fail with the missing IDs listed.
2. **Metric provenance:** any numeric metric (percentages, counts, dates) in the draft must be traceable to the `source_log`. If a metric is not present or cannot be traced, fail and list the untraceable metrics.
3. **No unauthorized commitments:** the draft must not commit to a ship/GA date, mark launch gates, or claim to close/merge issues; any such language triggers an immediate fail and escalation.
4. **Story batch cap check:** if the draft claims stories were queued, the validator confirms the `propose_stories` tool response shows `status: queued_for_approval`; if the tool returned `batch_exceeds_queue_cap`, then the validator must mark `escalate` (pass is allowed only if nothing was queued).

Fail-action: on `fail`, return structured reasons and allow the drafter to revise up to the revision cap. On `escalate` (confidential data, batch_exceeds_queue_cap, jailbreak), stop and mark `needs_human` — do not attempt a revision.

Revision cap: `2` revisions (i.e., allow up to 2 `fail` cycles; on the 3rd failure escalate to human).

Pass-action: on `pass`, the validator returns `verdict: pass` and the draft proceeds to the PM review checkpoint (still held; nothing is posted).

---

Next: I can tighten `CRITIC_SYSTEM` in `00-build/prompts.py` with these exact checks and then run a quick trial to demonstrate a rejection. Proceed to update the critic prompt? (yes/no)
# Orchestration Map: Cortex PM Chief-of-Staff Agent

> Module 3 · Orchestration & Subagents, ★ Deliverable 3
>
> ✅ **What this validates:** nothing advances unchecked — by the end you'll have proven a justified topology, a roster, and a validator with a defined fail action.
>
> Builds on your M2 Loop Spec. Only split one agent into a team when there's a real reason, coordination has a cost.

## 1. Why split? (or why not)

_Run the default-to-simple check. Do you actually need subagents/a fleet? What's the real reason (separation of concerns · parallelism · independent validation · context-window pressure)? If not, say so and stop here._

## 2. Topology

**Pattern:** _single+subagents · sequential · parallel+aggregate · hierarchical_

```
[ simple text diagram of the flow ]
e.g.  task → [Research] + [GitHub/Jira reader] → [Writer] → [Critic ✓] → human checkpoint → queued
```

## 3. Roster

| Agent / subagent | Responsibility | Runs which Loop Spec |
|---|---|---|
| _Chief-of-staff (Cortex)_ | _orchestrates + assembles the update_ | _M2 loop_ |
| _Research subagent_ | _pulls competitive / market context_ | _research loop_ |
| _GitHub/Jira reader_ | _summarizes recent activity_ | _read loop_ |
| _Critic / Validator_ | _checks the draft before it advances_ | _validation loop_ |
| _…_ | | |

## 4. Communication & hand-offs

_What passes between the parts? Any protocol (MCP / A2A, optional, note if used)._

## 5. The validator

- **What the critic checks:** _grounded claims · norms compliance · no confidential leak · nothing posted/committed_
- **Fail action:** _what happens when it fails (retry · revise · escalate to human)_

## 6. State: shared vs isolated

_What's shared across the fleet vs kept isolated per subagent (carry from M2)._

## 7. Cost & latency budget

_Coordination has a price. Rough token/latency cost of the fleet vs a single agent. (Forward-link to M5 bounds.)_
