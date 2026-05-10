# ADR-003: Boot DTM Source Strategy

**Status:** Accepted
**Date:** 2026-05-10
**Decision Makers:** Development Team
**Consulted:** Operations, Site Integrators
**Informed:** EMS Subsystem Maintainers

## Context

`ems-device-api` needs a DTM at boot to serve `/topology` and `/asyncapi` to its clients (gateway, line-controller, HMI). PR 1 + PR 2 of the redo-device-api foundation just landed the canonical Dtm shape that edp-api emits and device-api consumes. PR 3 added catalog-aware validation on `POST /topology`. None of those answer the question of where the day-1 DTM comes from — when an operator boots an ISO, deploys a CFN/ECS stack, or `docker compose up`s a dev environment, how does device-api find its first DTM?

Six candidate mechanisms were considered:

1. **Fetch from S3 at startup using a configurable endpoint URL** — works against real AWS S3 in cloud, against minio in ISO appliance (already a daemon there per `ems/readme.md` On-Prem Deployment), against LocalStack in dev (existing edp-api convention).
2. **Mount a JSON file at `/etc/ems/dtm.json`** — required four different deployment recipes (compose bind mount, k8s ConfigMap, ECS sidecar/EFS/S3-fetch-via-entrypoint, ISO image bake). Failed the simplicity test in practice.
3. **Bundle a default DTM into the device-api image at build time** — drift between edp-api emit and bundled default; production "accidentally works" against the bundled value.
4. **POST-only, no boot seed** — every deployment recipe gains a post-boot hook step; race conditions; doesn't compose with k8s readiness probes.
5. **Environment variable with the JSON inline** — DTMs are 10s of KB; awkward in env files.
6. **Parameter Store / Secrets Manager** — DTM is not a secret; over-tooling.

The pain point that drove this ADR: prior internal experience with multi-mechanism deployment-context plumbing (Docker volumes + k8s ConfigMaps + ECS sidecars + ISO bake-time writes) was painful. Picking ONE mechanism that works across every deployment context — with the only knob being an endpoint URL — eliminates the matrix.

## Decision

Day-1 boot uses **a single mechanism: an S3-compatible GET against a URL set via cfg.yml**. The S3 endpoint URL is also a cfg.yml setting so dev/CI/ISO/cloud all run the same code path against different storage backends. The fetch is fatal-on-error when configured; tests bypass via leaving the URL unset.

### Configuration

```yaml
local:
  boot_dtm_s3_url: ~                      # null in dev → skip fetch
  s3_endpoint_url: http://localhost:4566  # LocalStack
beta:
  boot_dtm_s3_url: s3://arcnode-artifacts/deployments/{deployment_id}/dtm.json
  s3_endpoint_url: ~                      # null = real AWS S3
```

ISO deployments override `s3_endpoint_url` to the on-prem minio. The container image is identical across every deployment context.

### Behavior matrix

| `boot_dtm_s3_url` | Topology table | Action |
|---|---|---|
| set | empty | fetch + parse + validate + seed |
| set | populated | fetch + skip seed (don't overwrite operator changes) |
| unset / null | empty | log "no boot_dtm_s3_url configured; starting empty" + skip |
| unset / null | populated | log + skip |

Note: when `boot_dtm_s3_url` is set and the table is populated, the fetch still happens (so we surface S3-side issues at boot rather than waiting for the next restart). The result is logged but not applied.

### Failure modes

| Failure | Behavior | Rationale |
|---|---|---|
| S3 returns 404 (object missing) | **Fatal** — process exits nonzero | URL is configured → operator promised the object exists; missing means deployment-pipeline error |
| S3 returns auth/4xx/5xx (network, perms) | **Fatal** | Same — URL configured means fetch is required |
| Object body invalid JSON | **Fatal** | Production config error must surface at boot |
| Body parses but fails Zod `Dtm` validation | **Fatal** | Schema drift between edp-api and device-api caught at boot |
| Body validates but `templates_used` slug unknown to bundled catalog | **Fatal** | Catalog drift caught at boot; identical check to POST /topology |
| DB write fails (e.g., DB unreachable) | **Fatal** with restart-loop | Pod restart until DB healthy; standard k8s/ECS pattern |
| `boot_dtm_s3_url` is unset | **Graceful empty start** with info log | Tests / CI / fresh-from-zero deployments |

### Storage backend per deployment context

Same code path. Different `s3_endpoint_url`:

| Context | `boot_dtm_s3_url` | `s3_endpoint_url` |
|---|---|---|
| **Cloud (CFN + ECS Fargate)** | `s3://arcnode-artifacts/deployments/<id>/dtm.json` | unset → real AWS S3 |
| **On-prem ISO appliance** | `s3://arcnode-artifacts/deployments/<id>/dtm.json` | `http://minio:9000` (per `ems/readme.md` On-Prem Deployment minio daemon) |
| **Dev (docker-compose)** | `s3://arcnode-artifacts/deployments/sample/dtm.json` | `http://localhost:4566` (LocalStack) |
| **CI / integration tests** | unset → skip fetch | n/a |
| **Smoke test (boot empty + POST later)** | unset → skip fetch | n/a |

### Idempotency

The behavior matrix above is idempotent across pod restarts: device-api never overwrites a populated topology table from a stale fetch. Operators who need to re-seed must either:
- Clear the topology table (e.g., a database migration step before redeploy)
- Use sub-project C's dynamic CRUD endpoints when ADR-002 §14 lands

### Authentication

For real AWS S3 (cloud deployment): standard IAM via ECS task role. Container has S3:GetObject on the artifact bucket. No credentials in the image.

For LocalStack / minio: anonymous or static credentials passed via environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`). Existing pattern in `edp-api/src/bom_generator/manifest_client.py`.

### Scope boundaries

- **Out of scope:** dynamic CRUD endpoints (covered by ADR-002 §14 + sub-project C)
- **Out of scope:** AsyncAPI version bump or `system/topology_changed` broadcast on boot seed (sub-project C)
- **Out of scope:** A bundled default DTM ("starter" inside the image) — explicitly rejected
- **Out of scope:** Mounted-file fallback — explicitly rejected (was the goal of this revision)

## Rationale

### Why S3-compatible fetch, not file mount?

A file-mount strategy required four different deployment recipes (compose bind mount, k8s ConfigMap, ECS sidecar/EFS/entrypoint-fetch, ISO image bake). Each recipe had its own failure modes. Operators doing cross-environment work had to internalize four mental models.

S3 fetch needs one code path. Endpoint URL is the only differentiation. ISO already has minio per the existing on-prem deployment diagram. Cloud has S3. Dev has LocalStack. Same SDK call, same parsing, same validation.

### Why ONE config knob (the URL)?

`boot_dtm_s3_url` set → fetch is mandatory. Unset → skip. No partial states, no fallback layers, no precedence rules. Tests configure unset; production configures set. The behavior matrix collapses.

### Why fatal-on-404?

A configured `boot_dtm_s3_url` is an operator promise that the object exists at the time the API boots. Missing object means the deployment pipeline (CFN, ISO bake, k8s + S3 sync) skipped a step. Catching that at process start beats discovering it via gateway smoke tests an hour later.

### Why graceful empty when URL unset?

Tests, CI, and "boot the API to verify it starts" smoke checks should not require a DTM. `POST /topology` is the canonical mutation surface (and the dynamic CRUD endpoints from ADR-002 §14 will also populate it). An empty topology is a valid state.

### Why S3 even on ISO?

`ems/readme.md` On-Prem Deployment shows `minio` as a daemon. The on-prem stack already runs S3-compatible storage. Reusing that storage for the boot DTM means zero new infra on the on-prem side.

### Why fetch on every boot, even when populated?

Fetching unconditionally (then deciding whether to apply) surfaces S3-side issues at boot. Operators detect a stale URL or expired creds during the next restart cycle, not when the next operator change happens. The cost is one S3 GET per pod start — negligible.

## Consequences

### Positive
- One mechanism, one code path, one test pattern
- ISO and cloud share the same image
- Drift surfaces at boot, not at runtime
- Idempotent restarts protect operator changes
- Reuses existing arcnode-artifacts bucket convention from edp-api

### Negative
- ems-device-api gains a runtime S3 SDK dependency (currently no S3 client in TS)
- Dev requires running LocalStack or a sample HTTP server (already common for edp-api work, modest dev-env friction)
- ISO deployments must populate minio with the DTM at imaging time

### Risks
- S3 outage during ECS task startup → restart loop. Mitigated by ECS task auto-retry; if S3 is down the cluster has bigger issues.
- A stale `boot_dtm_s3_url` pointing at a missing object → fatal exit. Error message names the URL; operator updates the deployment.
- ConfigMap/SSM/CFN parameter mismatch between cfg.yml `boot_dtm_s3_url` and what the deployment pipeline actually uploaded. Standard config-error category; surfaces at boot.

## Alternatives Considered

### Mount file at `/etc/ems/dtm.json` (initial proposal)
- **Rejected:** required four different deployment-context recipes; multiplies operational complexity. Replaced by single S3 fetch.

### Bundle a default DTM in the image
- **Rejected:** drift risk, production "accidentally works" failure mode

### POST-only, no boot seed
- **Rejected:** race condition with operator POSTs at boot; doesn't compose with k8s readiness probes

### Environment variable with inline JSON
- **Rejected:** DTMs too large for env var ergonomics

### Parameter Store / Secrets Manager
- **Rejected:** DTM is not a secret, over-tooling

## Review

This ADR should be reviewed:
- **When dynamic CRUD endpoints land** (ADR-002 §14 sub-project C) — confirm boot fetch and CRUD don't conflict
- **When multi-tenant or multi-site scenarios appear** — the assumption "one DTM per device-api instance" may need revision
- **When ISO deployments need an alternative to running minio** — currently relies on the on-prem minio daemon; if minio gets dropped from the on-prem stack, this ADR needs revision

## References

- [ADR-001: System Architecture](system_adr.md)
- [ADR-002: MQTT Topic Structure and Payload Conventions](topic_structure_adr.md)
- [Foundation design spec (sub-project A)](docs/superpowers/specs/2026-05-09-redo-device-api-foundation-design.md)
- [Sub-project B day-1 boot design](docs/superpowers/specs/2026-05-10-day-1-boot-design.md)
- [edp-api manifest_client S3 pattern](../edp-api/src/bom_generator/manifest_client.py)
