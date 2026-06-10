# infra/

Setup notes for the free-tier environment. No secrets, ever. Credentials live in a gitignored `.env`; this file only describes the steps.

## The three free pieces

### 1. Microsoft 365 E5 developer sandbox (the SOC side)

A free 90-day E5 tenant that auto-renews under active development. Reproduces the entire Microsoft SOC: Sentinel, Defender XDR, Entra ID, Purview, and Security Copilot.

- Sign up: Microsoft 365 Developer Program → instant sandbox. 25 user licenses, 1 admin.
- Includes Entra ID (identity signal + agent least-privilege control), Defender (endpoint signal), and as of the Apr–Jun 2026 rollout, Security Copilot bundled into E5 (400 SCU / 1,000 users, Agent Builder, MCP/Graph APIs).
- Stand up Microsoft Sentinel on a Log Analytics workspace inside this tenant. First 10 GB/day free for 31 days; Defender/Entra audit logs free in perpetuity. Demo volume is trivial.

### 2. AWS free tier (the non-Microsoft source side)

Reproduces the AWS-primary multicloud source.

- Enable CloudTrail (control-plane API calls) and VPC flow logs.
- One or two t-class instances or a couple of Lambda functions are enough to generate real CloudTrail signal.
- Optional: skip live AWS entirely and use the committed sample CloudTrail fixtures under `ingest/sample/`. The architecture is provable without spending a cent or opening an AWS account.

### 3. Synthetic subsidiary (the hard normalization case)

No infrastructure. A local emitter (`ingest/synthetic_northstar.py`) generates the deliberately ugly subsidiary log: pipe-delimited, abbreviated field names, local user IDs, off-format timestamps. Stands in for the un-migrated acquisition nobody has normalized. Configurable to inject the demo's cross-source attack pattern.

## Credential handling (the non-negotiable)

- All credentials (Azure app registration, AWS keys, Graph tokens) live in `.env`, which is gitignored.
- `.env.example` (committed) documents the variable names with empty values, so a reader knows what is needed without seeing any secret.
- Nothing under `secrets/`, no `*.pem`, no `*.key`, no creds JSON is ever committed. The `.gitignore` enforces this.
- This is a public portfolio repo. The credential discipline is itself part of what it demonstrates.

## Minimum path to a running demo

The leanest route, no cloud accounts required:

1. Use committed sample fixtures for all three sources (`ingest/sample/`).
2. Run ingest → normalize → correlate locally (Python).
3. Run the cross-source question against the local canonical store.
4. (Optional, additive) Wire Sentinel and the live clouds once the local slice works.

Build the local slice first. Cloud wiring is the second lap, not the first.

## Environment Configuration

This is the content of the .env file.

```
# Copy to .env (gitignored) and fill in. NEVER commit the real .env.
# The local slice (steps 0-5 of the runbook) needs NONE of these.
# These are only for wiring the real cloud environment.

# --- Microsoft E5 dev tenant (Graph + Sentinel) ---
AZURE_TENANT_ID=
AZURE_CLIENT_ID=
AZURE_CLIENT_SECRET=
LOG_ANALYTICS_WORKSPACE_ID=
LOG_ANALYTICS_DCR_ENDPOINT=

# --- AWS (CloudTrail / VPC flow) ---
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=us-east-1

# --- Agent / Security Copilot (MCP) ---
AGENT_IDENTITY_CLIENT_ID=
```