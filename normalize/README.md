# normalize/

The heart of the build. Map every raw event to one canonical, ASIM-aligned schema.

## Responsibility

Solve "nobody knows what it is." After this layer, every event has a known type, known fields, and known provenance, regardless of which of N sources it came from.

## Design

See [`../docs/02-data-model.md`](../docs/02-data-model.md) for the canonical event and the normalization contract.

## Components (planned)

- `canonical.py` — the canonical event definition + a `validate_canonical()` enforcing the contract (no invented fields, preserved `src_identifier`, ASIM-aligned `event_type`).
- `normalizers/aws_cloudtrail.py` — CloudTrail raw → canonical.
- `normalizers/entra_signin.py` — Entra sign-in → canonical.
- `normalizers/northstar.py` — the ugly subsidiary log → canonical. The one that proves normalization earns its keep.
- `tests/` — each normalizer unit-tested against committed raw fixtures producing known canonical output. This is where schema discipline is enforced, not hoped for.

## The contract (enforced in code)

1. Total on inputs, or explicitly `Partial` with a validation note. Never silently drops.
2. Never invents a value the source did not contain.
3. Preserves `src_identifier` verbatim; resolution is layered on, never destructive.
4. Independently testable: raw fixture in, known canonical event out.

## Why this is the heart

Every downstream claim, the fast query, the identity graph, the agent's reasoning, rests on the canonical events being trustworthy. If normalization is sloppy, everything above it is confidently wrong. This is the layer that, done right, makes the day-long question into a seconds-long one. Done wrong, it is the layer that produces "that's the wrong data."
