# Build Runbook

The order of construction, sequenced by demonstrability. Each step ends in something you can show. If you run out of time at any step, you still have an honest thing to demo.

The principle: **build the smallest local loop first, prove it, then add breadth, then add the cloud.** Cloud wiring is the second lap.

---

## Step 0 — Repo + environment (30 min)

- [ ] `git init`, push the skeleton to a public GitHub repo.
- [ ] Python venv, `requirements.txt` (start minimal: just a JSON/schema lib and a test runner).
- [ ] Copy `.env.example` to `.env` (gitignored). Leave it empty for now; the local slice needs no creds.
- [ ] Confirm `.gitignore` is doing its job: `git status` shows no `.env`, no `data/`.

**Demoable:** a clean public repo with a README that states the problem. Already a portfolio artifact.

---

## Step 1 — One source → landing → canonical (half day)

- [ ] Commit a small real CloudTrail sample to `ingest/sample/aws/`.
- [ ] `ingest/aws_cloudtrail.py`: read the sample, write verbatim raw records to `data/landing/aws/`.
- [ ] `normalize/canonical.py`: the canonical event + `validate_canonical()`.
- [ ] `normalize/normalizers/aws_cloudtrail.py`: raw → canonical.
- [ ] One unit test: known raw fixture → known canonical event.

**Demoable:** a raw AWS event becomes a clean canonical event, and a test proves it. The smallest possible proof that normalization works.

---

## Step 2 — Validation on the loop (1–2 hrs)

- [ ] Pre-ingest check: reject/flag a malformed raw record.
- [ ] Post-normalization check: `validate_canonical()` catches an invented field, a dropped `src_identifier`, a non-ASIM `event_type`.
- [ ] A test that feeds deliberately bad data and proves it is caught at both ends.

**Demoable:** "you can build a robust pipeline and still emit garbage if you do not validate both ends," shown working. This is the data-quality discipline made concrete.

---

## Step 3 — The hard second source (half day)

- [ ] `ingest/synthetic_northstar.py`: emit the ugly subsidiary log (pipe-delimited, abbreviated fields, local user IDs, off-format timestamps).
- [ ] `normalize/normalizers/northstar.py`: the ugly log → the same canonical event.
- [ ] Tests proving the ugly shape normalizes correctly, including the off-format timestamp.

**Demoable:** two completely different source shapes, one canonical output. Normalization has earned its keep. This is the "un-migrated subsidiary" made real.

---

## Step 4 — Identity correlation (half to full day)

- [ ] `correlate/graph.py`: entity / identifier / edge model with confidence + basis.
- [ ] `correlate/resolvers.py`: authoritative → deterministic → heuristic → manual, in priority order.
- [ ] Seed an authoritative link (a small HR-style mapping) + a deterministic rule (handle normalization).
- [ ] `correlate/resolve.py`: `src_identifier` → resolved entity + confidence.
- [ ] Tests: three identifiers (AWS, Entra, northstar) resolve to one entity; a weak link stays heuristic.

**Demoable:** the cross-source question becomes answerable. Same actor, three sources, one resolved entity, with confidence. **This is the moment the demo has a spine, even with zero cloud wiring.**

---

## Step 5 — The question, answered (2–4 hrs)

- [ ] Add the Entra sign-in sample + normalizer (third source).
- [ ] Inject the attack pattern across the three sources (fail-then-success auth + sensitive AWS access by the same resolved entity).
- [ ] Run the cross-source query locally over the canonical store, with the `actor_confidence >= 0.8` floor.
- [ ] Produce the one result row, with provenance to all three source records.

**Demoable:** the full "day-vs-seconds" punchline, end to end, locally. If you stop here, you have already won the conversation.

---

## Step 6 (second lap) — Wire the real SOC (half to full day)

- [ ] Stand up the E5 dev tenant + Sentinel on a Log Analytics workspace.
- [ ] `soc/ingest_to_sentinel.py`: push canonical events into a custom Log Analytics table.
- [ ] `soc/queries/cross_source_actor.kql`: the question as real KQL in the real SOC.
- [ ] (Optional) wire live AWS CloudTrail and Entra instead of samples.

**Demoable:** the answer returning from the actual Microsoft SOC, in KQL, in seconds. The architecture proven on the real stack, not just locally.

---

## Step 7 (second lap) — The guardrailed agent (half day)

- [ ] `guardrails/tools.py`: the bounded tool set, exposed via MCP.
- [ ] `guardrails/audit.py`: log every agent action with full provenance.
- [ ] `guardrails/policy.py`: least-privilege Entra identity + the human-approval gate.
- [ ] Demo: the agent triages the correlated signal, proposes an action, stops at the human boundary, and the audit trail shows everything it did.

**Demoable:** machine-speed triage under hard guardrails. The agentic story, with the part that is genuinely yours (the guardrails) front and center.

---

## Time budget

- Steps 0–5 (local, no cloud): roughly one focused weekend. This is the load-bearing demo.
- Steps 6–7 (cloud + agent): a second weekend, additive. Strengthens the demo but is not required to win the conversation.

Build to step 5 first. Everything after is upside.
