# System ADR

Three sections:

- §1–§7: Architecture
- §8–§21: MQTT contract
- §22: Boot

Diagrams: see [`readme.md`](readme.md).

## Architecture

### §1. MQTT as central event bus

*Why.* SCADA shops already speak MQTT. AsyncAPI describes the topics. Picking a non-MQTT bus would mean re-teaching every integrator we work with.

All telemetry flows over MQTT (HiveMQ CE). AsyncAPI documents the topics.

### §2. No API gateway

*Why.* A gateway adds a service we'd have to deploy, secure, observe, and recover. Service URLs are known per-deployment; the SPA can call them directly. Per-service auth is repetitive but cheap.

Frontend SPA calls backend services directly. Each service owns its own auth, CORS, rate limiting.

### §3. Dynamic configuration from device-api

*Why.* Commissioning a device or swapping a connection should not require redeploying the gateway and HMI. They re-fetch from device-api and pick up the change.

Protocol gateways and HMI fetch topology and AsyncAPI over HTTP from `ems-device-api`. No static config files distributed. Exponential backoff on startup.

### §4. Polyglot microservices

*Why.* Rust where memory safety on edge hardware matters. Python where the ML ecosystem lives. TypeScript where the web ecosystem lives. Standardizing on one would force at least one team to fight their toolchain.

- Rust — protocol handling (`industrial-fixtures`, `industrial-gateway`, `line-controller-pst`)
- Python — ML / agents / `line-controller-dlr`
- TypeScript — `device-api`, `hmi`

### §5. Two deployment paths: CFN or ISO

*Why.* Operators run the stack themselves. Kubernetes is too much to learn for a single-site EMS. CFN covers AWS-connected sites; ISO covers air-gapped defense forward. Same image, different wrapper.

| Path | Trigger | Mechanism |
|---|---|---|
| Cloud CFN | `aws_partition ∈ {standard, govcloud}` | Per-order CloudFormation YAML download from delivery portal; operator runs `aws cloudformation create-stack` locally |
| Air-gapped ISO | `aws_partition == none` or `deployment_context == defense_forward` | Self-contained image with DTM, SSH key, `ems_mode` baked in |

No Kubernetes. No IaC on operator side. `platform-api` composes the per-order CFN template on demand and uploads to S3 as `orders/{id}/ems-stack.yaml`. Each YAML bakes `deployment_uuid`, `dtm_url`, `ems_mode` directly.

### §6. SLD topology in DTM, rendered programmatically by HMI

*Why.* Per-site hand-drawn SVGs rot the moment topology changes. Topology is data; the diagram is a function of the data.

Electrical bus topology is encoded in a `buses[]` block in the DTM. `edp-api` computes `buses[]` from the sizing payload and emits it alongside `devices` in the same job. IEC 61850 alignment: `buses[].id` = `ConnectivityNode`, `buses[].type` = `VoltageLevel` domain, `buses[].members[]` = `Terminal` references, `port` = Terminal name. Bus bars are not devices — no `device_id`, no MQTT topics. HMI renders bars as horizontals with device nodes hanging off; live measurements overlay each device node keyed by `device_id`.

### §7. Cloud persistence: Defense and Commercial Variants
|                   | Commercial                                     | Defense / sovereign |
|---|---|---|
| Document + Vector | Aurora                                         | Aurora              |
| Time series       | Tiger Cloud (URL param)                        | Aurora pg_partman   |
| Graph             | Neo4j Aura (URL param)                         | Neptune             |
| FTS               | Aura native                                    | AOSS                |
| Customer params   | 2 connection URLs (Tiger, Aura)                | zero                |
| Idle cost         | Aurora $0 + Tiger $0-30 + Aura $65 = $65-95/mo | $480/mo             |
| Customer ops      | sign up at Tiger + Aura, paste 2 URLs          | nothing             |
*Why.* Defense and sovereign deployments physically can't depend on non-AWS SaaS — FedRAMP authorization, supply-chain review, and GovCloud-region constraints rule out Tiger Cloud and Neo4j Aura. Commercial deployments face none of those constraints and benefit from cheaper, feature-richer managed vendors. The variant is gated by `aws_partition` / `deployment_context` from §5: `govcloud` partition or `defense_forward` context → defense column; everything else → commercial column. App code is identical across variants — abstract `TimeseriesClient` and `GraphClient` shims swap implementations at boot from the cfg.yml-injected connection details. Aurora, S3, and the ISO path (§5) are unaffected.

## MQTT contract

### §8. Two families: `measurements` and `commands`

*Why.* Everything a device emits vs everything sent to a device. Collapsing "measurement / status / state / calculation / alarm" into one family removes the gray zone — they're all just things the device emits.

- `measurements/` — everything a device emits (sensor reads, computed values, health, alarms, command acks)
- `commands/` — everything sent to a device

`system/` carries broker-internal events (topology changed, etc.) — not a family. Boolean/enum readings are measurements with `unit=none`. Derived quantities (DLR dynamic rating) are measurements.

### §9. Fixed-depth topics

*Why.* Wildcards are predictable when every topic of a kind has the same number of slots. The parser switches on one slot. Broker ACL hooks match on slot positions without payload introspection.

```
sites/{site_id}/devices/{device_id}/measurements/{measurement}/{unit}        # 6 segments
sites/{site_id}/devices/{device_id}/commands/{verb}/{target}/{unit}          # 7 segments
system/{event_type}                                                          # 2 segment
```

Parser switches on slot 4 (`family ∈ {measurements, commands}`). Unitless measurements use literal `none` in the unit slot. Verb enum: `[set, reset, clear, start, stop, enable, disable]`.

### §10. Unit vocabulary in topic, enum-locked in spec

*Why.* Operator at 3am should be able to read the unit without looking it up. Enum-lock at codegen time prevents `degC` vs `celsius` divergence — build fails if anyone tries.

```
volts | amps | watts | vars | voltamperes | watt_hours | hertz |
celsius | percent | watts_per_m2 | meters_per_second |
bar | liters_per_minute | none
```

Lowercase, plural where natural (`volts` not `volt`), short compounds (`watts_per_m2` not `watts_per_meter_squared`), `vars` for reactive power. No SI-prefixed duplicates (`watt_hours` only). Adding a unit is a spec change → patch-level bump → `system/topology_changed` → consumers re-fetch.

### §11. Gateway converts raw → engineering units at boundary

*Why.* One place to put the scaling math. Three consumers downstream stay simple. The spec records the math for regulatory audit.

Industrial protocols carry raw integers with vendor scaling. Gateway applies scale and offset; MQTT carries the engineering value. Scaling metadata lives in AsyncAPI under `x-source` per channel:

```yaml
x-source:
  protocol: modbus_tcp
  server_unit_id: 1
  register_type: holding
  address: 3000
  data_type: int16
  word_order: high_low
  scale: 0.1
  offset: 0
```

Gateway code is codegenned from the spec.

### §12. Strict `{ts, value}` payload; quality via time-joined `status`

*Why.* Quality information is rare and recoverable. Don't pay wire bytes for it on every sample. Industrial historians that need it run the time-join once in an adapter.

Every sample is `{ts, value}`. No quality, no source flags, no correlation IDs. Quality lives in a separate `status` measurement, joined by timestamp at query time (`LATERAL` join of status ≤ sample.ts). `ts` is RFC3339/ISO8601 string with `Z` suffix — ingests natively into Postgres `TIMESTAMPTZ`.

### §13. Sample schemas

*Why.* Four shapes cover every channel. Float for sensors, boolean for two-state things, enum for labeled states, trigger for commands with no payload.

```
FloatSample    { ts, value: number }
BooleanSample  { ts, value: boolean }
EnumSample     { ts, value: string }    # channel schema sets value.enum
TriggerSample  { ts }                   # commands only — reset, clear, fire-and-forget
```

`EnumSample` channel sets `value.enum` from the template `values:` block (e.g. `["AUTO", "MANUAL", "RUNPQ"]`). Gateway translates raw register integers to the label before publish; HMI renders the label it receives. No int-to-string logic in any consumer.

Verb → schema:

| Verb | Schema |
|---|---|
| set | Float / Boolean / Enum |
| start, stop, enable, disable | Boolean (persistent state) |
| reset, clear | Trigger |

### §14. Device templates as canonical vocabulary

*Why.* Without templates, every deployment re-fights the naming bikeshed and downstream codegen has nothing concrete to target. Templates are PR-gated so the argument happens once at template-design time, then the rest of the system codegens against a known shape.

*Why opinionated, not vendor-agnostic.* ARCNODE owns the inside of every container we ship and the integration of every BESS we support. Vendor-agnostic templates would force codegen and HMI to generalize against "any device might look like anything" — the wrong abstraction for a product that owns the inside of its containers. Opinionated templates let codegen targets be specific and HMI views be designed against known shapes.

Measurements, commands, units, and protocol bindings live in **device templates** (e.g. `bess_module`, `dlr_sensor`). The DTM references templates by bare slug; it does not redefine measurements per deployment. Templates are PR-gated; canonical home is `edp-api/device_templates/`. Git history is the version source.

**Module types.** Three roots:

- `compute_module` — arcnode-fab (GPU servers, NVLink, DLC pumps, plate heat exchangers)
- `grid_module` — arcnode-fab (AC switchgear, transformer, PCS, metering relays)
- `bess_module` — BYO (Tesla Megapack/Megablock, CATL EnerOne)

For arcnode-fab containers, templates encode hardware design end-to-end. For BESS, templates encode the supported vendor's protocol surface and register map, authored by the ARCNODE team.

**Hierarchy.** Each template declares a `contains:` block listing direct child templates (required and scalable). Containment chains arbitrarily; depth is unbounded. The DTM expresses this as a parent-chain tree: every device has an optional `parent: device_id` pointer. Topic addressing stays flat — `device_id` in the topic is the leaf, not a path; depth lives in the parent chain, walked by HMI/analyst codegen at render time. IEC 61850 alignment carried as optional metadata (`iec_61850.logical_nodes`, `iec_61850_ref` per measurement).

Example tree (BESS-shaped; identical pattern for compute and grid):

```
bess_module_1             template: bess_module,   parent: null
├── bess_rack_1           template: bess_rack,     parent: bess_module_1
│   ├── bess_bms_1        template: bess_bms,      parent: bess_rack_1
│   ├── bess_inverter_1   template: bess_inverter, parent: bess_rack_1
│   └── bess_cell_1..N    template: bess_cell,     parent: bess_rack_1
├── bess_rack_2
└── ...
```

Each level declares its own measurements/commands. Module-level rollups (whole-module SOC), mid-level rollups (per-rack spread), and leaf reads (per-cell voltage) are independent channels, all flat in the topic, related only by `parent` links in the DTM.

**Distribution.** `edp-api` embeds a `templates_used: Record<templateRef, DeviceTemplate>` map in every DTM payload, containing exactly the templates referenced by `devices`. Downstream consumers receive everything they need in one POST.

**Per-deployment customization.** Per-device `extra_measurements:` escape hatch in the DTM. Use sparingly.

User-defined template authoring is out of scope — new templates land via PR to `edp-api/device_templates/`. Device-level instantiation (using an existing template at a new `device_id`) is in scope — see §21.

### §15. Two-layer naming: canonical + display

*Why.* The customer wants to call it "Pile A DC Bus". The code wants `voltage_dc`. A display name override in the DTM keeps both right without breaking topic stability.

Topic, code, and spec use the canonical name (e.g. `voltage_dc`). The DTM carries a `display_name` override per site, device, and measurement, propagated to AsyncAPI as `x-display-name` and rendered by the HMI. Renaming a label is a metadata-only change → patch bump → re-fetch but no resubscribe. Resolution order: DTM override → template default → humanized canonical.

### §16. Identifier format: snake_case slugs, immutable

*Why.* Operator readability — UUIDs make `mosquitto_sub -t '#'` unreadable. Immutability — renaming an ID would invalidate every subscriber.

`^[a-z][a-z0-9_]{0,62}[a-z0-9]$`. Display names are mutable; identifiers are not. Uniqueness: `site_id` unique within deployment, `device_id` unique within site, `measurement` unique within template.

### §17. Spec versioning: semver, diff-free events

*Why.* Consumers always need to re-fetch the spec to be safe. A diff carried in the event would lie or go stale. The version in the event is an audit breadcrumb, not a computation input.

AsyncAPI spec uses semver. Patch = metadata fix; minor = additive (new channel, new device); major = breaking (rename, remove, unit change). `system/topology_changed` carries only `{ts, version}` — consumers re-fetch the full spec and diff client-side. Consumers always re-fetch unconditionally.

### §18. QoS and Retain per family

*Why.* Retain on commands is the dangerous default — explicitly off so a setpoint replay does not unexpectedly spin a BESS.

| Family | QoS | Retain | Why |
|---|---|---|---|
| measurements | 0 | true | High-rate telemetry; new subscriber sees latest immediately |
| commands | 1 | false | Imperative-at-time; no replay on broker restart |
| system | 1 | false | Transient triggers; no fire-on-new-connect |

### §19. AsyncAPI spec written from device perspective

*Why.* Device codegen runs on the most constrained hardware (Rust on ESP32 / Pi). Writing the spec from the device's viewpoint makes their code path literal. HMI and analyst invert during their own codegen — they have CPU to spare.

`action: send` = device publishes; `action: receive` = device subscribes. Topology-update events appear as `action: receive` since devices subscribe to them. Non-device consumers (`hmi`, `analyst-server`, `platform-api`) invert during their own codegen. `family` (slot 4) is orthogonal to `action`: `action` distinguishes publisher from subscriber, `family` distinguishes data direction. MQTT bindings (QoS, retain) are inferred from `family` per §18; per-channel `bindings.mqtt` overrides only for exceptions.

### §20. Basic auth

*Why.* Broker is per-deployment private (per §5). Blast radius is one site. Roles, mTLS, and JWT are real work; do them when the threat model demands.

Single MQTT username/password from `template-secrets.env`. Topic ACLs open. Roles, mTLS, JWT deferred.

### §21. Dynamic device CRUD via `ems-device-api`

*Why.* Operators commission, repair, and replace equipment in production. Forcing every change through a redeploy is operational friction without a safety benefit — the spec-regen and `topology_changed` mechanism already handles consumer reconciliation correctly. Template *authoring* stays PR-gated because adding a new measurement vocabulary item is an engineering change; template *instantiation* is a deployment-state change.

Adding, removing, or updating a device — instantiating an existing template at a new `device_id`, swapping a `connection`, retitling a `display_name`, deleting a device — is a device-api operation, not a redeployment. Each successful mutation:

1. Validates referential integrity (template ref, parent acyclic, `bus.members[]` consistency)
2. Persists a new versioned topology row (audit reconstruction)
3. Regenerates the AsyncAPI spec
4. Bumps semver per §17
5. Publishes `system/topology_changed { ts, version }` exactly once

`POST /topology` (full DTM replacement) is the same regen-and-broadcast applied wholesale. Transactional CRUD groups multiple changes into one broadcast.

**Scope:**

| Operation | Allowed dynamically | Notes |
|---|---|---|
| Add device using existing template | yes | Must reference a template in `templates_used` or bundled set |
| Remove device | yes | Cascades to children when parent; refuses if referenced by `bus.members[].device_id` until bus is updated |
| Update `display_name` | yes | Patch bump |
| Update `connection` (host/port/unit_id) | yes | Patch bump |
| Update `parent` (re-parent) | yes | Validates parent chain acyclic |
| Add/remove `extra_measurements` on a device | yes | Minor bump (additive) |
| Introduce a new template at runtime | no | PR-gated (§14) |
| Modify a template | no | PR-gated (§14) |

`ems-hmi` is read-only against `/topology` and `/asyncapi` — not a CRUD client. CRUD is invoked by `platform-api` (bulk delivery), commissioning workflows, and integrator/operator tooling.

## Boot

### §22. Day-1 DTM via flat-file mount, configured by path

*Why.* `dtm.json` is config, not data: small (<100 KB), immutable per deployment, frozen at delivery. It belongs in the delivery bundle alongside `docker-compose.yaml` and the env files, not behind a runtime fetch. Treating it as a mounted file removes a runtime dependency on object storage at device-api boot, drops the minio container from the air-gapped variant, and removes LocalStack from the dev inner loop.

Day-1 boot uses one mechanism: read a JSON file at the path set via the `BOOT_DTM_PATH` env var. Same code path across cloud, ISO, dev, CI — only how the file lands at that path varies. Env-var contract matches the `DOCUMENT_URL` pattern: defaults to unset/skip, deployments set it explicitly.

**Behavior:**

| `BOOT_DTM_PATH` | Topology table | Action |
|---|---|---|
| set | empty | read + parse + validate + seed |
| set | populated | read + skip seed (don't overwrite operator changes) |
| unset | empty | log + skip (graceful empty start) |
| unset | populated | log + skip |

Read happens unconditionally when path is set — surfaces file-side issues at boot rather than at the next restart.

**Failure modes (all fatal when `BOOT_DTM_PATH` is set):**

| Failure | Detection |
|---|---|
| Env var set + file missing/unreadable | fs |
| Invalid JSON | JSON parser |
| Fails Zod `Dtm` validation | Zod |
| `templates_used` slug unknown to bundled catalog | Same check as `POST /topology` |
| DB write fails | Fatal with restart-loop (Docker restart policy) |

Env var unset = graceful empty start. `POST /topology` and the §21 CRUD endpoints are the canonical mutation surfaces.

**Delivery per context (same code, different stager):**

| Context | How `BOOT_DTM_PATH` is set + file lands |
|---|---|
| Cloud (CFN + EC2 + docker-compose) | Compose service block sets `BOOT_DTM_PATH=/app/dtm.json` and bind-mounts `/opt/arcnode/dtm.json` (staged by UserData via `curl https://arcnode-public/orders/<id>/dtm.json`) read-only |
| On-prem ISO | Same compose contract; ISO bake step writes the file to `/opt/arcnode/dtm.json` |
| Dev (docker-compose) | Same compose contract with a fixture file at `dev-fixtures/dtm.json`; or omit the env var to skip |
| CI / smoke | Env var unset → skip; tests POST DTMs via §21 endpoints |

**Idempotency:** device-api never overwrites a populated topology table from a stale read. To re-seed, clear the table (e.g., a migration step before redeploy) or use §21 CRUD endpoints.

**Operator mutation:** `POST /topology` and the §21 CRUD endpoints write directly to the topology document store. Re-mounting a new `dtm.json` and restarting device-api does not overwrite operator state — boot-read is gated on empty-table.

**Out of scope:** runtime fetch from object storage; bundled default DTM inside the image; env-var inline JSON; Parameter Store / Secrets Manager.

## Seed

### §23. Seed runs as a docker-compose init container

*Why.* EC2 already has every persistence secret in its environment at compose-up time. An init container that exits 0 fits docker-compose's lifecycle — compose blocks dependent services until it returns. Platform-api stays pure (provisioning + handing over secrets); the EMS team owns seed logic. No new Lambda, no new IAM perms.

**Seeded slices** (one time-series variant per deployment):

| Slice | Source | Tool |
|---|---|---|
| Vector | Public S3 corpus | `curl` + `psql` against Aurora pgvector |
| Graph | Public S3 corpus | `curl` + `cypher-shell` (Aura) or Neptune loader API (defense) |
| Time series | Public S3 SQL dump | `curl` + `psql` against Tiger Cloud or Aurora pg_partman |

**Not seeded by §23:**

- **Document** — DTM is loaded by `ems-device-api` boot per §22.
- **AOSS index** — auto-populated by Graphiti during the graph seed step (defense variant only).

**Idempotency:** init container checks if the target slice is non-empty before seeding. Re-running `docker compose up` after a healthy seed is a no-op (ms exit). Clearing a slice (manual operator action) gets re-seeded on the next compose-up.

**Trust boundary:** seed scripts live in the EMS repo and execute inside the customer's EC2 instance. 

## References

- [readme.md](readme.md) — topology, sequence, deployment diagrams
- [AsyncAPI 3.0](https://www.asyncapi.com/docs/reference/specification/v3.0.0)
- [Sparkplug B](https://www.eclipse.org/tahu/spec/sparkplug_spec.pdf) — SCADA vocabulary reference
- [edp-api `manifest_client.py`](../edp-api/src/bom_generator/manifest_client.py) — S3 fetch pattern
