# ADR-001: System Architecture for Energy Management Platform

**Status:** Accepted  
**Date:** 2025-08-05  
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
**Decision**: Operator deploys via a CFN deep link (AWS standard or GovCloud) or a self-contained ISO image (air-gapped / defense). No Kubernetes. No IaC tooling on the operator's side.

**Rationale**:
- Operators should not need K8s expertise to run an EMS
- CFN is a single click in the operator's own AWS console — Arcnode has no access to the resulting stack
- ISO embeds the DTM, SSH key, and ems_mode directly — no network round-trips post-delivery
- `aws_partition` determines the path: `standard` / `govcloud` → CFN, `none` → ISO
- `deployment_context == defense_forward` locks to ISO

**Implications**:
- `platform-api` composes the CFN template and builds the ISO — both are per-deployment artifacts templated from the DTM
- No runtime infrastructure dependency between Arcnode and the operator's EMS
- Versioned releases (ISO + CFN template + APK) are published manually via `POST /admin/releases` and fanned out to all active deployments

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
- Bus bars are not devices — they carry no `device_id` in `devices`, no MQTT topics, no class
- `ems-hmi` renders bus bars as horizontal bars with device nodes hanging off them; live MQTT measurements overlay each device node keyed by `device_id`
- Legacy approach (hand-drawn per-site SVG, e.g. Fractal) rejected: breaks on topology changes, doesn't scale

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

###  Managed Postgres Service (like Neon) with TimeSeries Extension
- **Rejected**: The educational value of learning production TimescaleDB feature is significant if you're building time-series expertise.
- **Would simplify**: Initial setup for one service for document and time-series data, like neon. Lower entry cost (\$0 free tier vs $30/month
- **Would lose**: Compression (5-10x storage savings), continuous aggregates, data tiering, optimized ingestion performance (50K+ vs 5K-15K device capacity), purpose-built time-series tooling and monitoring
- **Implications**: Seed data and run devices when demoing cloud. Pause service otherwise. still have to pay for storage ~$0.18/GB (not compute)


### Managed AWS Services (like Aurora Serverless) vs Specialized Third-Party Services
- **Rejected**: AWS services optimized for general workloads, not specialized use cases. Higher operational complexity managing multiple AWS database services vs purpose-built solutions
- **Would simplify**: Single cloud provider billing, unified IAM and networking, native AWS service integration, consolidated monitoring and logging
- **Would lose**: Specialized features (database branching, time-series optimizations), competitive pricing for small workloads, purpose-built developer experiences, best-in-class tooling for specific use cases
- **Implications**: Already using Neo4j Aura (third-party service), so maintaining consistency with specialized services for each data model. Would require building custom solutions for features that specialized services provide out-of-the-box. **No code changes required between services - only connection string environment variables need updating (all use standard PostgreSQL and Neo4j protocols)**

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
  CREATE (n:BESSGraph:Siemens {name: 'Energy Storage Unit', model: 'SIESTORAGE'})
  CREATE (m:EnergyTrading:Siemens {name: 'Grid Connection Point', location: 'Munich Plant'})
  CREATE (n)-[:CONNECTED_TO]->(m)
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
