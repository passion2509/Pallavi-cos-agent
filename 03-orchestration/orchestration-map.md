# Orchestration Map — Module 3

## Field 1 — Why split (one-line design)

One-line design: Add a Validator so drafts are independently verifiable and no numeric claims are self-validated.

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

Revision cap: `3` revisions (i.e., allow up to 3 `fail` cycles; on the 4th failure escalate to human). This matches the runtime `MAX_REVISIONS` bound in the agent loop.

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

Concrete topology (single Orchestrator + one Validator):

- Inbound: PM task brief enters the `Orchestrator/Writer` (Cortex).
- Read phase: Cortex calls read-only tools (`get_project`, `get_activity`, `search_past_updates`, `get_norms`) and builds a `source_log` of evidence.
- Write phase: Cortex composes a draft and sends the immutable payload `{run_id, draft, source_log}` to the `Validator` (Critic).
- Validation: Validator runs deterministic, auditable checks and returns `{verdict, reasons, _usage}`.
- Outcome: on `pass` the draft is persisted and held for human review; on `fail` the Orchestrator may revise (up to the cap) or escalate.

Properties:

- Sequential hand-offs, short-lived: no long-running shared process state beyond persisted artifacts.
- Single independent Validator avoids self-validation while keeping the run simple and auditable.

## 3. Roster

| Agent / subagent | Responsibility | Runs which Loop Spec |
|---|---|---|
| `Cortex` (Orchestrator / Writer) | Pulls project data, composes the draft, persists run artifacts, calls `propose_stories` (proposal-only) | M2 loop (daily)
| `Validator` (Critic) | Stateless, independent validation: reference integrity, metric provenance, no unauthorized commitments, story-batch check | Validation loop (per-run)
| `Human Reviewer` | Reviews held drafts, approves queued stories, resolves escalations | Manual checkpoint

Notes: keep the roster minimal; only the Validator has a distinct role and reduced privileges (read-only inputs, no side effects).

## 4. Communication & hand-offs

Summary: the system uses small, append-only hand-offs: Cortex → Validator → Persisted artifact → Human. Keep payloads minimal and auditable.

Hand-off schema (Orchestrator → Validator):

- `run_id` (string) — unique run key (e.g. `no-roadmap:2026-08-13`).
- `draft` (string) — the proposed status update text.
- `source_log` (array[string]) — ordered tool outputs and short evidence strings the draft relied on.

Validator response payload:

- `verdict`: `pass` | `fail`.
- `reasons`: array of concise, cited findings.
- `escalate`: boolean.
- `_usage`: optional token accounting for audit.

Persistence rules:

- Save both the request and the Validator response to `run-output/{run_id}.json` and index the run in `state.json` with `dedupe_key` and `status`.
- All hand-offs are append-only; Validator must not mutate shared state.

## 5. The validator

- **What the critic checks:** _grounded claims · norms compliance · no confidential leak · nothing posted/committed_
- **Fail action:** _what happens when it fails (retry · revise · escalate to human)_

## 6. State: shared vs isolated

Shared persistent state (authoritative):

- `state.json`: run registry with dedupe keys and run statuses (`success`, `needs_human`, `stuck`).
- `run-output/`: persisted drafts and validator responses (`run-output/{run_id}.json` and `status-update-{which}.md`).

Ephemeral/isolated state (per-run):

- In-memory model conversation/history used by the Orchestrator while composing a draft; discarded after persist.
- Validator working memory: runs checks on the provided `source_log` and returns a response; retains no long-lived state.

Access & invariants:

- Validator is strictly read-only: it receives `{draft, source_log}` and returns a verdict — no side-effecting tools.
- Only Cortex may call `propose_stories` (proposal-only); infrastructure enforces queue caps and rejects oversized batches.
- Any `CONFIDENTIAL` markers found in tool outputs must cause immediate escalate and must not be included in persisted external artifacts without explicit authorization.

## 7. Cost & latency budget

_Coordination has a price. Rough token/latency cost of the fleet vs a single agent. (Forward-link to M5 bounds.)_

Budget & latency guidance (concrete):

- Typical model calls per run: **2** (1 draft call by Cortex + 1 validator call).
- Worst-case model calls at revision cap: **2 * (revision_cap + 1)**. With `revision_cap = 3` this is `2*(3+1)=8` calls (4 drafts + 4 validations).
- Cost estimate: validator calls should be small (target <10k tokens). Aim for validator spend << draft spend — for affordable models this often maps to <$0.01 USD per validation; use `COST_CAP_USD` to enforce a run-level budget.
- Latency targets: validator response < 30s (typical); end-to-end typical latency < 60s. Worst-case latency with repeated revisions should be bounded (recommend: < 3 minutes) or escalate to a human to avoid blocking.
- Operational rule: if cumulative run cost >= `COST_CAP_USD` or total model calls exceed safety thresholds, persist the run as `needs_human` and escalate rather than continuing to loop.

Practical knobs to tune during M5:

- `CORTEX_COST_CAP_USD` — per-run dollar cap (example values for demos: $0.05; for demonstration runs you may raise to $1.00).
- `CORTEX_MAX_REVISIONS` / `revision_cap` — set to 3 in this map to allow a few automated fixes but limit runaway loops.
- `CORTEX_MAX_ITERATIONS` — guard overall loop length to avoid long-running runs.

Audit & fail-safe:

- Persist `_usage` token counts from both drafts and validator responses to `run-output/{run_id}.json` for post-run accounting.
- If repeated failures or cost spikes occur, require human intervention and record the reason in `state.json`.

This budget guidance should be carried forward to the M5 bounds and justified with expected model pricing for the chosen model.
---

## Detailed Topology, Roster, Hand-offs, State Split

This section consolidates the concrete orchestration details you can implement and test immediately.

Topology (concrete):

- `Orchestrator/Writer` (Cortex): calls read tools, builds `source_log`, composes draft, persists run artifacts, and sends `{draft, source_log, run_id}` to the Validator.
- `Validator` (Critic): stateless subagent that receives the draft + source_log, returns `{verdict, reasons, _usage}`.
- `Human Reviewer`: inspects held drafts and validator output, approves or requests manual edits.

Roster (concrete roles):

- `Cortex` — orchestrator/writer: privileges: read tools + write run-output/state.json + call `propose_stories` (proposal-only). Runs: daily M2 loop.
- `Validator` — critic: privileges: read-only `source_log` and draft; must not call side-effecting tools. Runs: validation loop per run.
- `Human Reviewer` — manual approver: privileges: approve/queue stories and resolve escalations.

Hand-offs & protocol:

- Orchestrator → Validator payload: `{"run_id": str, "draft": str, "source_log": str}`.
- Validator response: `{"verdict": "pass|fail|escalate", "reasons": [...], "_usage": {...}}`.
- Persist: both payload and response are saved to `run-output/{run_id}.json` and indexed in `state.json`.

State split and invariants:

- Shared persistent: `state.json` (run registry/dedupe), `run-output/` (drafts, validator verdicts). These are the only cross-run writable stores.
- Ephemeral/isolated: model message history, tool call responses kept in-memory until persisted; Validator retains no long-lived state.
- Access rules: Validator is read-only; only Cortex may call `propose_stories` and only as a proposal (no external side effects). Any confidential markers in tool outputs cause immediate escalate.

Budget & latency guidance:

- Target per-run cost: adhere to `COST_CAP_USD` (configurable via env). For demos, choose a small cap to force quick escalation when misbehaving.
- Validator must be efficient: small prompt, targeted checks → aim for <10k tokens.
- Operational rule: if revision retries exceed `MAX_REVISIONS` or cumulative cost >= `COST_CAP_USD`, escalate to human.

Next steps (recommended):

- Commit this map to the repo and update `00-build/prompts.py` `CRITIC_SYSTEM` to mirror the validator checks listed here.
- Add a short `orchestration/README.md` describing the hand-off JSON schema and sample `run-output/{run_id}.json` structure for auditors.

