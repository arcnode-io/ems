# EMS 🏭⚡


# Overview

This series of repos is designed to teach real world skills when implementing modern energy systems. It covers
everything from the sensors, to the UI (mobile and web), to predictive models, to AI agents. The four languages used
are


- 🌊 Typescript: Realtime UIs and APIs
- 🐍 Python: LLM/ML Apps and Platform Engineering
- 🦀 Rust: Grid Protocols and Embedded Systems
- ☕ Java: Grid-standards intake services (IEEE 2030.5 / OpenADR)


## Project Description

The EMS (Energy Management System) suite is the software that runs on a deployed Arcnode stack. It allows you to model different smart grid systems. For example, you could model a dynamic dlr system with a datacenter load. The bess could be modeled with modbus measurements and the datacenter could be modeled as snmp and redfish readings.

## Decisions

- [System ADR](system_adr.md) — architecture, MQTT contract, boot

# Diagrams

## Deployment*

```plantuml
cloud third_party_apis
rectangle ems #line.dashed {
collections "mock_industrial_protocols**" as mock_industrial_protocols
    rectangle industrial_gateway
    rectangle device_api
    rectangle der_control_api
    database timeseries
    database vector
    database graph
    database relational
    collections analyst_api
    database document
    collections ems_hmi
    person llm
    rectangle domain_mcp_server
}
rectangle  mock_derms #line.dashed {
  rectangle dlr_tap_regulator_sim
  rectangle dlr_rtu
  rectangle dispatch_api
}
dlr_rtu -d- dispatch_api: mqtt
dlr_tap_regulator_sim - dlr_rtu: mqtt
dispatch_api -l- der_control_api: http
industrial_gateway -u-> mock_industrial_protocols
industrial_gateway --> device_api: http
ems_hmi -u-> device_api: http
device_api -r-> document: sql
der_control_api -u-> relational: sql
analyst_api -l-> timeseries: sql
llm -d-> domain_mcp_server: mcp
domain_mcp_server -d-> vector: sql
domain_mcp_server -d-> graph: cypher
ems_hmi -u-> analyst_api: http
analyst_api -d-> llm: http
llm -l-> third_party_apis: http
```
> &ast; MQTT broker ommited for simplicity <br>
> &ast;&ast; dnp3, modbus, redfish, snmp, bacnet
 
## Sequence
### Default Use
```plantuml
participant device_api
database document
participant broker
participant industrial_gateway
participant dispatch_api
participant der_control_api
database relational
database timeseries
database vector
database graph
participant domain_mcp_server
participant llm
participant analyst_api
participant ems_hmi
participant ercot_api
collections third_party_apis
== bootstrap ==
device_api -> document: read /app/dtm.json, persist DTM + generate AsyncAPI v3 spec (per system_adr §23)
device_api -> broker: publish system/topology_changed { ts, version }
== distribute topics ==
industrial_gateway -> device_api: GET /asyncapi
ems_hmi -> device_api: GET /asyncapi\n(channels + schemas + x-protocol-source + x-enum-values)
ems_hmi -> device_api: GET /topology/view\n(sanitized DTM: devices + buses + per-measurement metadata)
ems_hmi -> device_api: GET /topology/sld.svg\n(generated SVG, regenerated on every topology change)
==  initialize messaging ==
industrial_gateway -> broker: pub grid protocols
broker -> timeseries: writes to db
broker -> ems_hmi: renders live data
== der dispatch (IP-native DNP3 twin) ==
dispatch_api -> der_control_api: POST /der-events (DERControl)
der_control_api -> relational: persist (upsert by mRID)
der_control_api -> broker: pub der_dispatch measurements\n(target_active_power, event_active, energize_enabled)
broker -> ems_hmi: Grid Events / DER Control panel
== ml workflows ==
ercot_api -> analyst_api: GET /solar-production
timeseries <- analyst_api: trains model
analyst_api -> ems_hmi: renders prediction
== ai agent workflows ==
ems_hmi -> analyst_api: GET /chat/completions  
analyst_api -> llm: query
llm -> domain_mcp_server: tool call
domain_mcp_server -> vector: agentic rag
domain_mcp_server -> graph: graph rag
llm -> third_party_apis: external api tool call
llm -> analyst_api: api prediction tool call
analyst_api -> llm: prediction response
llm -> analyst_api: synthesizes rag dbs and apis call
analyst_api -> ems_hmi: renders chat
```

## DER Event
```plantuml
participant gridstatus_api
participant dlr_rtu
participant dispatch_api
participant der_control_api
participant broker
participant dlr_tap_regulator_sim

== day-ahead forecast ==
gridstatus_api -> dispatch_api: load + weather forecast
dispatch_api -> dispatch_api: compute headroom curve (rating - forecast load)

== real-time monitoring ==
dlr_rtu -> dispatch_api: live rating (mqtt)
dlr_rtu -> dlr_tap_regulator_sim: live rating (mqtt)
note right of dlr_tap_regulator_sim: independent voltage-regulation loop\nno path to der_control_api
gridstatus_api -> dispatch_api: live loading
dispatch_api -> dispatch_api: trigger check (loading vs rating margin)

== constraint dispatch ==
dispatch_api -> dispatch_api: identify enrolled DER(s) + compute magnitude
dispatch_api -> der_control_api: POST /der-events (DERControl)

== compliance return path ==
der_control_api -> broker: pub der_dispatch measurements\n(target_active_power, event_active, energize_enabled)
broker -> dispatch_api: forward (same topic ems_hmi consumes)
dispatch_api -> dispatch_api: compare target vs measured active power\n(compliance + response time)

== continuous reassessment ==
dlr_rtu -> dispatch_api: live rating (mqtt)
gridstatus_api -> dispatch_api: live loading
dispatch_api -> dispatch_api: event-end check:\nrating recovered (sustained) OR duration >= max_duration_h

== event close ==
dispatch_api -> der_control_api: POST /der-events (DERControl: event_active=false)
```

## Cloud Deployment — Commercial

Split-topology: the ec2 stack runs in our AWS. The `industrial_gateway` runs on-prem at the customer site (next to their devices) and dials the cloud broker outbound. Gateway is shipped as a `docker-save` tarball via the platform-api delivery portal; the customer runs `docker load` + `docker run` on their site host.

```plantuml
rectangle ec2_docker_compose #line.dashed {
    rectangle analyst_agent
    rectangle analyst_model
    rectangle device_api
    rectangle der_control_api
    queue hivemq
    rectangle ems_hmi
    rectangle mlflow
    rectangle prometheus
    rectangle grafana
    rectangle analyst_server
}

rectangle customer_site #line.dashed {
    rectangle industrial_gateway
}

rectangle managed_persistence #line.dashed {
    database aurora_serverless
    database s3
}

rectangle external_managed_vendors #line.dashed {
    database tiger_cloud
    database neo4j_aura
}

rectangle managed_inference #line.dashed {
    cloud bedrock
}

rectangle third_party_apis #line.dashed {
    cloud ercot_api
    cloud openweather
    cloud yes_energy
    cloud permutable
}

industrial_gateway --> hivemq: mqtts (outbound from customer site)
```

## Cloud Deployment — Defense / Sovereign

Same split-topology as commercial: gateway runs on-prem at the customer site and dials the cloud broker outbound; shipped as `docker-save` tarball via the delivery portal.

```plantuml
rectangle ec2_docker_compose #line.dashed {
    rectangle analyst_agent
    rectangle analyst_model
    rectangle device_api
    rectangle der_control_api
    queue hivemq
    rectangle ems_hmi
    rectangle mlflow
    rectangle prometheus
    rectangle grafana
    rectangle analyst_server
}

rectangle customer_site #line.dashed {
    rectangle industrial_gateway
}

rectangle managed_persistence #line.dashed {
    database aurora_serverless
    database neptune
    database aoss
    database s3
}

rectangle managed_inference #line.dashed {
    cloud bedrock
}

rectangle third_party_apis #line.dashed {
    cloud ercot_api
    cloud openweather
    cloud yes_energy
    cloud permutable
}

industrial_gateway --> hivemq: mqtts (outbound from customer site)
```

## On-Prem Deployment (ISO)

Appliance/ISO orders bake the industrial-gateway into the live-build image alongside the rest of the stack — no separate tarball. Whole stack runs on the customer's on-site box.


```plantuml
rectangle daemons #line.dashed {
    database postgres_timeseries 
    database postgres_document 
    database postgres_vector
    database neo4j
    database minio
    rectangle ollama
    }

    rectangle docker_runtime #line.dashed {
    rectangle device_api
    rectangle der_control_api
    rectangle industrial_gateway
    rectangle analyst_server
    rectangle analyst_agent
    rectangle analyst_model
    rectangle ems_hmi
    rectangle mlflow
    queue hivemq
    rectangle prometheus
    rectangle grafana
}

```

## E2E Testing

Nightly job in a staging environment. Exercises the full data flow across all services.

```plantuml
participant ci_runner
participant device_api
participant industrial_gateway
participant industrial_fixtures
queue broker
participant ems_hmi
database timeseries
participant analyst_api

ci_runner -> device_api: POST /topology (test DTM)
device_api -> device_api: generate AsyncAPI spec + /topology/view projection

industrial_gateway -> device_api: GET /asyncapi
ems_hmi -> device_api: GET /asyncapi
ems_hmi -> device_api: GET /topology/view

== fixture telemetry ==
industrial_fixtures -> broker: publish sim measurements
broker -> industrial_gateway: forward
broker -> ems_hmi: forward
broker -> timeseries: persist

== analyst ==
analyst_api -> timeseries: query
analyst_api -> ci_runner: predictions + chat response

== assertions ==
ci_runner -> timeseries: verify measurements persisted
ci_runner -> ems_hmi: verify render (headless)
ci_runner -> analyst_api: verify predictions + chat
```

# Project Structure

## Repositories

The following repositories make up the EMS suite:

- [`ems-industrial-fixtures`](https://gitlab.com/arcnode-io/ems-industrial-fixtures) 🦀
- [`dlr-rtu-firmware`](https://gitlab.com/arcnode-io/dlr-rtu-firmware) 🐍
- [`dlr-tap-regulator-sim`](https://gitlab.com/arcnode-io/dlr-tap-regulator-sim) 🦀
- [`dlr-rtu-pcb`](https://gitlab.com/arcnode-io/dlr-rtu-pcb) 🐍
- [`mock-derms-dispatch-api`](https://gitlab.com/arcnode-io/mock-derms-dispatch-api) ☕
- [`ems-industrial-gateway`](https://gitlab.com/arcnode-io/ems-industrial-gateway) 🦀
- [`ems-der-control-api`](https://gitlab.com/arcnode-io/ems-der-control-api) ☕
- [`ems-device-api`](https://gitlab.com/arcnode-io/ems-device-api) 🌊
- [`ems-hmi`](https://gitlab.com/arcnode-io/ems-hmi) 🌊
- [`ems-analyst`](https://gitlab.com/arcnode-io/ems-analyst) 🐍
- [`ems-analyst-model`](https://gitlab.com/arcnode-io/ems-analyst-model) 🐍
