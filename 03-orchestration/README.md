# Orchestration README

This document describes the hand-off schema, the expected `run-output/{run_id}.json` format, and how auditors should inspect Cortex runs.

Hand-off schema (Orchestrator → Validator):

- `run_id` (string): unique run identifier, e.g. `no-roadmap:2026-08-13`.
- `draft` (string): the proposed update text.
- `source_log` (array of strings): timestamped tool call results and short excerpts the drafter relied on.

Validator response schema:

- `verdict`: `pass` | `fail`.
- `reasons`: array of human-readable findings with citations to `source_log` entries.
- `escalate`: boolean, true means human attention required.
- `_usage`: optional token accounting for auditing.

Sample auditor steps:

1. Open the persisted artifact `00-build/run-output/{run_id}.json` and verify the `draft`, `source_log`, and `validator_response` are present.
2. Confirm every factual claim in `draft` has a matching `source_log` entry (PR/issue IDs, numbers, dates, percentages).
3. If `verdict` is `fail`, review `reasons` to see whether the run should be escalated or revised.
4. Check `state.json` for the authoritative run status and dedupe keys.

Location of artifacts:

- Draft artifacts and validator responses: `00-build/run-output/`
- Run registry (dedupe & status): `00-build/state.json`

If you need an example run, see `00-build/run-output/run-no-roadmap-2026-08-13.json`.
