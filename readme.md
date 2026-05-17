# EMS 🏭⚡


# Overview

This series of repos is designed to teach real world skills when implementing modern energy systems. It covers
everything from the sensors, to the UI (mobile and web), to predictive models, to AI agents. The three languages used
are


- 🌊 Typescript: Realtime UIs and APIs
- 🐍 Python: LLM/ML Apps and Platform Engineering
- 🦀 Rust: Grid Protocols and Embedded Systems


## Project Description

The EMS (Energy Management System) suite is the software that runs on a deployed Arcnode stack. It allows you to model different smart grid systems. For example, you could model a dynamic dlr system with a datacenter load. The bess could be modeled with modbus measurements and the datacenter could be modeled as snmp and redfish readings.

## Decisions

- [System ADR](system_adr.md) — architecture, MQTT contract, boot

# Diagrams

## Deployment*

```plantuml
collections "mock_industrial_protocols**" as mock_industrial_protocols
rectangle  "front of the meter" #line.dashed {
  rectangle dlr_operating_envelope
  rectangle dlr_pst_sim
}
cloud third_party_apis
rectangle cluster #line.dashed {
    rectangle industrial_gateway
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
dlr_operating_envelope -- dlr_pst_sim: mqtt
dlr_operating_envelope -- industrial_gateway: dnp3  
industrial_gateway -u---> mock_industrial_protocols: modbus\nsnmp\ndnp3\nredfish\nbacnet
industrial_gateway --> device_api: http
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
 
## Sequence

```plantuml
participant device_api
database document
participant broker
participant industrial_gateway
participant dlr_operating_envelope
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
dlr_operating_envelope -> industrial_gateway: dnp3
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

## Cloud Deployment — Commercial

```plantuml
rectangle ec2_docker_compose #line.dashed {
    rectangle analyst_agent
    rectangle analyst_model
    rectangle device_api
    queue hivemq
    rectangle ems_hmi
    rectangle mlflow
    rectangle prometheus
    rectangle grafana
    rectangle industrial_gateway
    rectangle analyst_server
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

```

## Cloud Deployment — Defense / Sovereign

```plantuml
rectangle ec2_docker_compose #line.dashed {
    rectangle analyst_agent
    rectangle analyst_model
    rectangle device_api
    queue hivemq
    rectangle ems_hmi
    rectangle mlflow
    rectangle prometheus
    rectangle grafana
    rectangle industrial_gateway
    rectangle analyst_server
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

```

## On-Prem Deployment (ISO)

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
- [`dlr-operating-envelope`](https://gitlab.com/arcnode-io/dlr-operating-envelope) 🐍
- [`dlr-pst-sim`](https://gitlab.com/arcnode-io/dlr-pst-sim) 🦀
- [`dlr-pcb`](https://gitlab.com/arcnode-io/dlr-pcb) 🐍
- [`ems-industrial-gateway`](https://gitlab.com/arcnode-io/ems-industrial-gateway) 🦀
- [`ems-device-api`](https://gitlab.com/arcnode-io/ems-device-api) 🌊
- [`ems-hmi`](https://gitlab.com/arcnode-io/ems-hmi) 🌊
- [`ems-analyst-api`](https://gitlab.com/arcnode-io/ems-analyst-api) 🐍
- [`ems-analyst-model`](https://gitlab.com/arcnode-io/ems-analyst-model) 🐍
- [`ems-analyst-agent`](https://gitlab.com/arcnode-io/ems-analyst-agent) 🐍
- [`ems-analyst-server`](https://gitlab.com/arcnode-io/ems-analyst-server) 🐍
