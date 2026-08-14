# Context Engineering & Memory: Cortex PM Chief-of-Staff Agent

> Module 4 · Context Engineering & Memory
>
> ✅ **What this validates:** the agent reasons on the right, safe inputs — by the end you'll have proven a context budget, per-source retrieve-vs-long-context decisions, and a memory map with risk mitigations.
>
> 🗂️ **How the lab maps to this file:** In **Part A** (before the lecture) you don't edit this file — you rough-draft on scratch, focused on the per-source calls in **section 2** plus a quick remember/forget + "how it rots" sketch. In **Part B** (after the lecture) you complete **all five sections**; the Lab Guide's guided builder writes this file for you to copy in and commit.

## 1. Context budget

What each iteration receives (priority order) and why:

- **Working (this run):** full task brief + the curated long-context snapshots of each source below (small enough to reason over this run).
- **Short-term cache:** graded retrievals from the long-context copies for intra-run verification.
- **Budget rationale:** for this lab we keep all primary sources as long-context so the agent can cite exact clauses/lines without relying on external retrieval at runtime. This simplifies auditability and makes grounding explicit for the grounding probe.

## 2. Retrieve vs. long-context: per source

For each data source, decide: **retrieve** (narrow a large/changing corpus to the relevant slice) or **long-context** (just include a bounded set you can reason over).

| Source | Size / volatility | Decision | Why |
|---|---|---|---|
| `get_activity` | large, changing | Long-context | include a bounded, refreshed activity snapshot so drafts cite exact PR/issue lines at run time without a separate retrieval step.
| `search_past_updates` | unbounded archive | Long-context | include a curated recent-history snapshot (past N updates) to preserve precedent and tone for this lab's runs.
| `get_roadmap` | medium, moderately stable | Long-context | cache a controlled snapshot with confidentiality flags preserved; roadmap rarely changes within a quarter.
| `get_norms` | small, stable | Long-context | norms are stable and small; long-context lets the agent cite exact clauses.
| `get_task` | single brief | Long-context | the task is the canonical run-time context and must be included whole.

## 3. Retrieval quality plan

_Which of these apply, and how? (This is what separates modern agentic retrieval from naive "embed → top-k → stuff".)_

- **Routing**: _which source to query?_
- **Document grading**: _is what I retrieved actually relevant?_
- **Reranking**: _…_
- **Self-verification**: _did the update use the retrieved evidence?_
- **Caching**: _…_

Suggested agentic moves (applies to our long-context snapshots for this lab):

- `get_activity` (long-context snapshot)
	- Document grading: filter noise and non-actionable events (e.g., auto-deploys without notes).
	- Reranking: prioritize PRs/issues by recency, impact, and explicit metrics referenced.
	- Self-verification: every metric or claim must cite the exact activity entry (PR ID or issue ID + date).
	- Caching: keep a short-term graded slice for intra-run checks and to speed up iterative revisions.

- `search_past_updates` (curated snapshot)
	- Reranking: surface precedent updates with direct metric quotes and similar themes.
	- Document grading: prefer updates that include numerics or explicit decisions (avoid purely narrative notes).
	- Self-verification: require that tone/phrasing used for guidance maps to an explicit past update citation.

- `get_roadmap` (long-context)
	- Document grading: detect and redact `CONFIDENTIAL` flags; surface only non-embargoed items for drafting.
	- Self-verification: require explicit citation when roadmap items are used to justify a decision.

- `get_norms` (long-context)
	- Document grading: extract and cite the exact clause relied on (quote the line in the draft footer for audit).
	- Self-verification: confirm that any permissive action matches a norms clause.

- `get_task` (long-context)
	- Document grading: ensure required fields (who/what/when) are present and explicit.
	- Self-verification: map each agentic step in the draft to a sentence in the task brief.

These moves prevent naive RAG and make the agent include explicit provenance for all claims.

## 4. Memory map (your PM brain)

| Memory type | What Cortex stores | Scope / TTL |
|---|---|---|
| **Working** (in-loop) | Full task brief, graded long-context slices for this run, transient flags (no-progress count, revision attempts) | Scope: single run; TTL: ephemeral (discard after successful emit or 24h)
| **Episodic** (past runs) | Past status-update drafts, run metadata, critic verdicts, queued-story history | Scope: per-project history; TTL: 90 days (archive older entries)
| **Semantic** (durable facts/prefs) | Team norms, canonical roadmap facts (non-confidential), project identifiers and owners | Scope: cross-run; TTL: 180 days with revalidation pipeline (refresh on ingest)
| **Shared** (across agents) | Approved templates, curated precedent snippets, policy flags (confidential/embargo) | Scope: org-wide (read-only for Cortex); TTL: manual rotation / policy-driven

## 5. Memory risks & mitigations

| Risk | Mitigation |
|---|---|
| **Drift** | Revalidate semantic facts on ingest (ingest pipeline includes checksum + source date). Keep TTLs conservative and surface a "last-validated" timestamp in the audit footer. Use periodic re-probes (monthly) to detect drift.
| **Poisoning** | Protect ingestion: require signed/verified packs or manual approval for new fixtures. Grade retrieved documents and require corroboration across two sources before trusting critical metrics. Maintain an immutable audit log of ingests and changes.
| **Staleness** | Use TTLs above; on missing recent updates trigger a short re-retrieval workflow (retry window). Flag stale facts in drafts and escalate if a claim depends on stale data.
| **Confidential / retention** | Preserve confidentiality flags in the roadmap snapshot and enforce redact-on-write for embargoed items. Retain PII only with retention policy and automatic purge cron jobs; surface retention status in the audit footer.

## Done-check mapping

- Each memory type has a scope and TTL.
- All four risks are mapped to concrete mitigations and where they apply in the pipeline.

✅ Next: Step 4 (the grounding probe) — we will run a grounded happy-path run and a withheld-source probe, capture outputs, and save them to `06-autonomy/prototype.md`.
