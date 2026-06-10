# Architecture

## Purpose

Prototype a security operations data plane that enables fast, trustworthy, agent-actionable answers to critical security questions, on a stack that mirrors a real Microsoft-stack SOC for an enterprise multicloud environment. Built entirely on free tiers.

The design goal is not maximum features. It is the smallest end-to-end slice that proves the load-bearing claim: **normalization, schema discipline, and semantic clarity are the prerequisite for everything downstream, including agents.** Get that right and the enterprise is able to be quickly secured and answers are available at operator fingertips. Skip it and you have a faster way to be wrong.

---

## The five responsibilities

A data plane has five responsibilities.

### 1. Ingest

Pull raw events from heterogeneous sources into a landing zone without losing fidelity. The landing zone keeps the raw record verbatim. We never normalize on the way in and throw the original away, because when the normalization goes wrong, the raw record is the only thing that lets you recover.

Sources in this build:
- **AWS CloudTrail** (control-plane API calls) and **VPC flow logs** (network), the non-Microsoft cloud signal.
- **Entra ID sign-in and audit logs**, the identity signal, from the E5 tenant.
- **Endpoint signal** from Defender, in the same tenant.
- **Synthetic "subsidiary" logs** in a deliberately different shape, the stand-in for un-migrated acquisitions that nobody has normalized yet. This is the hard case, and it is the point.

### 2. Normalize

Map every raw event to a single canonical schema. This is the heart of the system. We align to **ASIM** (Microsoft's Advanced Security Information Model) rather than invent a schema, because the target SOC is Sentinel and ASIM is what Sentinel's content already speaks. Inventing our own schema would mean re-mapping again at the SOC boundary. Borrow the standard that already exists downstream. In practice, there may be an existing data plane that would either need to be migrated, or settle on normalizing to that schema and building a conversion pipeline layer to ASIM.

Normalization is where we solve the "nobody knows what the data is" gets solved: after this layer, every event has a known type, a known set of fields, and a known provenance. A question can be asked against the canonical shape without knowing which of many systems the data came from.

### 3. Correlate

Resolve the same real-world entity across sources. An AWS IAM principal, an Entra user, an endpoint device, and a subsidiary's local account may all be the same human or the same actor. Correlation builds the identity graph that lets a single question span sources. This is the part that directly solves needing to individually talk to people across multiple organizations to answer questions. The graph is the institutional knowledge, made queryable.

This responsibility leans directly on my identity-graph work from real-time fraud detection systems: keeping track of who the actor is, who the account holder is, who the device is, and stitching them in near real time. The security version is the same problem with different entities.

### 4. Validate

Check data quality at **both** ends of the pipeline. Pre-ingest: is the incoming record complete and well-formed, or is a required field empty where it should not be. Post-normalization: did we produce what we expected, with no funky values in fields and no silent drops. You can build a robust pipeline and still emit garbage if you only validate one end. This is non-negotiable because data quality is paramount.

This is also where the build-vs-buy lesson lives: where an off-the-shelf data-quality tool fits the pipeline shape, use it; where it does not, the validation logic is small enough to own, and owning it keeps it honest to the actual data contract.

### 5. Serve / guardrail

Land the normalized, correlated, validated signal somewhere query-ready (Sentinel / Log Analytics, queried with KQL), and expose it to agents under hard guardrails: bounded tools, least-privilege identity, audit trail, and a human-approval boundary for any consequential action. The agent reasons over the **same** trustworthy graph the analyst sees. It does not get a privileged side channel.

---

## Data flow

```
                      ┌────────────────────────────────────────────────────┐
                      │                     DATA PLANE                     │
                      │                                                    │
  ┌───────────────┐   │   ┌─────────┐   ┌───────────┐   ┌───────────┐      │   ┌───────────────────┐
  │ AWS CloudTrail│──►│   │ INGEST  │──►│ NORMALIZE │──►│ CORRELATE │─────►│──►│ Sentinel / LogAnl │
  │ AWS VPC flow  │──►│   │ (landing│   │ (to ASIM) │   │ (identity │      │   │  (KQL query)      │
  │ Entra logs    │──►│   │  zone,  │   │           │   │  graph)   │      │   └────────┬──────────┘
  │ Defender XDR  │──►│   │  raw    │   └─────┬─────┘   └─────┬─────┘      │            │
  │ Synthetic sub │──►│   │  kept)  │         │               │            │            ▼
  └───────────────┘   │   └─────────┘         ▼               ▼            │   ┌───────────────────┐
                      │                ┌────────────────────────────┐      │   │ Defender portal   │
                      │                │ VALIDATE (pre + post)      │      │   │ analyst + agent   │
                      │                └────────────────────────────┘      │   │ (Security Copilot │
                      │                                                    │   │  under guardrails)│
                      │ Entra identity = signal source AND least-privilege │   └───────────────────┘
                      │ guardrail for the agent (spans the whole plane)    │
                      └────────────────────────────────────────────────────┘
```

Two things about this diagram:

1. **Validate spans normalize and correlate**, it is not a final step. Bad data caught early is cheap; bad data caught in the SOC is an incident postmortem.
2. **Entra identity is both an input and a control.** It is a detection signal flowing in, and it is the mechanism that keeps the agent bounded. The same identity fabric that tells you who did something also decides what the agent is allowed to do. That dual role is the cleanest place to enforce least privilege.

---

## Infrastructure mapping (all free tier)

| Layer | Real target environment | This build | Cost |
|---|---|---|---|
| SOC / SIEM | Microsoft Sentinel | Sentinel on a Log Analytics workspace | First 10 GB/day free for 31 days; Defender + Entra audit logs free in perpetuity |
| Identity | Microsoft Entra | Entra ID in the E5 dev tenant | Free with E5 dev tenant |
| Detection | Microsoft Defender XDR | Defender in the E5 dev tenant | Free with E5 dev tenant |
| Agentic layer | Security Copilot + Analyst Agent | Security Copilot (now bundled in E5 as of the Apr–Jun 2026 rollout) + Agent Builder / MCP | Included with E5 (400 SCU / 1,000 users) |
| Non-MS cloud source | AWS-primary enterprise | AWS free tier (CloudTrail, VPC flow, a Lambda or two) | Free tier |
| Un-migrated subsidiary | The hard normalization case | Synthetic log emitter, deliberately off-schema | Local, free |
| Data plane compute | The role's platform | Local Python (and/or a free-tier container) | Local, free |

The E5 developer sandbox is a 90-day tenant that auto-renews under active development, with 25 user licenses, Entra, Defender, Purview, and now Security Copilot. That single free tenant reproduces the entire Microsoft SOC side of the target environment. AWS free tier reproduces the source side. The synthetic subsidiary reproduces the actual hard problem. Nothing here requires spend.

Full setup steps and the guardrail that keeps credentials out of git are in [`../infra/README.md`](../infra/README.md).

---

## What this build deliberately does NOT do

Scope limitation is a feature.

- **It does not run a SOC.** No 24x7, no on-call, no detically engineered detections beyond what proves the data path. The platform builds the road; operators drive.
- **It does not do vulnerability management.** Attack-path / toxic-combination reasoning is an adjacent capability. For our purposes we assume VM is a separate function. This build stays in the data-plane and SOC lane: cross-source identity correlation for detection and response.
- **It is not multi-region, HA, or production-hardened.** It is a faithful slice, sized to prove the architecture thinking.
- **It does not let the agent act unsupervised.** Every consequential action crosses a human-approval boundary. The agent triages and proposes; a human (for now) commits.

---

## The sequence (why this order)

The order of construction is itself a design decision, because it determines when the thing becomes demonstrable.

1. **One source + landing zone + the canonical schema.** Prove a single raw event becomes a clean canonical event. Smallest possible loop.
2. **Validation on that loop.** Before adding breadth, prove we can catch a malformed event. Quality before scale.
3. **Second, differently-shaped source (the synthetic subsidiary).** Normalization proved out: two shapes, one canonical output.
4. **Correlation across the two sources.** The identity graph stitches them. Cross-source questions become answerable.
5. **Land in Sentinel, write the KQL.** Able to query and answer questions in seconds.
6. **Wrap the agent with guardrails.** Can we make the agent layer make sense.

