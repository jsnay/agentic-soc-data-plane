# The Threat Question

The single question the whole build exists to answer fast. We'll pick one question, make it take seconds, and the architecture has proven itself.

---

## The question

> **"Is this actor doing something suspicious across more than one system at once, and is it the same actor in each place?"**

Concretely, the demo version:

> A failed-then-successful authentication pattern on an identity (Entra), followed within minutes by an unusual data-access call in AWS (CloudTrail) against a sensitive store, where the AWS principal and the Entra user resolve to **the same person**, and one of the involved identifiers comes from the un-migrated subsidiary (`northstar`) that nobody has normalized before.

This is a real, ordinary, cross-source detection question. It is not exotic. That is exactly why it is the right one: it is the kind of question that *should* take seconds and today takes a day or two.

### Why this question

- It hits the stated pain head-on: "is this threat occurring across the company" answered by "we'll get back to you in one to two days." This makes that question answerable now.
- It requires **all five data-plane responsibilities** to work: ingest (three sources), normalize (different shapes), validate (or the answer is garbage), correlate (same actor across sources), serve (query in seconds). A simpler question would not exercise the architecture.
- It rests on genuine prior experience: real-time identity graphs deciding who the actor is across signals, under latency pressure. The security framing is the same technique.
- It is squarely data-plane and SOC work. It is not vulnerability management. It stays in the lane the role actually owns.

---

## The day-vs-seconds comparison

This table is the demo's punchline. It is the before and after.

| Step | Today (manual, 1–2 days) | With the data plane (seconds) |
|---|---|---|
| Find the auth anomaly | Pull Entra logs, eyeball or hand-query | Already normalized, one KQL predicate |
| Find the AWS access | Different team, different console, different query language | Same canonical schema, same query |
| Confirm it's the same person | Email someone who owns the subsidiary's identity mapping; wait; get "that's the wrong data"; ask again | Identity graph resolves `northstar` UID + AWS principal + Entra UPN to one entity, with confidence and basis attached |
| Decide if it's actionable | Reassemble by hand, hope nothing was missed | One result row, cross-source, with provenance to every source record |
| Elapsed | Hours of human coordination across orgs | Sub-second query; the agent can propose the triage |

The point is not that the query is clever. The point is that **all the slow work moved left, into the data plane, and got done once, so the question is now cheap.** That is what a platform does: it makes the expensive thing cheap by paying the cost once, upstream, with discipline.

---

## How it's expressed

Against the canonical store in Sentinel, as KQL over the normalized schema. Sketch (the real query lives in [`../soc/`](../soc/)):

```kql
// 1. auth anomaly: fail then success on one resolved entity, short window
let authAnomalies =
    CanonicalEvents
    | where event_type == "Authentication"
    | summarize fails = countif(event_result=="Failure"),
                wins  = countif(event_result=="Success"),
                t = min(event_time)
        by actor_resolved_entity_id, bin(event_time, 10m)
    | where fails >= 3 and wins >= 1;
// 2. sensitive AWS access by the SAME resolved entity, soon after
let sensitiveAccess =
    CanonicalEvents
    | where source_system == "aws.cloudtrail"
    | where target_resolved_asset_id in (SensitiveAssets)
    | project actor_resolved_entity_id, access_time = event_time, target_src_identifier;
// 3. join on the RESOLVED entity, require a real cross-source link
authAnomalies
| join kind=inner sensitiveAccess on actor_resolved_entity_id
| where access_time between (t .. t + 15m)
| where actor_confidence >= 0.8         // refuse to act on a weak identity merge
| project actor_resolved_entity_id, t, access_time, target_src_identifier, actor_confidence
```

The `actor_confidence >= 0.8` line is the whole philosophy in one predicate: the cross-source answer is only as trustworthy as the identity link, and the query refuses to surface a result built on a weak merge. The threshold is a choice the analyst or guardrailed agent makes explicitly, not a hidden default.

---

## Demo script (5 minutes)

1. **Show the pain.** "Today, answering this means coordinating across the identity team, the AWS team, and whoever owns the subsidiary's accounts. One to two days." Show the three raw logs in three different shapes.
2. **Show the data plane do its job.** Run ingest → normalize → correlate on the three sources. Show the `northstar` ugly log become a clean canonical event. Show the identity graph stitch the three identifiers into one entity, with confidence and basis.
3. **Ask the question.** Run the KQL. One row comes back: same actor, cross-source, within the window, above the confidence floor, with provenance to every source record.
4. **Show the guardrail.** The agent proposes a triage action (e.g., "raise an incident, suggest disabling the session"). It stops at the human-approval boundary. Show the audit trail entry.
5. **Show the honesty.** Open the decision record. "Here is what I knew, here is what I assumed, here is where I would confirm with you." Turn it into a design review.

---

## Guardrails (the agentic contract)

Lives in [`../guardrails/`](../guardrails/). Summary of the design:

The agent that triages the correlated signal operates under four hard constraints. These are the genuinely defensible part of the agentic story, and the part that is actually ours to claim.

1. **Bounded tools.** The agent has a fixed, small set of tools (query the canonical store, read the identity graph, draft an incident, propose an action). It cannot call arbitrary APIs. The tool list is the blast radius.
2. **Least-privilege identity (via Entra).** The agent runs as its own Entra identity with the minimum permissions to do its job, read-mostly, with no standing ability to change production state. The same identity fabric that is a detection signal is the control that bounds the agent. You do not give the junior dev the ability to take down the environment; you do not give the agent one either.
3. **Audit trail.** Every agent action, proposed or taken, every tool call, every piece of data it read, is logged with the same provenance discipline as the data plane itself. An agent action is as traceable as a normalized event.
4. **Human-approval boundary.** The agent triages, enriches, suppresses noise, and proposes. A human commits any consequential action. The boundary is explicit in the design, not an afterthought. As trust accrues and the audit record proves the agent's judgment, the boundary can move, deliberately, one action type at a time, never wholesale.

The principle underneath all four: **treat the agent as a nondeterministic producer, exactly like a human developer, and put the same guardrails and checks around it that you would around a person who can move fast and occasionally be wrong.** The newness of the technology does not change the discipline. We have governed nondeterministic producers for decades. The audit trail and the approval boundary are how an agent earns autonomy the same way a junior engineer does: by being right, visibly, over time.
