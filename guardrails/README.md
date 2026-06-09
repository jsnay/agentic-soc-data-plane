# guardrails/

The agentic contract. How an agent acts on the correlated signal safely.

## Responsibility

Let an agent triage and propose at machine speed, without giving it the ability to do harm. This is the genuinely defensible part of the agentic story and the part that is actually ours to claim.

## The four hard constraints

See the full design in [`../docs/03-threat-question.md`](../docs/03-threat-question.md). Summary:

1. **Bounded tools.** Fixed, small tool set (query the canonical store, read the identity graph, draft an incident, propose an action). No arbitrary API access. The tool list is the blast radius.
2. **Least-privilege identity (via Entra).** The agent runs as its own Entra identity with minimum permissions, read-mostly, no standing ability to change production state.
3. **Audit trail.** Every action proposed or taken, every tool call, every datum read, logged with the same provenance discipline as the data plane.
4. **Human-approval boundary.** The agent triages, enriches, suppresses, proposes. A human commits any consequential action. The boundary moves only deliberately, one action type at a time, as the audit record earns it.

## Components (planned)

- `agent_contract.md` — the formal statement of the four constraints and the action taxonomy (what the agent may propose vs. what requires human commit).
- `tools.py` — the bounded tool definitions, exposed via MCP so the agent reasons over the same security graph as the analyst.
- `audit.py` — the agent audit logger; every call recorded with inputs, outputs, and the data read.
- `policy.py` — the least-privilege identity mapping and the approval-boundary gate.

## The principle

Treat the agent as a nondeterministic producer, exactly like a human developer, and put the same guardrails and checks around it. The newness of the technology does not change the discipline; we have governed nondeterministic producers for decades. An agent earns autonomy the way a junior engineer does: by being right, visibly, in the audit record, over time. Machine speed without guardrails is just a faster way to lose control.
