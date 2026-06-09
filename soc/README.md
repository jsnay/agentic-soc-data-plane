# soc/

Land the normalized, correlated, validated signal in Sentinel and express the cross-source question as KQL.

## Responsibility

The serve side. Make the canonical events query-ready in the actual SOC platform, and demonstrate the "answered in seconds" question literally returning an answer.

## Components (planned)

- `ingest_to_sentinel.py` — push canonical events into a Log Analytics custom table (via the Logs Ingestion API / DCR). Mirrors how a real data plane lands signal where the SOC can act.
- `queries/cross_source_actor.kql` — the threat question from [`../docs/03-threat-question.md`](../docs/03-threat-question.md), as runnable KQL over the canonical table, including the `actor_confidence` floor.
- `queries/normalization_coverage.kql` — a meta query: for a given question, which sources have trustworthy normalized data, and which do not. The "do we even know what we have" view.
- `notes/defender-copilot.md` — how the Security Copilot Analyst Agent and the unified Defender portal fit on top: triage, suppression, case management, all reasoning over the same canonical graph.

## The KQL is the punchline

The cross-source query is short. That is the point: all the expensive work moved left into the data plane and got done once, so the question itself is cheap. The query returns one row, cross-source, with provenance, above the confidence floor, in sub-second time, where today the same answer takes one to two days of human coordination.

## Free-tier note

Sentinel's first 10 GB/day is free for 31 days, and Defender/Entra audit logs ingest free in perpetuity. The demo volume is tiny. See `../infra/README.md`.
