# Architecture

## Purpose

Design a security operations data plane that turns a slow, manual, "we'll get back to you in a day or two" cross-source security question into a fast, trustworthy, agent-actionable answer, on a stack that mirrors a real Microsoft-SOC-on-multicloud enterprise, built entirely on free tiers.

The design goal is not maximum features. It is the smallest end-to-end slice that proves the load-bearing claim: **normalization and schema discipline are the prerequisite for everything downstream, including agents.** Get that right and the fast answer falls out. Skip it and you have a faster way to be wrong.

---

## The five responsibilities

A data plane is not one thing. It is five responsibilities that must not be conflated, because each fails differently.

### 1. Ingest

Pull raw events from heterogeneous sources into a landing zone without losing fidelity. The landing zone keeps the raw record verbatim. We never normalize on the way in and throw the original away, because the first time the normalization is wrong, the raw record is the only thing that lets you recover.

Sources in this build:
- **AWS CloudTrail** (control-plane API calls) and **VPC flow logs** (network), the non-Microsoft cloud signal.
- **Entra ID sign-in and audit logs**, the identity signal, from the E5 tenant.
- **Endpoint signal** from Defender, in the same tenant.
- **Synthetic "subsidiary" logs** in a deliberately different shape, the stand-in for the un-migrated acquisition that nobody has normalized yet. This is the hard case, and it is the point.

### 2. Normalize

Map every raw event to a single canonical schema. This is the heart of the system. We align to **ASIM** (Microsoft's Advanced Security Information Model) rather than invent a schema, because the target SOC is Sentinel and ASIM is what Sentinel's content already speaks. Inventing our own schema would mean re-mapping again at the SOC boundary. Borrow the standard that already exists downstream.

Normalization is where "nobody knows what it is" gets solved: after this layer, every event has a known type, a known set of fields, and a known provenance. A question can be asked against the canonical shape without knowing which of seven systems the data came from.

### 3. Correlate

Resolve the same real-world entity across sources. An AWS IAM principal, an Entra user, an endpoint device, and a subsidiary's local account may all be the same human or the same actor. Correlation builds the identity graph that lets a single question span sources. This is the part that directly attacks "individually talk to people across seven organizations." The graph is the institutional knowledge, made queryable.

This responsibility leans directly on identity-graph work from real-time fraud systems: keeping track of who the actor is, who the account holder is, who the device is, and stitching them in near real time. The security version is the same problem with different entities.

### 4. Validate

Check data quality at **both** ends of the pipeline. Pre-ingest: is the incoming record complete and well-formed, or is a required field empty where it should not be. Post-normalization: did we produce what we expected, with no funky values in fields and no silent drops. You can build a robust pipeline and still emit garbage if you only validate one end. This is non-negotiable and it is where most data platforms quietly rot.

This is also where the build-vs-buy lesson lives: where an off-the-shelf data-quality tool fits the pipeline shape, use it; where it does not, the validation logic is small enough to own, and owning it keeps it honest to the actual data contract.

### 5. Serve / guardrail

Land the normalized, correlated, validated signal somewhere query-ready (Sentinel / Log Analytics, queried with KQL), and expose it to agents under hard guardrails: bounded tools, least-privilege identity, audit trail, and a human-approval boundary for any consequential action. The agent reasons over the **same** trustworthy graph the analyst sees. It does not get a privileged side channel.

---

## Data flow

```
                          ┌──────────────────────────────────────────────────────────┐
                          │                     DATA PLANE                            │
                          │                                                          │
  ┌───────────────┐       │   ┌─────────┐    ┌───────────┐    ┌───────────┐         │     ┌──────────────────┐
  │ AWS CloudTrail│──────►│   │ INGEST  │───►│ NORMALIZE │───►│ CORRELATE │────────►│────►│ Sentinel / LogAnl│
  │ AWS VPC flow  │──────►│   │ (landing│    │ (→ ASIM)  │    │ (identity │         │     │  (KQL query)     │
  │ Entra logs    │──────►│   │  zone,  │    │           │    │  graph)   │         │     └────────┬─────────┘
  │ Defender XDR  │──────►│   │  raw    │    └─────┬─────┘    └─────┬─────┘         │              │
  │ Synthetic sub │──────►│   │  kept)  │          │                │               │              ▼
  └───────────────┘       │   └─────────┘          ▼                ▼               │     ┌──────────────────┐
                          │                 ┌──────────────────────────────┐       │     │ Defender portal  │
                          │                 │ VALIDATE (pre + post)        │       │     │ analyst + agent  │
                          │                 └──────────────────────────────┘       │     │ (Security Copilot│
                          │                                                          │     │  under guardrails)│
                          │   Entra identity = signal source AND least-privilege     │     └──────────────────┘
                          │   guardrail for the agent (spans the whole plane)        │
                          └──────────────────────────────────────────────────────────┘
```

Two things about this diagram matter most:

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

Stating the boundary is part of the design. Scope honesty is a feature.

- **It does not run a SOC.** No 24x7, no on-call, no detically engineered detections beyond what proves the data path. The platform builds the road; operators drive.
- **It does not do vulnerability management.** Attack-path / toxic-combination reasoning is a real and adjacent capability, but in the target org VM is a separate function. This build stays in the data-plane and SOC lane: cross-source identity correlation for detection and response.
- **It is not multi-region, HA, or production-hardened.** It is a faithful slice, sized to prove the architecture, not to carry load.
- **It does not let the agent act unsupervised.** Every consequential action crosses a human-approval boundary. The agent triages and proposes; a human (for now) commits.

These are not gaps to apologize for. They are the line between a credible slice and an over-claimed monument.

---

## The sequence (why this order)

The order of construction is itself a design decision, because it determines when the thing becomes demonstrable.

1. **One source + landing zone + the canonical schema.** Prove a single raw event becomes a clean canonical event. Smallest possible loop.
2. **Validation on that loop.** Before adding breadth, prove we can catch a malformed event. Quality discipline before scale.
3. **Second, differently-shaped source (the synthetic subsidiary).** Now normalization earns its keep: two shapes, one canonical output.
4. **Correlation across the two sources.** The identity graph stitches them. This is the moment the cross-source question becomes answerable.
5. **Land in Sentinel, write the KQL.** The "seconds" claim becomes literally true and demonstrable.
6. **Wrap the agent with guardrails.** Only now, on trustworthy data, does the agent layer make sense.

Each step is demonstrable on its own. If time runs out at step 4, there is still a real, honest thing to show. That is the point of sequencing by demonstrability rather than by component.
