# Industrial Gateway Stub — Contract Validation Design

**Goal:** Build a minimal `ems-industrial-gateway` binary + `mock-modbus-server` fixture that, run together with a real `ems-device-api` and broker, validate the AsyncAPI/Modbus/MQTT contract end-to-end.

**Why this exists:** `ems-device-api` ships PRs 1–6 (template catalog, DTM persistence, /asyncapi generation, Day-1 S3 boot, monotonic version bump, MQTT broadcast). All tested in isolation. None of it has been exercised by an actual consumer. This design defines the smallest consumer slice that proves the device-api contract is consumable.

**Sub-project boundary:** First slice ("Tier 1") — one Modbus device, one measurement, single read, single publish. Beacon-driven reconcile + multi-protocol + continuous polling are explicit non-goals for this spec.

---

## Architecture

Two repos involved:

| Repo | Role |
|---|---|
| `ems-industrial-gateway` | Real Rust binary; HTTP client + Modbus client + MQTT pub/sub; owns the end-to-end test |
| `ems-industrial-fixtures` | Rust workspace; `mock-modbus-server` binary + Dockerfile published to GitLab registry |

The e2e test lives in `ems-industrial-gateway/tests/` and spins up four testcontainers:

```
gateway_test (cargo test, real gateway binary runs in-process)
        │
        ├─ HTTP GET /asyncapi  ──▶ device-api (testcontainer, prod image)
        │                              │
        │                              ├─▶ postgres (testcontainer)
        │                              └─▶ emqx (testcontainer)
        │
        ├─ MQTT sub system/topology_changed ▶ emqx (same)
        ├─ MQTT pub sites/.../kwh_delivered ▶ emqx (same)
        │
        └─ Modbus TCP read ─────────▶ mock-modbus-server (testcontainer)
```

Dynamic ports for every container; gateway cfg is constructed from the testcontainer-assigned ports at test time.

### Client/server roles per actor

| Actor | HTTP | MQTT | Modbus |
|---|---|---|---|
| device-api | server | publisher (system/topology_changed) | — |
| gateway | client | sub system/topology_changed, pub sites/... | client |
| mock-modbus-server | — | — | server |
| test harness | client (seeds DTM) | subscriber (asserts publish) | — |

### Contract surfaces

Three contracts converge at the gateway and all get exercised in this single test:

1. **AsyncAPI v3 spec shape** (device-api → gateway). Modelina-rust generates Rust structs at gateway build time. Mismatch = build break.
2. **Modbus TCP RFC** (mock-modbus-server → gateway). Library-tested via `rodbus`.
3. **MQTT topic + payload** (gateway → broker → test subscriber). Asserted by the test.

---

## Test flow

### Seeded DTM

```yaml
deployment_uuid: 11111111-1111-1111-1111-111111111111
ems_mode: sim
sizing_params:
  P_compute_total_kW: 100
  E_BESS_total_kWh: 200
  T_coolant_setpoint_C: 18
devices:
  meter_01:
    device_id: meter_01
    template: revenue_meter
    parent: null
    connection:
      host: <mock-modbus-server container hostname>
      port: <mock-modbus-server mapped port>
      unit_id: 1
    blocking: []
buses: []
templates_used:
  revenue_meter: <embedded revenue_meter template>
```

### Order of operations

1. **Parallel startup**: postgres + emqx + mock-modbus-server containers.
2. **device-api container** boots pointed at postgres + emqx (envs: `POSTGRES_*`, `MQTT_BROKER_URL`). Test waits via `Wait::ForHttp` strategy on `/asyncapi` endpoint until healthy.
3. **POST DTM** to device-api `/topology` (test uses `reqwest`). device-api persists, regenerates spec, publishes `system/topology_changed { ts, version: "1.0.0" }`.
4. **Spawn gateway** in-process via `tokio::spawn(app::run(cfg))` with cfg pointing at device-api URL + broker URL + site_id="test_site".
5. **Gateway boots**:
   - subscribes `system/topology_changed`
   - HTTP GET /asyncapi with exponential backoff (1s→2s→4s→8s→16s→32s, ~60s cap)
   - parses spec into modelina-generated structs
   - extracts `x-protocol-source.meter_01.kwh_delivered` → `{protocol: modbus_tcp, function_code: 3, address: 4000, data_type: int32, word_order: high_low, scale: 1.0, offset: 0.0}`
   - connects to `<mock-modbus-server>:<port>` unit_id=1
   - reads holding registers 4000-4001
   - decodes int32 with word_order=high_low: `(reg[4000] << 16) | reg[4001]` = 1_000_000
   - applies scale/offset: `1_000_000 * 1.0 + 0.0` = 1_000_000.0
   - publishes `sites/test_site/devices/meter_01/measurements/kwh_delivered/watt_hours` payload `{ts: <ISO8601>, value: 1000000.0}` QoS=1
6. **Test asserts**: harness-side MQTT subscriber receives the exact topic + payload (within 5s timeout).
7. **Teardown**: containers stop, gateway task aborts.

### Failure modes the test must catch

- A testcontainer fails to boot → test fails fast
- Modbus client decodes wrong value (scaling, word order, register address) → assertion fails
- MQTT publish lost / wrong topic / wrong payload → assertion fails

Modelina catches spec/struct mismatches at gateway build time, not runtime — no hand-rolled defensive parsing needed.

---

## Gateway component breakdown

```
ems-industrial-gateway/
├── Cargo.toml          # rodbus 1.4, paho-mqtt 0.13, reqwest 0.12, tokio,
│                       # tracing, serde, serde_json, anyhow, backoff
│                       # build-dep: invokes @asyncapi/modelina at build time
├── build.rs            # runs `npx @asyncapi/modelina generate` against
│                       # contracts/asyncapi-snapshot.json (checked-in copy
│                       # of a representative /asyncapi from device-api)
│                       # → emits src/generated/asyncapi_types.rs
├── cfg.yml             # device_api_url, broker_url, site_id, log_level
├── src/
│   ├── main.rs         # tokio entry, init tracing, calls app::run(cfg)
│   ├── app.rs          # boot orchestration: load cfg → init clients
│   │                   #   → sub beacon → fetch /asyncapi → poll → publish
│   ├── config.rs       # cfg.yml deserialize
│   ├── http/
│   │   ├── mod.rs
│   │   └── client.rs   # GET /asyncapi with exponential backoff
│   │                   # returns modelina-typed AsyncApiDoc
│   ├── modbus/
│   │   ├── mod.rs
│   │   └── client.rs   # rodbus client; read_holding(addr, count) -> Vec<u16>
│   │                   # decode_int32(words, word_order) -> i32
│   │                   # apply_scale_offset(raw, scale, offset) -> f64
│   ├── mqtt/
│   │   ├── mod.rs
│   │   ├── publisher.rs  # publish FloatSample to sites/.../measurements/...
│   │   └── subscriber.rs # sub system/topology_changed (logs only in v1)
│   └── generated/      # gitignored, emitted by build.rs
│       └── asyncapi_types.rs
├── contracts/
│   └── asyncapi-snapshot.json   # checked-in /asyncapi sample, modelina input
└── tests/
    ├── fixtures/
    │   ├── containers.rs        # 4 testcontainer helpers
    │   └── seed_dtm.json        # DTM payload to POST
    └── e2e.rs                   # the single integration test
```

**Contract snapshot drift detection:** `contracts/asyncapi-snapshot.json` is the build-time source of truth for the gateway's Rust types. If device-api's runtime output diverges from this shape, serde deserialization in `fetch_asyncapi` fails, and the e2e test catches it. Refreshing the snapshot is a deliberate PR action whenever the device-api contract intentionally changes.

### Critical interfaces

```rust
// http/client.rs
pub async fn fetch_asyncapi(url: &str) -> Result<AsyncApiDoc, anyhow::Error>;
// → exponential backoff on connection refused / 5xx / timeout
// → returns modelina-typed AsyncApiDoc

// modbus/client.rs
pub async fn read_holding(host: &str, port: u16, unit_id: u8, addr: u16, count: u16) -> Result<Vec<u16>, anyhow::Error>;
pub fn decode_int32(words: &[u16], word_order: WordOrder) -> i32;
pub fn apply_scale_offset(raw: i32, scale: f64, offset: f64) -> f64;

// mqtt/publisher.rs
pub fn publish_measurement(client: &MqttClient, topic: &str, sample: FloatSample) -> Result<(), anyhow::Error>;

// app.rs
pub async fn run(cfg: Config) -> Result<(), anyhow::Error>;
```

### File-size budget (per arcnode CLAUDE.md ≤200 LOC)

All files target well under 200 LOC. `app.rs` orchestrator likely ~80; each client ~80–150; `e2e.rs` test ~150–200 (split if it grows).

### Consumer-side resilience

Boot-time race when 4 containers come up together: device-api (NestJS+Node) takes ~30-60s. Gateway must tolerate this. Two layers:

1. **Test layer**: `Wait::ForHttp` on device-api before spawning gateway.
2. **Gateway layer**: exponential backoff in `http::client::fetch_asyncapi`. Same logic also applies to broker connect + modbus connect for prod parity.

Per saved feedback memory: producer fails loud, consumer owns retry/backoff. This is consumer code → retry is correct.

### What's NOT in v1

- Multiple devices (just meter_01)
- Multiple measurements per device (just kwh_delivered)
- Beacon-driven reconcile diffing (sub the topic, log it; no diff logic yet)
- Continuous poll loop (single read on boot, log → exit; matches reference's `mode=development` shortcut)
- Commands / north→south
- Other protocols (SNMP, DNP3, Redfish, CANbus)

---

## Fixture component breakdown

```
ems-industrial-fixtures/
├── mock-modbus-server/
│   ├── Cargo.toml          # rodbus 1.4, tokio, tracing (minimal)
│   ├── Dockerfile          # 2-stage: rust:1.93 builder → debian:slim runtime
│   ├── cfg.yml             # host, port, unit_id, log_level
│   ├── src/
│   │   ├── main.rs         # tokio entry, init tracing, spawn_tcp_server_task
│   │   ├── handler.rs      # MeterHandler impl rodbus::RequestHandler
│   │   │                   #   (only overrides read_holding_register)
│   │   └── registers.rs    # static u16 map for revenue_meter
│   └── tests/
│       └── server_test.rs  # rodbus client → connect → read regs → assert
└── docker-compose.yml      # (existing) orchestrates all 5 mocks for local dev
```

### Register map (v1)

```rust
// int32 1_000_000 = 0x000F4240 → high_low word_order:
//   holding[4000] = 0x000F (high word, 15)
//   holding[4001] = 0x4240 (low word, 16960)
const HOLDING: &[(u16, u16)] = &[
    (4000, 0x000F),
    (4001, 0x4240),
];
```

### Handler

```rust
struct MeterHandler {
    holding: HashMap<u16, u16>,  // sparse, only populated regs
}

impl RequestHandler for MeterHandler {
    fn read_holding_register(&self, address: u16) -> Result<u16, ExceptionCode> {
        self.holding.get(&address).copied().ok_or(ExceptionCode::IllegalDataAddress)
    }
}
```

### Image publishing

- GitLab CI in `ems-industrial-fixtures` repo pushes `registry.gitlab.com/arcnode-io/ems-industrial-fixtures/mock-modbus-server:latest` on every main push.
- Gateway test pulls this image via `testcontainers::GenericImage`.
- Local dev: `docker build` locally + tag with the same name; testcontainers will use the local image first.

### What's NOT in v1

- Multiple unit_ids (just unit_id=1)
- Dynamic register values (only static)
- Input registers, coils, discrete inputs (only holding registers FC=3)
- Other measurements beyond kwh_delivered

---

## Library choices

| Concern | Library | Notes |
|---|---|---|
| Modbus client (gateway) + server (fixture) | `rodbus 1.4` | Matches arcnode CLAUDE.md domain reference; same lib for both sides |
| MQTT pub/sub | `paho-mqtt 0.13` | Matches fullstack-energy reference; battle-tested |
| HTTP client | `reqwest 0.12` | Standard |
| Async runtime | `tokio 1.x` | Standard |
| Logging | `tracing` + `tracing-subscriber` | Matches arcnode conventions |
| AsyncAPI Rust types | `@asyncapi/modelina` (Rust generator) | Build-time codegen with serde macros |
| HTTP backoff | `backoff` crate | Or hand-rolled if too heavy |
| Errors | `anyhow` (binaries) + `thiserror` (library boundaries) | Standard |
| Testcontainers | `testcontainers-rs` | Standard |

---

## Pre-flight checklist before plan execution

1. Verify `ems-device-api` Docker image is published to `registry.gitlab.com/arcnode-io/ems-device-api:latest` (or equivalent tag). Confirm test can pull.
2. Confirm `@asyncapi/modelina` Rust generator emits compilable structs for the actual `/asyncapi` shape device-api produces. Quick spike: feed a real /asyncapi sample through modelina, inspect output.
3. Confirm `rodbus 1.4` API matches reference usage (server `spawn_tcp_server_task` + client `connect_tcp` patterns).
