# correlate/

Resolve the same real-world entity across sources. The identity graph that makes cross-source questions answerable.

## Responsibility

Directly attack "individually talk to people across seven organizations." Stitch the AWS principal, the Entra UPN, and the subsidiary's local user ID into one resolved entity, with confidence and basis attached.

## Design

See the identity-graph section of [`../docs/02-data-model.md`](../docs/02-data-model.md).

## Components (planned)

- `graph.py` — the entity/identifier/edge model. Nodes for resolved entities, nodes for every source-native identifier, `is` edges carrying `confidence` and `basis`.
- `resolvers.py` — edge creation in priority order: authoritative join (shared strong key) → deterministic match (normalized handle equality) → heuristic (co-occurrence) → manual assertion. Each writes its basis and confidence.
- `resolve.py` — given a `src_identifier`, return the resolved entity (or none) with confidence. Built for query-time lookup, not batch recompute.
- `tests/` — known identifiers resolve to known entities at known confidence; weak links never auto-promote to authoritative.

## Why a graph, why confidence

A maintained graph makes resolution a lookup, not a nightly recompute, which is what the "seconds" mandate requires (ADR-005). Confidence and basis travel with every resolution so a high-stakes query can refuse to rely on a weak merge. The failure mode of correlation is the false merge; carrying confidence makes a wrong merge visible and reversible. Stitch aggressively to see the whole actor; never let a weak stitch drive an irreversible action.

This reuses real-time fraud identity-graph patterns: track who the actor, account holder, and device are, stitch as signal arrives, under latency pressure. The security entities differ; the technique is proven.
