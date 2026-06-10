# Agentic SOC Data Plane

A working reference build of a **security operations data plane**: the layer that pulls security signal out of a messy, multi-source enterprise, normalizes it with schema discipline, and lands it query-ready in a cloud-native SOC and actioned safely by agents under guardrails.

---

## The problem this solves

In most large enterprises, especially ones grown through acquisition, the security data situation looks like this:

> There is a lot of data, and nobody quite knows what it is. When a leader asks "is this threat occurring, can we determine X across the company," the answer is "we'll get back to you," followed by one to two days of research.

The logs are usually already centralized. That part is often solved. The hard part is **normalization**: different subsidiaries, clouds, and tools each describe the same event, the same identity, the same asset differently. Until that is fixed, every cross-source question is a manual archaeology project.

Meanwhile the threat clock has changed. Frontier-model-assisted attackers compress discovery-to-exploit to hours. A human-paced, "get back to you tomorrow" operation is the wrong shape for a machine-paced adversary. The defender side has to move to machine speed, which means the data underneath has to be trustworthy enough for agents to reason over it without a human babysitting every step.

So the platform's job is twofold: **make the data fast and trustworthy first, then let agents act on it within hard guardrails.** In that order. Agents on top of un-normalized data just produce confident nonsense faster.

---

## The shape

What I aim to build/prototype as I think through the architecture.
```
  SOURCES (multi-cloud/subsidiary)  DATA PLANE (this build)             SOC PLATFORM             HUMANS + AGENTS
  ┌────────────────────────────┐    ┌──────────────────────────────┐    ┌────────────────────┐   ┌────────────────────┐
  │ AWS CloudTrail / VPC flow  │    │ Ingest                       │    │ Sentinel (SIEM)    │   │ Analyst in portal  │
  │ Endpoint / identity (Entra)│──► │ Normalize (schema discipline)│──► │ Defender XDR       │──►│ Security Copilot   │
  │ Synthetic "subsidiary" logs│    │ Correlate (identity graph)   │    │ KQL over Log Anlyt.│   │ agent (guardrailed)│
  │ (the un-migrated hard case)│    │ Validate (pre + post)        │    └────────────────────┘   └────────────────────┘
  └────────────────────────────┘    └──────────────────────────────┘

  Entra identity = signal source + least-privilege guardrail for agents
```

The data plane is the middle column. It does not run the SOC. It builds the road the SOC drives on.

---

## What works today

> Filled in as the build progresses. This section is the honest scoreboard.

- [ ] Synthetic + AWS-origin telemetry emitting into a local landing zone
- [ ] Normalization engine mapping raw events to a single canonical schema (ASIM-aligned)
- [ ] Pre-ingest and post-ingest validation (you can build a robust pipeline and still produce garbage if you do not validate both ends)
- [ ] Cross-source identity correlation: resolve the same actor across AWS, endpoint, and identity sources
- [ ] The "answered in seconds" question, demonstrated against the normalized store
- [ ] Sentinel landing + a KQL query that returns the answer
- [ ] Guardrail design for an agent that triages the correlated signal (bounded tools, least privilege, audit trail, human approval boundary)

---

## How to read this repo

If you have a few minutes, read this README and [`docs/00-architecture.md`](docs/00-architecture.md).

If you want to see how the thinking was done, read [`docs/01-decision-record.md`](docs/01-decision-record.md). That is the knowns / assumptions / choices / trade-offs log. It is the most important document here. It shows my reasoning, not just the result.

| Path | What's in it |
|---|---|
| [`docs/00-architecture.md`](docs/00-architecture.md) | System design, component responsibilities, the free-tier infrastructure mapping |
| [`docs/01-decision-record.md`](docs/01-decision-record.md) | Every meaningful choice, what it cost, what was rejected and why |
| [`docs/02-data-model.md`](docs/02-data-model.md) | The canonical event schema and the identity-correlation design |
| [`docs/03-threat-question.md`](docs/03-threat-question.md) | Demo script - Identifying answers to critical security questions quickly, agentic enablement |
| `ingest/` | Source emitters: AWS CloudTrail/flow logs, synthetic subsidiary logs |
| `normalize/` | The schema-discipline engine. The heart of the build. |
| `correlate/` | Cross-source identity resolution |
| `soc/` | Sentinel wiring, KQL, Defender/Copilot integration notes |
| `guardrails/` | Agent least-privilege, audit trail, human-approval boundary design |
| `infra/` | Tenant + AWS setup notes (no secrets, ever) |

---

## Why it's built on this stack

The target environment this is modeled on runs a Microsoft-centric SOC (Sentinel, Defender XDR, Security Copilot, Entra identity) sitting on top of a multicloud enterprise that is primarily AWS, with subsidiaries in varying states of integration. This build mirrors that: a free Microsoft E5 developer tenant for the SOC side, AWS free tier for the non-Microsoft source side, and a synthetic "un-migrated subsidiary" to stand in for the hard normalization case. Full infrastructure and cost notes are in [`infra/README.md`](infra/README.md).

---

## Status

Active build. This is a personal engineering project exploring how a security operations data plane should be designed when the adversary moves at machine speed. The architecture and decision record are stable; the running components land incrementally.
