# ADR-001: System Architecture for Energy Management Platform

**Status:** Accepted  
**Date:** 2025-08-05 (revised 2026-05-10 — cloud persistence stack folded in from former ADR-004)  
**Decision Makers:** Development Team  
**Consulted:** Energy Domain SMEs, Platform Engineering  
**Informed:** Course Participants, Operations Team

## Context

We need to design a system architecture for a modern energy management platform that:

1. **Handles diverse industrial protocols** (Modbus, DNP3, Redfish, SNMP, CANbus)
2. **Processes real-time telemetry** at 2-second intervals from grid equipment
3. **Supports both cloud and on-premise deployments** with different operational models
4. **Enables ML forecasting and AI-powered analysis** of energy data
5. **Provides real-time visualization** for grid operators
6. **Serves as educational platform** demonstrating modern cloud-native practices

The system must balance:
- Industry standards vs modern development practices
- Operational simplicity vs educational value
- Cost efficiency vs scalability requirements
- Protocol diversity vs architectural consistency

## Decision

We will implement a **microservices architecture with MQTT-based event streaming**, prioritizing industry standards and educational value over operational simplicity.

### Core Architecture

See [`ems/readme.md`](readme.md) for live topology, sequence, and deployment diagrams.

### Key Architectural Decisions

#### 1. MQTT as Central Event Bus
**Decision**: Use MQTT for all telemetry data flow rather than direct protocol translation or WebSockets.

**Rationale**:
- Industry standard for SCADA/IoT systems
- AsyncAPI documentation support
- Existing team expertise in energy sector
- Can be productized as standalone offering

**Implications**:
- Requires MQTT broker infrastructure (EMQX)
- Topic namespace management becomes critical

#### 2. No API Gateway Pattern
**Decision**: Frontend SPA calls backend services directly without API gateway abstraction.

**Rationale**:
- Reduces architectural complexity
- Avoids single point of failure
- Direct error handling per service
- Sufficient for known service endpoints

**Implications**:
- Per-service authentication handling
- Multiple CORS configurations
- Less centralized rate limiting
- Potential performance impact from multiple calls

#### 3. Dynamic Configuration Model
**Decision**: Protocol gateways fetch configuration via HTTP from device-api rather than static files.

**Rationale**:
- Enables dynamic topology updates
- Centralized configuration management
- Auto-generates MQTT topics from topology
- Supports multi-tenant deployments

**Implications**:
- Network dependency at gateway startup
- Requires device-api availability
- Exponential backoff for resilience
- More complex than file-based config

#### 4. Polyglot Microservices
**Decision**: Use language best suited for each domain rather than standardizing on one.

**Rationale**:
- Rust for safety-critical protocol handling
- Python for ML/data science ecosystem
- TypeScript for modern web development
- Leverages existing libraries per domain

**Implications**:
- Multiple build pipelines and toolchains
- Diverse operational patterns
- Higher team skill requirements
- Library-agnostic interfaces required

#### 5. Two Deployment Paths: CloudFormation or Air-Gapped ISO
**Decision**: Operator deploys via a per-order CFN yaml download (AWS standard or GovCloud) or a self-contained ISO image (air-gapped / defense). No Kubernetes. No IaC tooling on the operator's side.

**Rationale**:
- Operators should not need K8s expertise to run an EMS
- Per-order CFN yaml is downloaded from the portal and deployed by the operator via `aws cloudformation create-stack` CLI or AWS Console upload — Arcnode has no access to the resulting stack
- Each yaml is a complete, standalone artifact baking deployment_uuid + dtm_url + ems_mode directly in; eliminates release-coordination overhead
- ISO embeds the DTM, SSH key, and ems_mode directly — no network round-trips post-delivery
- `aws_partition` determines the path: `standard` / `govcloud` → CFN, `none` → ISO
- `deployment_context == defense_forward` locks to ISO

**Implications**:
- `platform-api` composes the per-order CFN template on demand (no release dance) and uploads yaml to S3 as `orders/{id}/ems-stack.yaml`
- Operator downloads from the delivery portal and runs locally — full control, no single-click convenience, but no hidden state
- No runtime infrastructure dependency between Arcnode and the operator's EMS
- ISO path unchanged: self-contained image with DTM + SSH key + ems_mode baked in

#### 6. SLD Topology in DTM, Rendered Programmatically by HMI

**Decision**: Electrical bus topology is encoded in a `buses[]` block in the DTM. `ems-hmi` renders the SLD from DTM data at runtime — it is not a hand-drawn per-deployment SVG.

**Rationale**:
- DTM is already the single source of truth for per-deployment data; bus topology is a relationship between DTM `device_id`s, not a separate concern
- Per-deployment SVG authoring doesn't scale and breaks on every site change
- `edp-api` already has the full electrical design (sizing payload) and is the natural author of `buses[]`
- IEC 61850-aligned: `buses[].id` = `ConnectivityNode`, `buses[].type` = `VoltageLevel` domain, `buses[].members[]` = `Terminal` references; `port` = Terminal name for bridging devices

**Implications**:
- `edp-api` computes `buses[]` from the sizing payload when it generates the DTM — it produces both in the same job
- `buses[]` entries are validated against `devices`: every `member.device_id` must resolve to a declared device
- Bus bars are not devices — they carry no `device_id` in `devices`, no MQTT topics, no template
- `ems-hmi` renders bus bars as horizontal bars with device nodes hanging off them; live MQTT measurements overlay each device node keyed by `device_id`
- Legacy approach (hand-drawn per-site SVG, e.g. Fractal) rejected: breaks on topology changes, doesn't scale

#### 7. Cloud Persistence: Aurora Serverless PG + Tiger Cloud + Neo4j Aura, CFN-Provisioned

**Decision**: Cloud-path persistence splits across three managed services, provisioned by the single per-order CFN template:

| Slice | Service | Provisioning |
|---|---|---|
| Document | Aurora serverless PG (`ems_document` db) | Native `AWS::RDS::DBCluster` |
| Vector | Aurora serverless PG (`ems_vector` db, `vector` ext) | Same cluster, separate db |
| Time series | Tiger Cloud | Inline CFN custom-resource Lambda → Tiger REST API |
| Graph | Neo4j Aura | Inline CFN custom-resource Lambda → Aura REST API |
| Object | S3 | Native `AWS::S3::Bucket` (per ADR-003) |

Operator pastes six vendor API tokens (Tiger access key + secret + project id; Aura OAuth client + secret + tenant id) as CFN parameters. All resulting connection strings flow through Secrets Manager into EC2 UserData and the docker-compose stack it starts. Operator never copies a connection string.

**Rationale**:
- OSS plug-and-play: one CFN file → fully provisioned stack; no manual vendor-console instance creation, no connection-string copy/paste
- Aurora serverless scale-to-0-ACU (Nov 2024) made AWS-native managed PG cost-competitive at idle; `pgvector` is native (0.7.0+ with HNSW parallelism) — replaces Neon for document + vector with no app code change
- Tiger Cloud kept for TimescaleDB-specific features (compression, continuous aggregates, hyperfunctions); Aurora cannot host the TimescaleDB extension
- Neo4j Aura kept for Cypher; Neptune-openCypher port is out of scope for a deployment redesign
- Inline Lambdas (Python `Code.ZipFile` ~3 KB each) keep the CFN artifact self-contained per §5 — no runtime dependency on Arcnode infra
- Neon dropped AWS Marketplace billing on 2026-02-06, closing the cleanest single-AWS-bill path for Neon

**Implications**:
- Operator subscribes to two vendors (Tiger Cloud, Neo4j Aura) and generates API tokens — six CFN params total
- Aurora bootstrap Lambda requires a psycopg2 layer (community ARN at MVP; self-publish later)
- Single-AZ subnet group at MVP; second AZ is a follow-up
- Stack-delete may orphan vendor instances if the Delete handler fails — handlers are idempotent (404 → success) to mitigate; operator may fall back to vendor console
- ISO + dev-compose unchanged: symmetry at the protocol layer (Postgres + Bolt + S3); cloud just splits time series into a separate Tiger Cloud cluster

## Implementation Details

See [`ems/readme.md`](readme.md) for cloud (AWS) and on-prem (ISO) deployment diagrams and data flow sequences.

## Consequences

### Positive
- **UTC-only timestamps**: All timestamps — sensor samples, API responses, UI display — are UTC epoch milliseconds. No local-time conversion at any layer; operators do their own offset math.
- **Industry Alignment**: Uses familiar patterns for energy sector teams
- **Educational Value**: Exposure to modern cloud-native stack
- **Flexibility**: Supports diverse protocols and deployment models
- **Scalability Path**: Can grow from pilot to production
- **Real-time Performance**: 2-second telemetry with 30 FPS visualization

### Negative
- **Operational Complexity**: Multiple languages, databases, and services
- **Over-Engineering**: Kubernetes adds unnecessary overhead for most loads
- **Cost Inefficiency**: Managed services and K8s increase expenses
- **Learning Curve**: Students must understand entire stack
- **Maintenance Burden**: Diverse technology stack requires broad expertise

### Risks
- **MQTT Broker Failure**: Central dependency requires HA configuration
- **Protocol Library Changes**: Rust ecosystem volatility
- **Database Sprawl**: Multiple specialized databases increase complexity
- **Skill Availability**: Finding polyglot developers is challenging 

## Alternatives Considered

### Monolithic Architecture
- **Rejected**: Doesn't demonstrate microservices patterns
- **Would simplify**: Deployment, debugging, initial development
- **Would lose**: Scalability, educational value, independent deployment

### WebSocket-Only Architecture  
- **Rejected**: Not industry standard for SCADA systems
- **Would simplify**: Browser integration, protocol count
- **Would lose**: Ecosystem compatibility, AsyncAPI support

### Kubernetes Deployment
- **Rejected**: Operators shouldn't need K8s expertise to run an EMS
- **Would add**: Scalability patterns, educational K8s exposure
- **Would lose**: Single-click deployment, air-gapped simplicity, zero-ops operator experience

### Pure AWS-native persistence (Aurora for all slices + Neptune)
- **Rejected for MVP**: TimescaleDB features (compression, continuous aggregates, hyperfunctions) and Cypher are in active app code; moving to Aurora-with-pg_partman + Neptune-openCypher would require app changes outside the scope of a deployment redesign. Aurora *is* used for the document + vector slices — see Key Decision #7.
- **Would simplify**: Single cloud provider billing, unified IAM and networking, consolidated monitoring
- **Would lose**: TimescaleDB features for time series; Cypher for graph
- **Revisit**: If AWS ships a Timescale-compatible extension, or if app code moves off TimescaleDB / Cypher

### Neon for document + vector
- **Rejected (2026-05-10)**: Neon dropped AWS Marketplace billing on 2026-02-06 (post-Databricks acquisition), closing the single-AWS-bill path. Aurora serverless became cost-competitive at idle with scale-to-0-ACU (Nov 2024) and supports `pgvector` natively — same vanilla-Postgres protocol surface, no app code change. See Key Decision #7.

### Pre-provisioning vendor instances via platform-api before CFN download
- **Rejected**: Customer data would live under Arcnode's vendor org account, complicating ownership and billing. OSS users without an Arcnode platform-api relationship would have no path forward.

### Neo4j as Graph Database
- **Chosen Neo4j**: Mature graph database with robust ecosystem, extensive documentation, and established community
- **Benefits**: 
  - Native support for labeled property graphs
  - Strong multi-tenancy support via node labels
  - Comprehensive AI and ML integration capabilities
  - Cypher query language for flexible graph querying
- **Implementation Strategy**:
  - Use node labels to create logical separations within the graph
  - Leverage Neo4j's built-in graph algorithms and ML libraries
  - Utilize Neo4j's cloud offerings (AuraDB) for managed deployments
- **Graph Modeling Example**:
  ```cypher
  CREATE (b:BessModule:Tesla {name: 'External BESS', model: 'Megapack 3'})
  CREATE (g:GridModule:Arcnode {name: 'Grid Container', site: 'brookside-dc-1'})
  CREATE (b)-[:CONNECTED_TO]->(g)
  ```
- **Implications**: Enables sophisticated graph-based analytics for energy management, supporting complex relationship tracking and AI-powered insights

## Review

This ADR should be reviewed:
- **Quarterly**: Assess if Kubernetes overhead is justified
- **Per cohort**: Gather feedback on architectural complexity
- **Major changes**: New protocols, deployment targets, or requirements
- **Cost reviews**: Evaluate if educational value justifies expenses

## References

- [MQTT in Industrial IoT](https://mqtt.org/use-cases/manufacturing/)
- [AsyncAPI Specification](https://www.asyncapi.com/)
- [Kubernetes Alternatives Analysis](https://k3s.io/)
- [Time-Series Database Comparison](https://www.timescale.com/)
