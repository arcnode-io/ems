# Cloud Persistence Provisioning Design

**Status:** Draft
**Date:** 2026-05-10
**Sub-project:** Cloud-deployment plug-and-play follow-up to ADR-001 §5 and ADR-003
**Depends on:** ADR-001 (System Architecture), ADR-003 (Boot DTM Source Strategy)
**Future ADR:** This spec is the design source for **ADR-004: Cloud Persistence Provisioning Strategy**

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
pastes is the opposite of plug-and-play. The pain point is the **vendor
signup matrix**, not the use of specialized DBs per se.

Two further constraints landed since ADR-001:

- **Neon dropped AWS Marketplace billing** on 2026-02-06 (post-Databricks
  acquisition). Neon-on-marketplace was the cleanest path to single-AWS-bill
  for Neon; that path is now closed.
- **Aurora serverless** (formerly Aurora Serverless v2; renamed April 2026)
  added scale-to-0-ACU with auto-pause in Nov 2024. A managed Postgres on
  AWS is now genuinely cheap at idle, removing the historical cost
  objection in ADR-001's "Managed AWS Services" rejection note.

This spec resolves both: replace Neon (both halves) with Aurora
serverless PG; provision Tiger Cloud and Neo4j Aura via inline CFN
custom-resource Lambdas using vendor API tokens pasted as CFN parameters.
Operator does one round of vendor signup (or two AWS Marketplace 1-clicks)
and one CFN launch. Everything else flows through Secrets Manager into
ECS task definitions.

The existing app code (`ems-device-api`, `ems-analyst-*`,
`ems-industrial-gateway`, `ems-line-controller`, `ems-hmi`) targets
Postgres protocol and Bolt/Cypher. **No app code changes** are required —
this is purely a deployment-layer redesign.

## Goals

1. A single per-order CFN yaml produces a fully provisioned cloud EMS
   stack: VPC, ECS cluster, Aurora serverless PG, Tiger Cloud instance,
   Neo4j Aura instance, S3 buckets, Secrets Manager entries, ECS services.
2. Operator inputs to the CFN are limited to:
   - `DeploymentUuid` and `DtmS3Url` (baked in by `platform-api` per ADR-001 §5)
   - `TigerCloudApiKey` and `TigerCloudProjectId` (operator-pasted)
   - `Neo4jAuraClientId` and `Neo4jAuraClientSecret` (operator-pasted)
3. Connection strings flow CFN custom resource → Secrets Manager → ECS
   task definition `secrets:` block. Containers receive them as env vars at
   start. No connection strings appear in the CFN template literal text.
4. ISO and dev-compose paths remain unchanged. Symmetry holds at the
   protocol layer (Postgres + Bolt + S3).
5. Zero direct vendor account exposure to the operator beyond the
   one-time API token generation step (or zero if marketplace SaaS-contract
   auto-provision is later wired up — see Future Work).

## Non-goals

- Replacing Tiger Cloud or Neo4j Aura with AWS-native counterparts
  (Aurora-with-partitioning, Neptune). Existing app code targets
  Timescale features and Cypher; rewriting is out of scope. Tiger Cloud
  and Neo4j Aura remain on the cloud path.
- Changing the ISO or dev-compose deployment recipe. Those keep the
  current single-postgres-with-extensions + neo4j daemon setup per
  `engineering-with-ai/tooling-playbooks/dev-services-setup.yml`.
- Multi-region failover, read replicas, or HA topology for the persistence
  layer. MVP cloud is single-AZ for the managed DBs (vendor defaults).
- Marketplace SaaS-contract auto-provisioning (vendor receives AWS account
  entitlement and auto-creates instance keyed to the AWS account, no API
  tokens). Possible future enhancement; requires per-vendor verification.
- Backup, snapshot, or disaster-recovery configuration of the managed DBs.
  Vendor defaults at MVP.

## Design

### Cloud persistence layout (revised)

| Slice | Service | Provisioning mechanism | Idle cost (us-east-1) |
|---|---|---|---|
| Document | Aurora serverless PG (one DB) | Native CFN `AWS::RDS::DBCluster` + `AWS::RDS::DBInstance` | ~$0 (auto-pause) |
| Vector | Aurora serverless PG (same cluster, `vector` ext) | Native CFN | shared with document |
| Time series | Tiger Cloud (one service) | Inline-Lambda CFN custom resource via Tiger REST API | ~$30+/mo (Tiger PAYG floor) |
| Graph | Neo4j Aura (one instance) | Inline-Lambda CFN custom resource via Aura REST API | ~$65+/mo (Aura PAYG floor) |
| Boot DTM + object | S3 | Native CFN `AWS::S3::Bucket` (already in ADR-003) | metered |

Aurora serverless PG hosts both the document and vector workloads in a
single cluster. The existing app split — `device-api` writes to the
document database, the domain MCP server writes to the vector database —
is preserved by creating two logical databases inside the one Aurora
cluster (`ems_document`, `ems_vector`), with `CREATE EXTENSION vector;`
applied to `ems_vector` only. Database creation and extension installation
run via a small inline-Lambda custom resource that connects with the
master credentials produced by `AWS::SecretsManager::Secret` and executes
the SQL once at stack-create time.

### Operator UX (one-time)

1. **Subscribe** to Tiger Cloud and Neo4j Aura. Two paths, operator picks:
   - **AWS Marketplace 1-click** — both vendors have PAYG marketplace
     listings. Charges land on the AWS bill (single bill).
   - **Direct vendor signup** — separate vendor billing relationship.
     Same downstream flow.
2. **Generate API tokens** in each vendor's console:
   - Tiger Cloud → "API Keys" → create key → record `TigerCloudApiKey`
     and the `TigerCloudProjectId` shown alongside it.
   - Neo4j Aura → "API Keys" (org-level) → create OAuth client → record
     `Neo4jAuraClientId` and `Neo4jAuraClientSecret`.
3. **Download** the per-order CFN yaml from the Arcnode delivery portal.
   Per ADR-001 §5, `platform-api` has already baked in
   `DeploymentUuid` + `DtmS3Url` as default parameter values.
4. **Launch** the stack:

   ```bash
   aws cloudformation create-stack \
     --stack-name ems-${ORDER_ID} \
     --template-body file://ems-stack.yaml \
     --capabilities CAPABILITY_NAMED_IAM \
     --parameters \
       ParameterKey=TigerCloudApiKey,ParameterValue=$TIGER_KEY \
       ParameterKey=TigerCloudProjectId,ParameterValue=$TIGER_PROJECT \
       ParameterKey=Neo4jAuraClientId,ParameterValue=$AURA_CLIENT_ID \
       ParameterKey=Neo4jAuraClientSecret,ParameterValue=$AURA_SECRET
   ```

5. **Done.** CFN provisions Aurora natively, calls Tiger and Aura APIs via
   the inline Lambdas, writes all conn strings to Secrets Manager,
   stands up ECS services. Total CFN runtime: ~10-15 minutes (Aurora is
   the slow leg; SaaS-API calls are seconds).

The operator never copies a connection string. The one place vendor
credentials are needed is the four CFN parameter pastes at step 4 above.

### CFN template structure

Sections, in order they appear in the per-order yaml:

1. **Parameters** — `DeploymentUuid`, `DtmS3Url`, `TigerCloudApiKey`,
   `TigerCloudProjectId`, `Neo4jAuraClientId`, `Neo4jAuraClientSecret`.
   Vendor params marked `NoEcho: true`.
2. **Networking** — `AWS::EC2::VPC`, two private subnets, security
   groups, NAT gateway. Standard ECS Fargate setup.
3. **Aurora serverless PG** — `AWS::RDS::DBCluster` (engine
   `aurora-postgresql`, engine version `16.3` or later, serverless
   scaling config with `MinCapacity: 0`, `SecondsUntilAutoPause: 300`),
   `AWS::RDS::DBInstance` (instance class `db.serverless`),
   `AWS::SecretsManager::Secret` for master credentials with managed
   rotation enabled.
4. **Aurora bootstrap Lambda** — inline custom resource that creates the
   `ems_document` and `ems_vector` databases and installs the `vector`
   extension. Runs once at stack-create. Unlike the vendor Lambdas in
   §5–§6, this Lambda needs `VpcConfig` (private subnets + security group
   allowing outbound to the Aurora cluster) because it speaks Postgres
   to a private RDS endpoint. The vendor Lambdas hit public REST APIs
   and run outside the VPC.
5. **Tiger Cloud Lambda + custom resource** — inline Python Lambda
   (described below) provisions a Tiger service via REST API, polls until
   ready, writes `TigerCloudConnectionString` to Secrets Manager.
6. **Neo4j Aura Lambda + custom resource** — same pattern, against the
   Aura API. Writes `Neo4jAuraUri`, `Neo4jAuraUsername`, `Neo4jAuraPassword`
   to Secrets Manager.
7. **S3 buckets** — DTM artifact bucket per ADR-003 (or operator-supplied
   bucket reference) + ML/MLflow buckets per readme.md cloud diagram.
8. **ECS cluster + services** — task definitions for `ems-device-api`,
   `ems-industrial-gateway`, `ems-line-controller`, `ems-hmi`,
   `ems-analyst-server`, `ems-analyst-agent`, `ems-analyst-model`,
   `ems-mlflow`, `emqx`, `prometheus`, `grafana`. Each task definition's
   `secrets:` block references the Secrets Manager entries by ARN.
9. **Outputs** — public load-balancer URL for `ems-hmi`, deployment
   summary for the operator.

### Inline Lambda design

Each vendor gets one inline Python Lambda embedded in the CFN template
via `AWS::Lambda::Function` with `Code.ZipFile`. Code budget: ~3 KB per
Lambda. The Lambda uses only the `boto3` and `urllib.request` modules
(both available in the AWS-managed Python 3.13 runtime), so no external
deps and no zip upload step.

**Common contract:**

- `RequestType: Create` →
  POST instance to vendor REST API →
  poll vendor's "is ready" endpoint with exponential backoff (cap 10 min) →
  on ready, write connection string + credentials to Secrets Manager →
  return `PhysicalResourceId = <vendor>-<instance-id>` and `Data` block with
  the Secrets Manager ARN.
- `RequestType: Delete` →
  DELETE instance via vendor REST API using stored PhysicalResourceId →
  delete the Secrets Manager entry →
  return success (also success on 404 — already gone).
- `RequestType: Update` →
  treat as immutable; return new PhysicalResourceId so CFN does
  Create-then-Delete. (Resizing managed-DB instances live is out of scope
  for MVP.)

**Failure semantics:**

| Vendor API response | Lambda action | CFN result |
|---|---|---|
| 4xx (auth, validation) | Send `FAILED` to CFN with message | Stack ROLLBACK; operator reads error in CFN events |
| 5xx (vendor outage) | Retry up to N times with backoff; if exhausted, send `FAILED` | Stack ROLLBACK; retry by re-launching |
| Timeout (>14 min Lambda limit) | Lambda returns `FAILED` | Operator increases retry count or contacts vendor |
| Network error | Same as 5xx | Same |

Lambda execution role: minimal `secretsmanager:CreateSecret` /
`PutSecretValue` / `DeleteSecret` for one wildcard prefix
(`arn:aws:secretsmanager:*:*:secret:ems/${DeploymentUuid}/*`), plus
`logs:*` for CloudWatch.

**Why inline ZipFile, not S3-hosted Lambda zip:**

ADR-001 §5 establishes that the per-order CFN yaml is a complete,
standalone artifact — "one yaml = no hidden state, no runtime dependency
between Arcnode and the operator's EMS." Hosting the Lambda zip in an
Arcnode-public S3 bucket would introduce exactly the runtime dependency
ADR-001 §5 rejects (operator's stack would fail to redeploy if Arcnode's
public bucket changed). Inline ZipFile keeps the artifact self-contained
at the cost of a tight code budget. Two ~3 KB Lambdas fit the CFN 1 MB
template limit comfortably, alongside the rest of the stack.

### Secrets Manager layout

All persistence credentials live under
`ems/${DeploymentUuid}/persistence/`:

| Secret name | Source | Consumers |
|---|---|---|
| `ems/${id}/persistence/aurora-master` | Aurora-managed rotation | Aurora bootstrap Lambda only |
| `ems/${id}/persistence/aurora-document` | Aurora bootstrap Lambda writes app-user creds | `device-api` |
| `ems/${id}/persistence/aurora-vector` | Same | `domain-mcp-server` |
| `ems/${id}/persistence/tiger` | Tiger Lambda writes `{host, port, user, password, database}` | `analyst-server`, `device-api` (timescale writes) |
| `ems/${id}/persistence/neo4j-aura` | Aura Lambda writes `{uri, username, password}` | `domain-mcp-server` |

ECS task definitions reference each secret by ARN in the `secrets:`
block; Fargate injects them as env vars at task start. The env var names
each service expects are defined per-service in its `cfg.yml` /
`template-secrets.env` (per ADR-001 conventions); the CFN task definition
maps Secrets Manager entries to whatever names the service expects. The
goal is to preserve the same env vars across cloud, ISO, and dev — exact
names are confirmed per service when wiring up its task definition.

### Symmetry with ISO and dev contexts

| Layer | Cloud | ISO / dev | Symmetry |
|---|---|---|---|
| Document store | Aurora serverless PG | postgres daemon | Postgres protocol both sides |
| Vector store | Aurora serverless PG (`vector` ext) | postgres daemon (`vector` ext) | Postgres protocol both sides |
| Graph store | Neo4j Aura | neo4j daemon | Bolt protocol both sides |
| Object store | S3 | minio | S3 protocol both sides (per ADR-003) |
| Time series | Tiger Cloud (separate cluster, Timescale features) | postgres daemon (`timescaledb` ext, same daemon as document) | Postgres protocol both sides; topology differs |

The only asymmetry is the time-series **topology** — cloud splits TS into
its own Tiger Cloud cluster (because Aurora cannot host the TimescaleDB
extension), while ISO and dev share the single postgres daemon for
document + vector + TS. App code is unaffected because connection strings
abstract this; the analyst service simply gets a different `DATABASE_URL`
for TS than it does for document.

### ISO and dev-compose paths (unchanged)

- **ISO**: continues to use the daemons set up by
  `engineering-with-ai/tooling-playbooks/dev-services-setup.yml` —
  one postgres with `timescaledb` + `vector` extensions, neo4j, minio.
  An ncurses TUI runs on first SSH login to populate `/etc/ems/cfg.yml`
  with broker password, deployment_uuid, and the boot DTM URL (per
  ADR-003). No vendor tokens involved because all daemons are local.
- **Dev**: docker-compose stack matches the ISO daemon layout (or the
  current Ansible host install for developers using a long-lived dev VM).
  No vendor signups needed for dev work.

### Updates to existing docs

- **`ems/readme.md`** — Cloud Deployment (AWS) PlantUML diagram:
  remove `cloud neon_vector` and `cloud neon_document`; add
  `database aurora_serverless` (or rename inside the existing
  `managed_persistence` rectangle). Tiger Cloud and Neo4j Aura
  unchanged.
- **`ems/system_adr.md` (ADR-001)** — Add a small banner at the top of
  the "Managed Postgres Service (like Neon) with TimeSeries Extension"
  rejection note pointing to ADR-004, of the form:
  *"Note: ADR-004 supersedes this rejection for the document + vector
  slice (Aurora serverless PG with `pgvector` ext). The rejection still
  holds for the time-series slice (Tiger Cloud kept)."*
  ADR-001's body otherwise stays untouched as historical record.
- **New ADR-004** — `cloud_persistence_provisioning_adr.md`, sourced from
  this design spec, covering the Decision / Rationale / Consequences /
  Alternatives Considered / Review sections in standard ADR form.

### Failure modes and rollback

| Failure | Behavior | Rationale |
|---|---|---|
| Operator pastes wrong vendor API token | Vendor API returns 4xx → Lambda sends FAILED → CFN ROLLBACK; previously created Aurora cluster is deleted by CFN | Surfaces config error at stack-create; operator fixes token and retries |
| Vendor API outage during Create | Lambda retries with backoff; on exhaustion FAILED → ROLLBACK | Operator retries when vendor recovers |
| Aurora bootstrap SQL fails | Custom resource sends FAILED → ROLLBACK | Indicates Aurora was provisioned but extension/database creation hit a real error; operator reads CloudWatch log |
| ECS task fails to read secret | Task definition `secrets:` block enforces this; task fails to start; ECS service event surfaces it | Standard ECS failure mode; not specific to this design |
| Stack delete leaves orphaned vendor instances | Lambda Delete handler deletes vendor instance + Secrets Manager entry; if Lambda fails, vendor instance is orphaned (still billing) | Documented; operator must manually delete via vendor console as fallback |

CFN's stack-rollback semantics handle the happy negative path
(failed-create → undo). The orphan risk on stack-delete is real but
limited; the Lambda's Delete handler being idempotent (404-on-already-gone
treated as success) reduces it.

### Cost floor estimate (us-east-1, idle)

Numbers below are rough estimates from public PAYG pricing pages at the
time of writing; actual cost varies by region, instance tier, and
workload. Treat as order-of-magnitude only.

| Component | Idle cost (estimate) |
|---|---|
| Aurora serverless PG (auto-paused, 0 ACU) | ~$0 (storage only, ~$0.10/GB/mo) |
| Tiger Cloud (PAYG smallest tier) | ~$30/mo |
| Neo4j Aura (PAYG smallest tier) | ~$65/mo |
| ECS Fargate (services scaled down) | ~$5-10/mo |
| S3 + data transfer | ~$1/mo |
| NAT gateway (always-on) | ~$33/mo |
| **Total idle** | **~$135-145/mo** |

Compares favorably to today's all-managed setup (Tiger ~$30 + Neon
document ~$20 + Neon vector ~$20 + Aura ~$65 + ECS/S3/NAT ~$45 ≈
$180/mo idle). Active workload pricing is dominated by Aurora ACU
consumption + ECS task hours; no significant change.

NAT gateway is the dominant always-on cost. A future optimization
(out of scope for MVP) is replacing it with VPC endpoints for the
specific AWS services the ECS tasks need (S3, Secrets Manager,
CloudWatch Logs) plus a NAT instance for vendor-API egress, which can
shave ~$25/mo off the idle floor.

### Future work

- **AWS Marketplace SaaS-contract auto-provisioning.** If Tiger Cloud and
  Neo4j Aura implement the marketplace-entitlement-driven provisioning
  flow (vendor receives AWS account ID, auto-creates instance keyed to
  it, exposes connection string via marketplace metering API), the
  operator's API-token paste step disappears entirely. Per-vendor
  verification needed; track in a follow-up spike.
- **Reverting Tiger Cloud to AWS-native.** If Aurora ever ships a
  TimescaleDB-compatible extension (e.g., a future Aurora extension or
  an AWS-native equivalent of compression + continuous aggregates), the
  Tiger Cloud Lambda can be removed and time-series consolidated into the
  single Aurora cluster. Re-evaluate at the next ADR review cycle.
- **Marketplace billing toggle.** Document both paths (marketplace 1-click
  vs direct vendor signup) in operator-facing onboarding docs once the
  marketplace listings are confirmed compatible with the API-token
  provisioning flow.

## Rationale

### Why keep Tiger Cloud and Neo4j Aura instead of going pure AWS-native

Existing app code targets TimescaleDB-specific features (compression,
continuous aggregates, hyperfunctions) and Cypher. Switching to
Aurora-with-pg_partman + Neptune-openCypher would require touching
`ems-analyst-server`, `ems-analyst-agent`, `ems-analyst-model`, and the
`domain-mcp-server` graph layer — well outside the scope of a deployment
redesign. The Tiger + Aura signup pain is solvable at the deployment
layer (this spec); the feature-set rewrite is not.

### Why replace Neon specifically

Two reasons converged:

1. **Neon dropped AWS Marketplace billing on 2026-02-06.** The cleanest
   "single AWS bill" path for Neon is closed. Direct Neon signup is the
   only remaining option, and that's the friction this spec is trying to
   eliminate.
2. **Aurora serverless became cost-competitive at idle in Nov 2024**
   with scale-to-0-ACU (the feature originally landed in "Aurora
   Serverless v2" before the April 2026 rename). The cost objection in
   ADR-001's "Managed AWS Services" rejection note no longer applies to
   the document + vector slice. Aurora supports `pgvector` natively (since 2023, currently
   at version 0.7.0+ with HNSW parallelism).

Together: replacing Neon with Aurora gives a cheaper, AWS-native,
zero-signup story for document + vector with no app code changes.

### Why inline Lambdas in CFN, not a separate provisioning service

ADR-001 §5 is explicit: the per-order yaml is a complete, standalone
artifact with no runtime dependency between Arcnode and the operator's
EMS. A separate provisioning service (Step Functions, EventBridge, or
external orchestrator) would violate that. Inline Lambdas keep the
artifact self-contained. The 1 MB CFN template limit and the
~4 KB per-Lambda inline ZipFile limit are the binding constraints; both
are comfortably above the 3 KB per-Lambda we need for thin REST-call
wrappers.

### Why operator pastes API tokens (not Arcnode pre-provisions)

This is OSS. Every operator deploys their own stack against their own
AWS account. Arcnode does not (and per ADR-001 §5 cannot) hold customer
credentials or pre-provision vendor instances on the operator's behalf.
The API-token-paste step is the minimum information the operator must
provide so the CFN Lambdas can act on the operator's behalf.

The marketplace-entitlement path would remove even this step but
requires per-vendor implementation (see Future work).

### Why Secrets Manager instead of CFN-template-literal env vars

Connection strings contain database passwords. Embedding them as
literal strings in the CFN template (or as plaintext ECS task definition
env vars) puts them in CloudTrail, CFN drift detection diffs, and any
log of the rendered template. Secrets Manager is the standard AWS pattern
for credential injection into ECS tasks; it's also what
`AWS::RDS::DBCluster`'s managed rotation expects.

## Alternatives Considered

### Pure AWS-native (Aurora + Neptune + Aurora-for-TS)

Considered in brainstorming. Rejected for MVP because it requires
non-trivial app code changes (TimescaleDB feature loss, Cypher to
openCypher port). Could be revisited if the time-series feature loss
becomes acceptable or if AWS ships a Timescale-compatible extension.

### Marketplace SaaS-contract auto-provisioning

Cleanest possible UX (zero manual signup, zero token paste) but requires
per-vendor implementation that may or may not exist. Tracked in Future
work; not blocking MVP.

### Hosting the provisioning Lambdas in Arcnode-owned S3

Allows larger Lambda code (no 4 KB inline limit). Rejected because it
introduces a runtime dependency between the operator's stack and Arcnode
infrastructure, violating ADR-001 §5.

### Using AWS Service Catalog instead of raw CFN

Service Catalog adds a UI layer and governance features over CFN. Not
needed for an OSS single-stack deployment; adds complexity without
benefit at this scope.

### Pre-provisioning vendor instances via platform-api before CFN download

Would let `platform-api` bake connection strings directly into the
per-order yaml. Considered briefly but rejected: customer's data would
live under Arcnode's vendor org account, complicating data ownership and
billing. OSS users without an Arcnode platform-api relationship would
have no path forward.

### Mounting connection strings as a JSON file via `/etc/ems/db.json`

Same family of "deployment-context multiplication" problem ADR-003
already rejected for the boot DTM. Skipped for the same reason.

## References

- [ADR-001: System Architecture](../../../system_adr.md) — particularly §5 (CFN/ISO deployment) and the "Managed Postgres Service" rejection note this spec partially flips
- [ADR-002: MQTT Topic Structure](../../../topic_structure_adr.md)
- [ADR-003: Boot DTM Source Strategy](../../../boot_strategy_adr.md) — establishes the "single mechanism, configurable endpoint" pattern this spec extends to vendor provisioning
- [Aurora Serverless v2 scale-to-0-ACU announcement (Nov 2024)](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-aurora-serverless-v2-scaling-zero-capacity/)
- [Aurora pgvector 0.7.0 + HNSW parallelism](https://aws.amazon.com/blogs/database/supercharging-vector-search-performance-and-relevance-with-pgvector-0-8-0-on-amazon-aurora-postgresql/)
- [Tiger Cloud AWS Marketplace listing](https://aws.amazon.com/marketplace/pp/prodview-iestawpo5ihca)
- [Neo4j Aura AWS Marketplace listing (PAYG)](https://aws.amazon.com/marketplace/pp/prodview-xd42uzj2v7dae)
- [Neon dropped AWS Marketplace billing (2026-02-06)](https://neon.com/docs/introduction/billing-aws-marketplace)
- [AWS CloudFormation custom resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)
- [`engineering-with-ai/tooling-playbooks/dev-services-setup.yml`](../../../../engineering-with-ai/tooling-playbooks/dev-services-setup.yml) — current dev/ISO daemon set
