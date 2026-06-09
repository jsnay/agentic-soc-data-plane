# ingest/

Pull raw events from heterogeneous sources into a landing zone, keeping the raw record verbatim.

## Responsibility

Get events in without losing fidelity. Never normalize on the way in. The landing zone is the source of truth for "what actually arrived."

## Components (planned)

- `aws_cloudtrail.py` — read CloudTrail events (from a free-tier AWS account or committed sample fixtures), write each as a verbatim raw record to `data/landing/aws/...`.
- `aws_vpcflow.py` — VPC flow log lines, same treatment.
- `entra_signin.py` — pull Entra sign-in / audit logs from the E5 dev tenant via Graph API.
- `synthetic_northstar.py` — emit the deliberately ugly subsidiary log (pipe-delimited, abbreviated fields, local user IDs, off-format timestamps). The hard normalization case. Configurable to inject the demo's attack pattern.

## Contract

- Every record lands with a generated `raw_ref` path and an `ingest_time`.
- Raw bytes are preserved exactly. The normalize layer reads from here; it never reaches back to the live source.
- No credentials in code. Source config comes from a gitignored `.env`. See `../infra/README.md`.

## Sample fixtures

Small, committed, scrubbed sample records live under `sample/` so the pipeline and tests run with zero infrastructure. Real ingestion is additive, not required, to demonstrate the architecture.
