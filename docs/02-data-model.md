# Data Model & Identity Correlation

The canonical schema and the identity graph. This is where "nobody knows what it is" gets solved.

---

## The canonical event

Every raw event from every source becomes one canonical event with a known shape. We align to **ASIM** field names so the SOC (Sentinel) consumes it natively, but we keep the model small: only the fields the demo needs, extend later.

A canonical event, conceptually:

```jsonc
{
  // --- provenance: where this came from, always present ---
  "event_id": "uuid",                 // our id for this canonical record
  "raw_ref": "landing/aws/2026-06-09/00412.json",  // pointer back to the verbatim raw record
  "source_system": "aws.cloudtrail",  // which source produced it
  "source_subsidiary": "northstar",   // which org/subsidiary (the synthetic one is "northstar")
  "ingest_time": "2026-06-09T17:04:22Z",
  "schema_version": "1.0",

  // --- ASIM-aligned core ---
  "event_type": "Authentication",     // ASIM schema: Authentication | NetworkSession | ProcessEvent | AuditEvent ...
  "event_result": "Failure",          // Success | Failure | Partial
  "event_time": "2026-06-09T17:03:58Z",

  // --- actor: who did it (the correlation surface) ---
  "actor": {
    "src_identifier": "AIDAEXAMPLE123",       // identifier AS IT APPEARS in the source
    "src_identifier_type": "aws_iam_principal",
    "resolved_entity_id": "ent:person:0091",  // the graph's resolved entity, filled by correlation
    "display_name": "j.okafor",               // best-known human-readable, if resolvable
    "confidence": 0.92                        // correlation confidence, 0–1
  },

  // --- target: what was acted on ---
  "target": {
    "src_identifier": "s3://ng-claims-archive",
    "src_identifier_type": "aws_resource",
    "resolved_asset_id": "asset:store:0143"
  },

  // --- network, when relevant ---
  "src_ip": "10.0.0.44",
  "src_geo": "US-WA",

  // --- the original, untranslated, for recovery and audit ---
  "raw_excerpt": { /* the source-native fields, verbatim, for the fields we mapped */ }
}
```

### Why these fields and not more

The model is intentionally minimal. Three field groups carry the whole design:

- **Provenance** (`raw_ref`, `source_system`, `source_subsidiary`) makes every claim traceable. When someone says "that's the wrong data," provenance is how you settle it in seconds instead of a day.
- **Actor with `src_identifier` AND `resolved_entity_id`** is the correlation hinge. The source identifier is kept exactly as it appeared (never overwritten); the resolved entity is added alongside. You can always see both what the source said and who we think it is, plus a confidence. That separation is what makes the correlation auditable instead of magic.
- **`event_type` aligned to ASIM** lets the same question run across sources without per-source special-casing.

Can be extended later.

---

## The normalization contract

Each source gets a **normalizer**: a small, testable mapping from that source's raw shape to the canonical event. The contract for every normalizer:

1. **Total on its inputs or explicitly partial.** Either it maps a record fully, or it emits a canonical event flagged `event_result: "Partial"` with a validation note. It never silently drops a field it did not understand.
2. **Never invents.** If a field is absent in the source, it is absent (null) in the canonical event. No defaulting that fabricates signal. (Just like a RAG system we don't want hallucinating columns: output is bounded to what the input supports.)
3. **Preserves the source identifier verbatim.** `src_identifier` is the source's truth. Resolution is layered on top, never destructive.
4. **Is independently testable.** Given a raw fixture, it produces a known canonical event. Normalizers are unit-tested against committed sample records.

The three normalizers in the build:

| Source | Raw shape | Canonical mapping notes |
|---|---|---|
| `aws.cloudtrail` | AWS JSON, `userIdentity`, `eventName`, `errorCode` | IAM principal → actor; `errorCode` presence → `event_result` |
| `entra.signin` | Entra sign-in log JSON | UPN → actor; `status.errorCode` → result; rich geo |
| `northstar.syslog` | Deliberately ugly: pipe-delimited, abbreviated field names, local user IDs, different timestamp format | Made harder to prove normalization. Local user IDs must be correlated to real entities. |

---

## The identity graph

Correlation resolves `src_identifier` values across sources to a single `resolved_entity_id`. Built as a graph because the cross-source question must resolve in seconds, not after a batch recompute (see ADR-005).

### Entities and edges

```
  (person:0091  "j.okafor")
       │  is        │  is              │  is
       ▼            ▼                  ▼
 (aws_iam:        (entra_upn:        (northstar_uid:
  AIDAEXAMPLE123)  jokafor@corp...)   ns\jok)
```

- **Entity nodes**: the resolved real-world things (`person`, `service_account`, `device`, `asset`).
- **Identifier nodes**: every source-native identifier seen (`aws_iam`, `entra_upn`, `northstar_uid`, `device_id`, ...).
- **`is` edges**: bind an identifier to an entity, each with a **confidence** and a **basis** (how we know: exact email match, HR feed, manual assertion, heuristic).

### How edges get created (in priority order)

1. **Authoritative join** — a shared strong key (corporate email present in both Entra and a subsidiary HR export). Confidence ~1.0, basis `authoritative`.
2. **Deterministic match** — normalized handle equality (`jokafor` ≡ `j.okafor` after a documented normalization rule). High confidence, basis `deterministic`.
3. **Heuristic match** — co-occurrence (same source IP, same device, overlapping time) suggesting the same actor. Lower confidence, basis `heuristic`, **never auto-promoted to authoritative**.
4. **Manual assertion** — a human says "these are the same." Confidence 1.0, basis `manual`, audited.

The confidence and basis travel with the resolution into every canonical event. A query can demand "only authoritative or deterministic links" when the stakes are high, or accept heuristic links when casting a wide net. The analyst (or the agent, under guardrail) chooses the confidence floor. **That is the difference between an identity graph you can defend and one you just hope is right.**

### Why confidence is non-negotiable

The failure mode of cross-source correlation is false merges: deciding two identifiers are the same actor when they are not, then making a security call on it. Carrying confidence and basis means a wrong merge is visible and reversible, and a high-stakes query can refuse to rely on a weak link. This is the security analog of the fraud-identity-graph discipline: you stitch aggressively to see the whole actor, but you never let a weak stitch drive an irreversible action.

---

## Where this connects to the rest

- The **normalizers** live in [`../normalize/`](../normalize/).
- The **identity graph** lives in [`../correlate/`](../correlate/).
- The **validation** that enforces the normalization contract lives alongside both and is described in [`03-threat-question.md`](03-threat-question.md) and the guardrails design.
- The canonical events land in **Sentinel**, where the cross-source question is expressed as KQL over the canonical schema (see [`../soc/`](../soc/)).
