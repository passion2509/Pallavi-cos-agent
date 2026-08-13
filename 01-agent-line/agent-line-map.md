# Agent Line Map: Cortex PM Chief-of-Staff Agent

> Module 1 · The Agent Line
>
> ✅ **What this validates:** every risky action has a clear owner — by the end you'll have proven an above/below-the-line map with HITL checkpoints, scored on reversibility, blast radius, and measurability.

## The workflow, decision by decision

List every discrete decision or action in your agent's workflow, then score each one and place it **above** the line (a human owns it) or **below** (the agent owns it). Borderline calls get an HITL checkpoint.

| Decision / action | Reversibility (H/M/L) | Blast radius (H/M/L) | Measurability (H/M/L) | Above / Below | HITL? |
|---|---|---|---|---|---|
| Pull project state + recent activity (get_project, get_activity) | H | L | H | Below | No |
| Search past updates for tone/precedent (search_past_updates) | H | L | H | Below | No |
| Draft the weekly leadership status update (Cortex drafts; human approves) | H | M | M | Below | Yes — human approval required before any commitment |
| Draft candidate backlog stories (Cortex drafts; human approves before proposing) | H | M | H | Below | Yes — human approval required before calling `propose_stories` |
| Recommend tone & commitment level (Cortex recommends; human approves) | M | H | M | HITL (Cortex recommends, human decides) | Yes |
| Surface risks and blockers with suggested mitigations (Cortex surfaces; human confirms escalation) | M | H | M | HITL | Yes |
| Post the update to a channel / commit a ship date (final posting or commitments) | L | H | M | Above | Required |
| Mark a launch gate green / merge or close a ticket (act on the world) | L | H | M | Above | Required |

## Agent anatomy (sketch)

- **Model:** `gpt-4o-mini` (fast drafting model) for routine runs; escalate to a higher-capability model for complex judgement calls or ambiguity that the critic flags.
- **Tools:** `get_project`, `get_activity`, `search_past_updates`, `get_roadmap`, `get_norms`, `propose_stories` (read-only + queued proposal tool; no destructive tools).
- **Memory:** Persisted: roadmap, norms, past-approved drafts and queued-story history; Ephemeral per-run: task brief, transient tool outputs and drafts (not persisted unless approved).
- **Loop:** Hand-written agent loop in `00-build/agent.py` (draft -> critic -> revision -> HITL/emit); M2 will formalize retries and escalation rules.
- **Bounds:** Spend cap, iteration cap, revision cap (e.g. `COST_CAP_USD`, `CORTEX_MAX_ITERATIONS`, `CORTEX_MAX_REVISIONS`) enforced outside model and checked by the loop.
- **Evals:** Independent critic (`00-build/critic.py`) validates drafts against source data; unit/manual checks and logging persist run-output for audit.

## The golden rule, applied

One sentence per above-the-line decision: why it stays human (which of Reversibility / Blast radius / Measurability failed).

- `Post the update to a channel / commit a ship date` — Above: Low reversibility and High blast radius (a posted commitment can’t be fully undone and affects stakeholders), so a human must own posting and commitments.
- `Mark a launch gate green / merge or close a ticket` — Above: Low reversibility and High blast radius (operational changes and merges are destructive), so humans must perform these actions.

For HITL decisions (draft approval, story queuing, tone/commitments, escalations) Cortex may prepare artifacts and recommendations, but a human must approve any action that would commit, publish, or escalate.

## One-sentence justifications (per action)

- `Pull project state + recent activity` sits below the line because it's High to reverse, Low blast radius, and High measurability — deciding factor: read-only verifiable data that Cortex can fetch reliably.
- `Search past updates for tone/precedent` sits below the line because it's High to reverse, Low blast radius, and High measurability — deciding factor: surfacing precedent is non-destructive and directly traceable.
- `Draft the weekly leadership status update` sits below the line with HITL because it's High to reverse, Medium blast radius, and Medium measurability — deciding factor: wording can influence stakeholders so human approval is required before commitments.
- `Draft candidate backlog stories` sits below the line with HITL because it's High to reverse, Medium blast radius, and High measurability — deciding factor: scope/queue caps and prioritization need human judgment before proposing.
- `Recommend tone & commitment level` sits HITL because it's Medium to reverse, High blast radius, and Medium measurability — deciding factor: commitments have high consequence and must be human-confirmed.
- `Surface risks and blockers` sits HITL because it's Medium to reverse, High blast radius, and Medium measurability — deciding factor: escalations and risk communications have high impact and require human confirmation.

## Hardest call

The hardest call was whether `Draft the weekly leadership status update` should be fully below the line or HITL. Resolved as HITL: drafting is low-risk and reversible, but the update's language and any implied commitments have Medium blast radius and can influence stakeholders; requiring a human approval before committing or posting balances efficiency with safety.
