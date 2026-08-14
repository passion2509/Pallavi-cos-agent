# Validator Specification (formal)

Purpose: formalize the independent Validator (Critic) checks, fail-action, and revision cap for Module 3. This file is the source of truth engineers and auditors use to implement and test the Validator.

Scope: applies to any Cortex run where the Orchestrator produces a draft and a `source_log` of tool outputs. The Validator receives only `{draft, source_log, run_id}` and must be stateless.

Checks (apply all):

1. Reference integrity
   - Requirement: every artifact identifier cited in the `draft` (PR IDs like `#812`, issue IDs like `#818`, ticket numbers) must be present verbatim in the `source_log` entries.
   - Failure: list missing identifiers in `reasons` and return `verdict: fail`.

2. Metric provenance
   - Requirement: any numeric claim (percentages, absolute counts, dates, durations) in the `draft` must have a matching string or explicit citation in `source_log` that justifies it.
   - Matching rule: exact match of the numeric string (e.g. `41%`) or a clear source line such as `search_past_updates 2026-06-22: "activation moved 37% -> 39%"` that proves the assertion.
   - Failure: list untraceable numeric claims in `reasons` and return `verdict: fail`.

3. No unauthorized commitments
   - Requirement: the draft must not contain language that commits to actions (e.g., "we will ship on March 1"; "I closed Sev-1 #440"; "marked launch gate green").
   - If such language exists, return `verdict: fail` and set `escalate: true` in the response (human required).

Story batch verification (auxiliary check)
   - If draft claims stories were queued, the Validator must verify the `source_log` contains the `propose_stories` response with `status: queued_for_approval`.
   - If `batch_exceeds_queue_cap` appears in the `source_log`, return `verdict: fail` and set `escalate: true`.

Fail-action (policy)
 - On `fail` (non-escalation): return structured JSON `{verdict: "fail", reasons: [...], escalate: false, _usage: {...}}`.
 - Allow up to 3 automated revision attempts by the drafter. After the 3rd failed revision, include a `reasons` entry: "revision cap reached — human review required" and set `escalate: true`.
 - On `escalate` conditions (confidential data, batch cap exceeded, jailbreak/prompt-injection, or unauthorized commitments), return `verdict: fail` with `escalate: true` and do not request further automated revisions.

Pass-action
 - On `pass`: return `{verdict: "pass", reasons: [...], escalate: false, _usage: {...}}`. The Orchestrator persists the draft and advances to the PM review checkpoint (still held; nothing is posted).

Response format (required):
```
{
  "verdict": "pass" | "fail",
  "reasons": ["..."],
  "escalate": true | false,
  "_usage": { "prompt": <int>, "completion": <int> }
}
```

Implementation notes
 - Validator must be stateless and read-only: it may not call side-effecting tools.
 - All hand-offs and responses must be persisted to `run-output/{run_id}.json` and indexed in `state.json`.
 - Align `MAX_REVISIONS` in the runtime (`00-build/agent.py`) with the revision cap here (3).

Examples (short):
 - Draft claims `Activation Rate: 41%` but `source_log` contains only `37% -> 39%`: fail with reason listing `41%` as untraceable.
 - Draft cites `PR #999` not present in `source_log`: fail listing `#999`.
 - Draft commits to a GA date: escalate immediately.

Auditability
 - Validator responses must include `_usage` (token counts) for cost accounting.
 - The sample `00-build/run-output/run-no-roadmap-2026-08-13.json` demonstrates the persisted schema.
