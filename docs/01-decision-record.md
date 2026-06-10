# Decision Record

The reasoning behind the project. This captures how my thinking was done.

Structured in three parts:
1. **What we know** — facts about the target environment, stated as facts.
2. **What we're assuming** — inferences we are acting on but have not confirmed. Flagged so they can be corrected.
3. **The choices** — each meaningful decision as a short ADR: context, decision, trade-off, what was rejected.

The discipline here is the point. It's okay not having the right answer on day one. Ownership of the analysis that defends decisions, with what's known vs. assumed.

---

## Part 1 — What we know

These are grounded facts about the class of environment this build models. Where a fact came from direct conversation it is treated as confirmed; where it came from public sources it is cited as such.

| # | Known | Source |
|---|---|---|
| K1 | The SOC is Microsoft-centric: Sentinel as SIEM/SOAR, Defender XDR for detection, Security Copilot for agentic triage, Entra for identity. | Direct (hiring-manager-adjacent conversations) |
| K2 | The SOC recently migrated off Splunk onto Sentinel. Teams are on the ADX/KQL learning curve; the mindset shift is from "send everything and search" to "send the right things." | Direct |
| K3 | Security logs are **already centralized** across subsidiaries. Collection is a solved baseline. The hard part is **normalizing** across different technologies. | Direct |
| K4 | Subsidiary integration is uneven. One large acquired entity is **not** migrated the same way and is the hard case. | Direct |
| K5 | The enterprise estate is multicloud: primarily AWS for compute/containers, plus GCP (BigQuery/Vertex) and Azure for specialized AI, plus an on-prem XML tail. | Public (CIO interview) |
| K6 | The core pain: "a lot of data, nobody knows what it is." A cross-source question that should take seconds takes one to two days of manual research across multiple orgs. | Direct |
| K7 | Success is defined as moving **off manual processes** toward automation ("things that don't require people to do the right thing, it just happens automatically") and not getting breached. | Direct |
| K8 | The why-now is the machine-speed-adversary shift. The defender side must move to machine speed because the attacker side already has. This is the top initiative driving the org refresh. Anthropic Mythos anxiety. | Direct + public |
| K9 | Vulnerability management is a **separate** org, deliberately pulled out from under the SOC. | Direct |

## Part 2 — What we're assuming

Stated as assumptions on purpose. Each is actionable but unconfirmed. In the room, these become questions, not assertions.

| # | Assumption | Why we're acting on it | How we'd confirm |
|---|---|---|---|
| A1 | The "log query language" referenced generically is KQL (Kusto), since the SOC is Sentinel/Log Analytics. | Sentinel and Azure Monitor both run on KQL. | Ask which query surface analysts actually use day to day. |
| A2 | ASIM is the right normalization target because the SOC is Sentinel. | Sentinel's built-in content speaks ASIM; mapping to anything else means re-mapping at the SOC boundary. | Confirm whether existing detections are ASIM-based or custom. |
| A3 | The identity correlation problem is real and currently manual. | "Talk to people across seven orgs, three say wrong data" is an identity/provenance problem at root. | Ask how cross-source entity resolution is done today. |
| A4 | Subsidiaries have genuinely different log shapes, not just different volumes. | Acquisition-grown estates almost always do. | Ask for two real sample schemas from two subsidiaries. |
| A5 | The agentic layer is wanted but not yet trusted on the data. | Stated desire for agents + stated data-quality pain = agents can't be trusted yet. | Ask what would have to be true before an agent is allowed to action without review. |

---

## Part 3 — The choices (ADRs)

### ADR-001: Build a working slice, not a slide deck

**Context.** The capability could be communicated as a design document alone. The question is whether to also build a running slice.

**Decision.** Build a running, end-to-end slice on free tiers.

**Trade-off.** Costs real engineering time before it can be shown. A design alone would be faster to produce.

**Why.** A running slice converts the entire conversation from "here is what I would do" to "here is what I did, and here is what I learned doing it, including where my assumptions were wrong." The second conversation is unbeatable. The time cost is the price of that shift, and it is worth it.

**Rejected.** Design-only. It leaves the implementer without the value of learning hands-on.

---

### ADR-002: Normalize to ASIM, do not invent a schema

**Context.** Every event needs a canonical shape. We could design our own or adopt an existing model.

**Decision.** Align the canonical schema to ASIM (Advanced Security Information Model).

**Trade-off.** ASIM is opinionated and broad; we only need a slice of it, so we carry some conceptual overhead we will not fully use. A bespoke minimal schema would be smaller.

**Why.** The SOC is Sentinel. ASIM is what Sentinel's content already speaks. A bespoke schema would have to be re-mapped to ASIM at the SOC boundary anyway, so we would do the normalization work twice and own a translation layer forever. Adopt the standard that already exists downstream.

**Rejected.** Bespoke schema (re-mapping debt). OCSF (vendor-neutral and excellent, but it would add a translation hop to a Sentinel-native SOC; noted as the choice we would make in a vendor-neutral or multi-SIEM environment).

---

### ADR-003: Keep the raw record; normalize as a derived layer

**Context.** Normalization could transform events in place, or keep raw and produce a normalized derivative.

**Decision.** Landing zone keeps the raw record verbatim. Normalized events are a derived layer that points back to the raw.

**Trade-off.** Storage cost roughly doubles for the overlap, and the pipeline is slightly more complex.

**Why.** The first time a normalization rule is wrong, the raw record is the only thing that lets you recover without re-ingesting. It also makes the pipeline auditable: every normalized field can be traced to its source bytes. In security, provenance is not optional.

**Rejected.** Transform-in-place (cheaper, unrecoverable when wrong).

---

### ADR-004: Validate at both ends, build it ourselves where the shape demands

**Context.** Data quality can be checked pre-ingest, post-normalization, both, or delegated to an off-the-shelf DQ tool.

**Decision.** Validate at both ends. Use an off-the-shelf validator where it fits the data contract; own the validation logic where the pipeline shape does not match the tool.

**Trade-off.** Owning validation is maintenance the team carries. A purchased tool would be someone else's maintenance.

**Why.** A robust pipeline can still emit garbage if only one end is checked. And off-the-shelf DQ tools assume a pipeline shape; when the real shape differs, the tool either blocks legitimate data or waves through bad data. The validation rules that match the actual contract are small enough to own and too important to mis-fit. This is build-vs-buy decided on the merits of the data shape, not on vendor pull.

**Rejected.** Single-ended validation (fails silently). Blanket purchase of a DQ tool regardless of fit (mis-fit risk).

---

### ADR-005: Identity correlation as a graph, grounded in real-time fraud experience

**Context.** Cross-source entity resolution could be done as periodic batch joins or as a maintained identity graph.

**Decision.** Maintain an identity graph that stitches entities across sources, resolvable at query time.

**Trade-off.** A graph is more to build and keep current than a nightly join.

**Why.** The cross-source question has to be answered in seconds, not after the next batch job. A maintained graph makes resolution a lookup, not a recompute. This directly reuses real-time identity-graph patterns from fraud detection: tracking who the actor, account holder, and device are, and stitching them as signal arrives. The security entities differ; the technique is the same one already proven under low-latency load.

**Rejected.** Batch joins (too slow for the "seconds" mandate; recomputes instead of maintaining).

---

### ADR-006: Free-tier Microsoft E5 dev tenant + AWS free tier + synthetic subsidiary

**Context.** The build needs infrastructure that mirrors the target environment without spend.

**Decision.** E5 developer sandbox for the entire Microsoft SOC side (Sentinel, Defender, Entra, Security Copilot), AWS free tier for the non-Microsoft source, a synthetic emitter for the un-migrated subsidiary.

**Trade-off.** Dev tenants and free tiers have limits (ingestion caps, license counts, the 90-day renewal). Not production scale.

**Why.** This trio reproduces the exact architecture: Microsoft SOC, AWS-primary multicloud source, and the hard normalization case, at zero cost. The synthetic subsidiary is arguably better than a real one for a demo because we control its off-schema-ness to make the normalization point cleanly.

**Rejected.** Paid Azure/AWS (unnecessary spend for a slice). All-synthetic with no real cloud (loses the genuine cross-cloud normalization proof).

---

### ADR-007: Stay in the data-plane / SOC lane, not vulnerability management

**Context.** The most visually impressive demo would be attack-path / toxic-combination reasoning. It is also adjacent to a separate org.

**Decision.** Build cross-source identity correlation for detection and response. Do not build VM.

**Trade-off.** Identity correlation is less flashy than a crown-jewels attack-path graph.

**Why.** VM is deliberately a separate function in the target org (K9). Building a VM demo would signal either not understanding the org boundary or reaching across it. Identity correlation is squarely the data plane's job, hits the actual stated pain, and rests on genuine prior experience.

**Rejected.** Toxic-combination/attack-path demo (org-boundary risk, and it implies a VM mandate this role does not have).

---

### ADR-008: Agent acts only on validated data, only under guardrails, only up to a human boundary

**Context.** The agentic layer could be given broad autonomy to maximize the "machine speed" story.

**Decision.** The agent reasons over the same validated graph the analyst sees, with bounded tools, least-privilege identity (via Entra), a full audit trail, and a human-approval boundary for any consequential action.

**Trade-off.** A human-in-the-loop boundary caps how "autonomous" the demo looks.

**Why.** Treat the agent as a nondeterministic producer, the same way you treat human developers who are also nondeterministic, and put the same kind of guardrails and checks around it. An agent on un-validated data produces confident mistakes faster. An agent with unbounded tools is an incident. The guardrail design is the genuinely defensible part of the agentic story, and it is the part that is actually ours to claim. Machine speed without guardrails is just a faster way to lose control. Future would include graduated autonomy levels for agentic roles.

**Rejected.** Broad autonomy (impressive until the first wrong action; wrong shape for a regulated environment).

---
