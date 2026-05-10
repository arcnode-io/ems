# ADR-004: Cloud Persistence Provisioning Strategy

**Status:** Accepted
**Date:** 2026-05-10
**Decision Makers:** Development Team
**Consulted:** Operations, Site Integrators, OSS contributors
**Informed:** EMS Subsystem Maintainers, Course Participants

## Context

ADR-001 §"Cloud Deployment (AWS)" specifies four managed persistence
services on the cloud path: Tiger Cloud (time series), Neon (document),
Neon (vector), Neo4j Aura (graph). The ADR's implicit assumption was a
commercial Arcnode operator that handled vendor signups out-of-band.

EMS is an open-source project. A fresh operator who downloads the
per-order CFN yaml currently has to:

1. Sign up at Tiger Cloud, generate API token, manually create instance
2. Sign up at Neon, generate API token, manually create two databases
   (document, vector with pgvector ext)
3. Sign up at Neo4j Aura, manually create instance
4. Paste four connection strings into the CFN template before launch

Four signups + four manual instance creations + four connection-string
pastes is the opposite of plug-and-play. The pain is the **vendor
signup matrix**, not the use of specialized DBs per se.

Two further constraints landed since ADR-001:

- **Neon dropped AWS Marketplace billing** on 2026-02-06 (post-Databricks
  acquisition). The cleanest single-AWS-bill path for Neon is closed.
- **Aurora serverless** (formerly Aurora Serverless v2; renamed
  April 2026) added scale-to-0-ACU with auto-pause in Nov 2024. AWS-native
  managed Postgres is now genuinely cheap at idle, removing the cost
  objection in ADR-001's "Managed AWS Services" rejection note.

## Decision

Replace Neon (both halves) with **Aurora serverless PG** (with the
`pgvector` extension) on the cloud path. Provision Tiger Cloud and
Neo4j Aura via **inline CFN custom-resource Lambdas** that call the
vendor REST APIs using API tokens pasted as CFN parameters. All
resulting connection strings flow through Secrets Manager into EC2
UserData and the docker-compose stack it starts.

### Cloud persistence layout (revised)

| Slice | Service | Provisioning |
|---|---|---|
| Document | Aurora serverless PG (`ems_document` db) | Native CFN `AWS::RDS::DBCluster` |
| Vector | Aurora serverless PG (`ems_vector` db, `vector` ext) | Same cluster, separate db |
| Time series | Tiger Cloud | Inline-Lambda CFN custom resource via Tiger REST API |
| Graph | Neo4j Aura | Inline-Lambda CFN custom resource via Aura REST API |
| Object | S3 | Native CFN `AWS::S3::Bucket` (per ADR-003) |

### Operator UX (one-time, OSS plug-and-play)

1. Subscribe to Tiger Cloud + Neo4j Aura (AWS Marketplace 1-click for
   single-AWS-bill or direct vendor signup — both work).
2. Generate API tokens in each vendor's console:
   - Tiger Cloud: access key + secret key + project id (3 values).
   - Neo4j Aura: OAuth client id + secret + tenant id (3 values).
3. Download per-order CFN yaml from the Arcnode portal.
4. `aws cloudformation create-stack` passing the six vendor token params
   above. `DeploymentUuid` and `DtmS3Url` are baked into the template
   literal by `platform-api` per ADR-001 §5.
5. CFN provisions Aurora natively, calls Tiger and Aura APIs via the
   inline Lambdas, writes all conn strings to Secrets Manager, then
   launches the EC2 instance whose UserData fetches the secrets and
   starts the docker-compose stack.

The operator never copies a connection string. Vendor credentials are
needed only at the six CFN parameter pastes in step 4.

## Rationale

### Why keep Tiger Cloud and Neo4j Aura instead of going pure AWS-native

Existing app code targets TimescaleDB-specific features (compression,
continuous aggregates, hyperfunctions) and Cypher. Switching to
Aurora-with-pg_partman + Neptune-openCypher would require app code
changes outside the scope of a deployment redesign. The Tiger + Aura
signup pain is solvable at the deployment layer (Lambda custom
resources); the feature-set rewrite is not.

### Why replace Neon specifically

Two reasons converged:

1. **Neon dropped AWS Marketplace billing on 2026-02-06.** Direct Neon
   signup is the only remaining option, and that's the friction this
   ADR eliminates.
2. **Aurora serverless became cost-competitive at idle in Nov 2024**
   with scale-to-0-ACU. Aurora supports `pgvector` natively (since 2023,
   currently 0.7.0+ with HNSW parallelism). No app code change needed —
   both Neon and Aurora speak vanilla Postgres.

### Why inline Lambdas in CFN, not a separate provisioning service

ADR-001 §5 is explicit: the per-order yaml is a complete, standalone
artifact with no runtime dependency between Arcnode and the operator's
EMS. Inline Lambdas (Python `Code.ZipFile` ~3 KB each) keep the
artifact self-contained.

### Why operator pastes API tokens

This is OSS — every operator deploys their own stack against their own
AWS account. Arcnode does not (and per ADR-001 §5 cannot) hold customer
credentials. The API-token paste is the minimum information the
operator must provide so the CFN Lambdas can act on the operator's
behalf. Marketplace SaaS-contract auto-provisioning would remove even
this step but requires per-vendor implementation (tracked as future
work).

## Consequences

### Positive

- **OSS plug-and-play.** Single CFN file produces a fully provisioned
  stack. No manual instance creation in vendor consoles. No connection
  string copy/paste.
- **No app code changes.** Aurora speaks vanilla Postgres; Tiger Cloud
  speaks Postgres + TimescaleDB features; Neo4j Aura speaks Bolt/Cypher.
- **ISO + dev-compose unchanged.** Symmetry holds at the protocol layer
  (Postgres + Bolt + S3); cloud just splits time series into a separate
  Tiger Cloud cluster because Aurora cannot host the TimescaleDB
  extension.
- **Drift surfaces at stack-create.** Wrong API token, vendor outage,
  or schema drift all FAIL the CFN custom resource cleanly with
  rollback.

### Negative

- **Operator must still subscribe to two vendors** (Tiger Cloud, Neo4j
  Aura) and generate API tokens. Marketplace 1-click reduces this to
  two AWS-billed subscriptions, but the token-generation step remains
  manual until vendor SaaS-contract auto-provisioning is wired up.
- **psycopg2 Lambda Layer required** for the Aurora bootstrap Lambda
  (psycopg2 isn't in the python3.13 runtime). MVP uses a public
  community-published layer ARN; self-publishing an arcnode layer is a
  follow-up.
- **Single-AZ subnet group at MVP.** RDS production clusters require
  two subnets in different AZs; the existing minimal VPC has one
  subnet that's reused twice. Adding `EmsSubnetB` to the VPC is a
  follow-up.

### Risks

- **Stack-delete orphans.** If the vendor-Lambda Delete handler fails
  (network error, vendor outage), the vendor instance is orphaned and
  continues to bill. Lambda's Delete handler is idempotent (404 →
  success) to reduce this; operator may need to delete via vendor
  console as fallback.
- **CFN inline ZipFile 4 KB limit.** Each Lambda must fit in ~3 KB to
  leave headroom. Tiger and Aura provisioners are at the budget edge.
  If the API contract evolves to require more code, fall back to an
  S3-hosted Lambda zip (violates ADR-001 §5 spirit; defer).

## Alternatives Considered

### Pure AWS-native (Aurora + Neptune + Aurora-for-TS)

Rejected for MVP because it requires non-trivial app code changes
(TimescaleDB feature loss, Cypher to openCypher port). Could be
revisited if AWS ships a Timescale-compatible extension.

### Marketplace SaaS-contract auto-provisioning

Cleanest possible UX (zero token paste) but requires per-vendor
implementation that may or may not exist. Tracked as future work.

### Hosting provisioning Lambdas in Arcnode-owned S3

Rejected — introduces a runtime dependency between the operator's
stack and Arcnode infrastructure, violating ADR-001 §5.

### Pre-provisioning vendor instances via platform-api before CFN download

Rejected — customer's data would live under Arcnode's vendor org
account, complicating data ownership and billing. OSS users without
an Arcnode platform-api relationship would have no path forward.

## Implementation

See:
- Design spec: [`docs/superpowers/specs/2026-05-10-cloud-persistence-provisioning-design.md`](docs/superpowers/specs/2026-05-10-cloud-persistence-provisioning-design.md)
- Implementation plan: [`platform-api/docs/superpowers/plans/2026-05-10-cloud-persistence-provisioning.md`](../platform-api/docs/superpowers/plans/2026-05-10-cloud-persistence-provisioning.md)
- Code: `platform-api/src/cfn/persistence/`

## Review

This ADR should be reviewed:
- When marketplace SaaS-contract auto-provisioning becomes viable per vendor
- When Aurora ships a TimescaleDB-compatible extension
- When the ISO deployment context changes
- When a new persistence slice is added (e.g., search, queue)

## References

- [ADR-001: System Architecture](system_adr.md)
- [ADR-002: MQTT Topic Structure](topic_structure_adr.md)
- [ADR-003: Boot DTM Source Strategy](boot_strategy_adr.md)
- [Aurora Serverless v2 scale-to-0-ACU (Nov 2024)](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-aurora-serverless-v2-scaling-zero-capacity/)
- [Tiger Cloud REST API reference](https://docs.tigerdata.com/api/latest/api-reference/)
- [Neo4j Aura API authentication](https://neo4j.com/docs/aura/classic/platform/api/authentication/)
- [Neon dropped AWS Marketplace billing (2026-02-06)](https://neon.com/docs/introduction/billing-aws-marketplace)
- [AWS CloudFormation custom resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)
