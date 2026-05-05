# EMS 🏭⚡


# Overview

This series of repos is designed to teach real world skills when implementing modern energy systems. It covers
everything from the sensors, to the UI (mobile and web), to predictive models, to AI agents. The three languages used
are


- 🌊 Typescript: Realtime UIs and APIs
- 🐍 Python: LLM/ML Apps and Platform Engineering
- 🦀 Rust: Grid Protocols and Embedded Systems


## Project Description

The EMS (Energy Management System) suite is the software that runs on a deployed Arcnode stack. It allows you to model different smart grid systems. For example, you could model a dynamic dlr system with a datacenter load. The bess could be modeled with canbus measurements and the datacenter could be modeled as snmp and redfish readings.

## Decisions

- [ADR-001: System Architecture](system_adr.md)
- [ADR-002: MQTT Topic Structure and Payload Conventions](topic_structure_adr.md)

# Diagrams

## Deployment*

```plantuml
collections "mock_industrial_protocols**" as mock_industrial_protocols
rectangle  "front of the meter" #line.dashed {
  collections dlr_sensors
  rectangle phase_shift_transformer
}
cloud third_party_apis
rectangle cluster #line.dashed {
    rectangle industrial_gateway
    rectangle line_controller
    rectangle device_api
    database timeseries
    database vector
    database graph
    collections analyst_api
    database document
    collections ems_hmi
    person llm
    rectangle domain_mcp_server
}
dlr_sensors -d-> line_controller: mqtt
line_controller -u-> phase_shift_transformer: mqtt
industrial_gateway -u---> mock_industrial_protocols: modbus\nsnmp\ndnp3\nredfish\ncanbus
line_controller -d-> device_api: http
industrial_gateway -d-> device_api: http
ems_hmi -u-> device_api: http
device_api -r-> document: sql
analyst_api -l-> timeseries: sql
llm -d-> domain_mcp_server: mcp
domain_mcp_server -d-> vector: sql
domain_mcp_server -d-> graph: cypher
ems_hmi -u-> analyst_api: http
analyst_api -d-> llm: http
llm -l-> third_party_apis: http
```
> &ast; MQTT broker ommited for simplicity <br>
> &ast;&ast; canbus, modbus, dnp3, snmp, and redfish, 
 
## Sequence

```plantuml
participant ems_startup
participant device_api
database document
participant industrial_gateway
participant line_controller
participant broker
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
ems_startup -> device_api: mount /etc/ems/dtm.json (from ISO/CFN payload)
device_api -> device_api: load DTM from disk
device_api -> document: generate AsyncAPI v3 spec
== distribute topics ==
industrial_gateway -> device_api: GET /asyncapi
line_controller -> device_api: GET /asyncapi
ems_hmi -> device_api: GET /asyncapi
ems_hmi -> device_api: GET /topology\n(devices + buses[] for module browser + SLD)
==  initialize messaging ==
industrial_gateway -> broker: pub grid protocols
line_controller -> broker: pub direct sensor data
broker -> timeseries: writes to db
broker -> ems_hmi: renders live data
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

## Topology Update Sequence

Topology changes only via redeployment. The DTM is delivered by `platform-api` (CFN update or new ISO) and posted programmatically to `device-api`. The HMI has no topology editor.

```plantuml
participant platform_api
participant device_api
database document
queue broker
participant industrial_gateway
participant line_controller
participant ems_hmi

platform_api -> device_api: POST /topology (new DTM)
device_api -> document: regenerate AsyncAPI v3 spec (semver bump)
device_api -> broker: publish system/topology_changed { ts, version }
broker -> industrial_gateway: forward
broker -> line_controller: forward
broker -> ems_hmi: forward
industrial_gateway -> device_api: GET /asyncapi
line_controller -> device_api: GET /asyncapi
ems_hmi -> device_api: GET /asyncapi
industrial_gateway -> industrial_gateway: diff + reconcile topic subs
line_controller -> line_controller: diff + reconcile topic subs
ems_hmi -> ems_hmi: diff + reconcile topic subs
```

All topology changes — including renames, display-name overrides, and sizing changes — flow through the configurator → platform-api → edp-api → device-api path. No in-EMS topology editing.

## Cloud Deployment (AWS)

```plantuml
rectangle ecs_cluster #line.dashed {
    rectangle analyst_agent
    rectangle analyst_model
    rectangle device_api
    queue emqx
    rectangle ems_hmi
    rectangle mlflow
    rectangle prometheus
    rectangle grafana
    rectangle industrial_gateway
    rectangle analyst_server
}

database s3

rectangle managed_persistence #line.dashed {
    cloud timescale_cloud 
    cloud neon_vector
    cloud neon_document
    cloud neo4j_aura 
}

rectangle third_party_apis #line.dashed {
    cloud ercot_api
    cloud openweather
    cloud yes_energy
    cloud permutable
}

```

## On-Prem Deployment (ISO)

```plantuml
rectangle daemons #line.dashed {
    database postgres_timescale 
    database postgres_document 
    database postgres_vector
    database neo4j
    database minio
    rectangle ollama {
      rectangle llama
      rectangle nomic_embed
    }
    }

    rectangle docker_runtime #line.dashed {
    rectangle device_api
    rectangle industrial_gateway
    rectangle analyst_server
    rectangle analyst_agent
    rectangle analyst_model
    rectangle ems_hmi
    rectangle mlflow
    queue emqx
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
device_api -> device_api: generate AsyncAPI spec

industrial_gateway -> device_api: GET /asyncapi
industrial_fixtures -> device_api: GET /asyncapi
ems_hmi -> device_api: GET /asyncapi

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
- [`ems-line-controller`](https://gitlab.com/arcnode-io/ems-line-controller) 🐍🦀
- [`ems-line-controller-dlr`](https://gitlab.com/arcnode-io/ems-line-controller-dlr) 🐍
- [`ems-line-controller-pst`](https://gitlab.com/arcnode-io/ems-line-controller-pst) 🦀
- [`ems-industrial-gateway`](https://gitlab.com/arcnode-io/ems-industrial-gateway) 🦀
- [`ems-device-api`](https://gitlab.com/arcnode-io/ems-device-api) 🌊
- [`ems-hmi`](https://gitlab.com/arcnode-io/ems-hmi) 🌊
- [`ems-analyst-api`](https://gitlab.com/arcnode-io/ems-analyst-api) 🐍
- [`ems-analyst-model`](https://gitlab.com/arcnode-io/ems-analyst-model) 🐍
- [`ems-analyst-agent`](https://gitlab.com/arcnode-io/ems-analyst-agent) 🐍
- [`ems-analyst-server`](https://gitlab.com/arcnode-io/ems-analyst-server) 🐍

