# Sub-Project C3 — MQTT topology_changed Broadcast Design

**Status:** Draft
**Date:** 2026-05-10
**Sub-project:** C3 of {C1: dropped, C2: monotonic version (PR 5 shipped), C3: MQTT broadcast}
**Depends on:** C2 (PR 5 — version field on Topology row)

## Context

PR 5 (sub-project C2) lands monotonic version on every `/topology` save. The version is visible via `GET /asyncapi` (`info.version`). What's missing: consumers (gateway, line-controller, HMI) don't know when the version has changed unless they poll. Per [ADR-002 §10](../../../topic_structure_adr.md):

> "`system/topology_changed` carries only `{ts, version}` — consumers re-fetch the full spec and compute the diff client-side."

This sub-project lands the broadcast: device-api connects to the deployment's MQTT broker at boot and publishes `system/topology_changed { ts, version }` after every successful `TopologyService.save`. Consumers subscribe to that topic and re-fetch `/asyncapi` + `/topology` on every event.

## Goals

1. `ems-device-api` connects to a configurable MQTT broker on app boot via the in-process `mqtt` Node client. Lives as a NestJS-injectable singleton.
2. After every successful `TopologyService.save` row insertion, fires `client.publish("system/topology_changed", JSON.stringify({ ts, version }), { qos: 1, retain: false })` per ADR-002 §3 + §11.
3. Anonymous connection (no auth) per the v1-no-auth project rule.
4. Failure-tolerant: broker unreachable at boot or at publish-time logs a warning; device-api continues. Save semantics unchanged.
5. Integration test against `emqx/emqx:latest` testcontainer asserts a real subscriber receives the broadcast.

## Non-goals

- TLS / mqtts:// — production deployments override `mqttBrokerUrl` to `mqtts://...` via env config but the code is transport-agnostic.
- MQTT Last Will & Testament — device-api is a publisher; LWT is for measurement publishers.
- mqtt v5 features beyond basic publish.
- Authentication — explicitly anonymous per v1-no-auth memory.
- Diff classification in payload — ADR-002 §10 says payload is `{ts, version}`, no diff.

## Design

### Configuration

`cfg.yml` adds:

```yaml
local:
  mqttBrokerUrl: mqtt://localhost:1883
beta:
  mqttBrokerUrl: mqtt://emqx:1883
```

Production override via env (cloud + ISO): customer sets `MQTT_BROKER_URL=mqtts://...` if TLS needed.

`Config` schema in `src/config.ts` adds:

```typescript
mqttBrokerUrl: z.string().url(),
```

### `MqttClientService`

NestJS-injectable, single instance, lifecycle hooks:

```typescript
import { Injectable, Logger, OnModuleInit, OnModuleDestroy } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { MqttClient, connect } from "mqtt";

@Injectable()
export class MqttClientService implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(MqttClientService.name);
  private client?: MqttClient;

  constructor(private readonly cfg: ConfigService) {}

  async onModuleInit(): Promise<void> {
    const url = this.cfg.get<string>("mqttBrokerUrl")!;
    this.client = connect(url, { reconnectPeriod: 5000 });
    this.client.on("connect", () => this.logger.log(`mqtt connected ${url}`));
    this.client.on("error", (err) =>
      this.logger.warn(`mqtt error: ${err.message}`),
    );
  }

  async onModuleDestroy(): Promise<void> {
    await this.client?.endAsync();
  }

  publishTopologyChanged(version: string): void {
    if (!this.client?.connected) {
      this.logger.warn(
        `mqtt not connected; dropping topology_changed v${version}`,
      );
      return;
    }
    const payload = JSON.stringify({
      ts: new Date().toISOString(),
      version,
    });
    this.client.publish(
      "system/topology_changed",
      payload,
      { qos: 1, retain: false },
      (err) => {
        if (err) this.logger.warn(`mqtt publish failed: ${err.message}`);
      },
    );
  }
}
```

**Failure semantics:**
- Broker unreachable at boot → `mqtt.connect` returns a client that retries every 5s. `onModuleInit` resolves immediately; broker connects when available.
- `publishTopologyChanged` called while disconnected → log warning, return; no buffer, no replay. ADR-002 §11 says system family is `retain: false` and "transient triggers; old events should not fire on new connect" — so dropping during outage is correct.
- `publish` callback errors → log warning. POST /topology already returned 201; broadcast is best-effort.

### `MqttModule`

```typescript
@Module({
  imports: [ConfigModule],
  providers: [MqttClientService],
  exports: [MqttClientService],
})
export class MqttModule {}
```

`TopologyModule` imports `MqttModule`. `TopologyService` constructor gains `MqttClientService` parameter.

### `TopologyService.save` hook

After the existing `repo.save(row)`:

```typescript
async save(dtm: DtmType): Promise<Topology> {
  // ... existing version computation
  const row = await this.repo.save(this.repo.create({ dtm: ..., version }));
  this.mqtt.publishTopologyChanged(version);
  return row;
}
```

Fire-and-forget. Doesn't block POST /topology response.

### File structure

**Create:**
- `src/mqtt/mqtt.client.service.ts` — service class + lifecycle hooks. ~80 lines.
- `src/mqtt/mqtt.client.service.test.ts` — unit tests with mocked client.
- `src/mqtt/mqtt.module.ts` — DI wiring. ~10 lines.
- `tests/mqtt.test.ts` — integration with emqx testcontainer + real subscriber.

**Modify:**
- `src/config.ts` — add `mqttBrokerUrl: z.string().url()`.
- `cfg.yml` — add `mqttBrokerUrl` to `local:` and `beta:`.
- `src/topology/topology.module.ts` — import `MqttModule`.
- `src/topology/topology.service.ts` — inject `MqttClientService`, call `publishTopologyChanged` after save.
- `src/topology/topology.service.test.ts` — extend tests with mocked mqtt client.
- `tests/fixtures/containers.ts` — add `startEmqx()` helper.
- `package.json` — `mqtt` dependency.

### Test plan

#### Unit (`src/mqtt/mqtt.client.service.test.ts`)

Mock the `mqtt` package's `connect` function:

```typescript
it("publishTopologyChanged sends correct topic + payload", () => {
  // Arrange — mock client connected
  const publishMock = mock.fn();
  const fakeClient = {
    connected: true,
    on: mock.fn(),
    publish: publishMock,
  };
  const cfg = { get: () => "mqtt://test:1883" };
  const svc = new MqttClientService(cfg as never);
  (svc as never).client = fakeClient;
  // Act
  svc.publishTopologyChanged("1.0.42");
  // Assert
  assert.equal(publishMock.mock.calls.length, 1);
  const [topic, payload, opts] = publishMock.mock.calls[0].arguments;
  assert.equal(topic, "system/topology_changed");
  const parsed = JSON.parse(payload);
  assert.equal(parsed.version, "1.0.42");
  assert.match(parsed.ts, /^\d{4}-\d{2}-\d{2}T/);  // ISO8601
  assert.deepEqual(opts, { qos: 1, retain: false });
});

it("publishTopologyChanged drops + logs warning when not connected", () => {
  const fakeClient = { connected: false, on: mock.fn(), publish: mock.fn() };
  const cfg = { get: () => "mqtt://test:1883" };
  const svc = new MqttClientService(cfg as never);
  (svc as never).client = fakeClient;
  // Act
  svc.publishTopologyChanged("1.0.42");
  // Assert
  assert.equal((fakeClient.publish as { mock: { calls: unknown[] } }).mock.calls.length, 0);
});
```

#### Unit (`src/topology/topology.service.test.ts` extension)

Add a mock `MqttClientService` to existing service tests; assert `publishTopologyChanged` called with the new version after save.

#### Integration (`tests/mqtt.test.ts`)

Spin up emqx testcontainer + Postgres testcontainer. Bootstrap NestJS app pointing `mqttBrokerUrl` at the emqx port. Subscribe to `system/topology_changed` from a separate `mqtt` client. POST `/topology`. Assert subscriber receives a message with the expected version.

```typescript
test("save → emqx broadcasts system/topology_changed", async () => {
  // Arrange — emqx + postgres testcontainers
  const broker = await startEmqx();
  const pg = await startPostgres("test", { dbname: "postgres" });
  process.env["MQTT_BROKER_URL"] = `mqtt://localhost:${broker.port}`;
  // ... bootstrap app
  const subscriber = mqtt.connect(broker.url);
  await new Promise((res) => subscriber.on("connect", res));
  const messages: string[] = [];
  subscriber.subscribe("system/topology_changed");
  subscriber.on("message", (_topic, payload) => messages.push(payload.toString()));

  // Act
  await http.post("/topology").send(SAMPLE_DTM).expect(201);
  await new Promise((r) => setTimeout(r, 500));  // give the broadcast time

  // Assert
  assert.equal(messages.length, 1);
  const parsed = JSON.parse(messages[0]);
  assert.equal(parsed.version, "1.0.0");
});
```

#### `tests/fixtures/containers.ts`

```typescript
async function startEmqx(): Promise<Container> {
  const started = await new GenericContainer("emqx/emqx:latest")
    .withExposedPorts(1883)
    .withWaitStrategy(
      Wait.forLogMessage("Listener tcp:default on 0.0.0.0:1883 started."),
    )
    .start();
  const port = started.getMappedPort(1883);
  return {
    host: "localhost",
    port,
    url: `mqtt://localhost:${port}`,
    stop: () => started.stop(),
  };
}
```

Mirrors existing `startContainer` / `startPostgres` / `startLocalStack` patterns. Wait-for-log string copied from existing emqx testcontainer pattern at `/home/resister/fullstack-energy/grid-gateway/tests/integration_test.rs`.

## Migration

No DB migration. No breaking change. Additive only — when `mqttBrokerUrl` is configured, broadcasts happen; clients subscribed to `system/topology_changed` start receiving events.

Consumers (gateway, line-controller, HMI) currently don't subscribe — they're future work. C3 produces the events; consumers light up later.

## Risks

| Risk | Mitigation |
|---|---|
| Broker outage drops broadcasts | Per ADR-002 §11 system family is retain: false; lost events stay lost. Consumers must be idempotent — they already are (re-fetch + diff client-side). |
| `mqtt.connect` blocks NestJS boot | Library returns client immediately; reconnect loop is async. Verified in pattern. |
| Anonymous broker exposure | ADR-002 §13 covers this — broker is per-deployment private. Auth deferred to ADR-004. |
| Memory leak from unhandled events | `mqtt` lib + Node event loop are well-tested. No state retained across publishes. |

## Implementation Order (Hint to writing-plans)

Roughly:
1. cfg.yml + Config schema add `mqttBrokerUrl`
2. Add `mqtt` npm dependency
3. `MqttClientService` + unit tests (mocked)
4. `MqttModule`
5. Wire into `TopologyModule` + `TopologyService.save`
6. `startEmqx` testcontainer helper
7. Integration test against real emqx
8. Push + CI

Single PR. ~7 commits.
