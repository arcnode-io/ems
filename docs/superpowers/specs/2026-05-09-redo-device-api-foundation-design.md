# Redo device-api — Foundation Design

**Status:** Draft
**Date:** 2026-05-09
**Sub-project:** A of {A: Foundation, B: Day-1 boot, C: Dynamic CRUD}
**Depends on:** ADR-002 (revised 2026-05-09 — §7 + §14)

## Context

`ems-device-api` was built for a static-DTM model — POST a blob, regenerate the AsyncAPI spec, broadcast `topology_changed`. The schema and the workflow assumed redeployment was the only mutation path. Two facts make that wrong:

1. **edp-api ↔ device-api DTM drift.** edp-api emits `{ems_mode, modules: list[Module], devices: list[Device]}` with `device_type: str` and UUIDs. device-api expects `{templates_used, devices: Record<id>, buses[], parent chains}` per ADR-002 §7. They can't talk to each other today.
2. **Topology must mutate dynamically.** Per ADR-002 §14 (added 2026-05-09), add/remove/update equipment is a runtime operation in `ems-device-api`, not a redeploy. Templates remain PR-gated; *device-level* CRUD is dynamic.

This sub-project lays the foundation: one canonical DTM schema shared by both services, and the templates layer that ADR-002 §7 specifies but doesn't yet exist in code.

Out of scope here (handled by sub-projects B and C):
- Day-1 boot from `/etc/ems/dtm.json` pointing at fixtures (sub-project B)
- Dynamic CRUD endpoints + spec regen + broadcast (sub-project C)

## Goals

1. One DTM schema. edp-api emits it; device-api receives it. Drift impossible at the boundary because the same schema definition is the source.
2. Templates exist as real artifacts in `edp-api/device_templates/` per ADR-002 §7 — currently zero exist.
3. Templates own protocol bindings (Modbus FC + register, DNP3 addrs, SNMP OIDs, scale/offset) so the AsyncAPI generator can resolve `x-source` without consulting other sources.
4. Module templates are first-class — `compute_module`, `grid_module`, `bess_module` exist as catalog entries with `contains:` blocks and (optionally) rollup measurements.
5. Slug-based device identity per ADR-002 §9. UUIDs go away in the DTM.

## Non-goals

- Multi-vendor support per logical role. One canonical vendor per role for MVP. (Future.)
- Template versioning (`vN`). MVP uses bare slugs; git is the version source. (Future, when fleet has multi-firmware revisions in flight.)
- User-defined templates at runtime. Templates are PR-gated to `edp-api/device_templates/` per ADR-002 §7.
- HMI as a topology editor. Per ADR-002 §14, `ems-hmi` is not a CRUD client — read-only against `/topology` and `/asyncapi`. CRUD endpoints are invoked by `platform-api`, commissioning workflows, and operator tooling outside the HMI surface.

## Design

### Template schema

```yaml
# edp-api/device_templates/leaf/revenue_meter.yaml
template: revenue_meter
kind: leaf                       # leaf | module
equipment_id: GRD-MTR-001        # required for kind=leaf, null for kind=module
vendor: Schneider Electric
model: PowerLogic ION9000
description: Class 0.1S revenue meter, ANSI C12.20 / IEC 62053-22

contains: []                     # leaves are terminal

measurements:
  voltage_a:
    unit: volts                  # ADR-002 §3 enum-locked vocabulary
    type: float
    poll_rate_hz: 1
    display_name_default: "Phase A Voltage"
    iec_61850_ref: MMXU1.PhV.phsA   # optional
    binding:                     # gateway codegen target
      protocol: modbus_tcp
      function_code: 4
      address: 100
      data_type: int16
      word_order: high_low
      scale: 0.1
      offset: 0
  state:
    unit: none
    type: enum
    values: { 1: AUTO, 2: MANUAL, 3: REVENUE_LOCK }
    binding: { protocol: modbus_tcp, function_code: 3, address: 200, data_type: uint16 }

commands:
  reset_counters:
    verb: reset
    target: counters
    unit: none
    payload: trigger
    binding: { protocol: modbus_tcp, function_code: 6, address: 300 }
```

```yaml
# edp-api/device_templates/module/bess_module.yaml
template: bess_module
kind: module
equipment_id: null               # aggregation, not a single piece of equipment
vendor: null
model: null
description: BESS module aggregation

contains:
  - template: bess_rack
    qty: scalable                # or specific integer

measurements:                    # rollups — no binding
  state_of_charge:
    unit: percent
    type: float
    poll_rate_hz: 1
    publisher: line_controller   # gateway skips; line-controller emits

commands:
  set_active_power:
    verb: set
    target: active_power
    unit: watts
    payload: float
    fanout: line_controller      # line-controller fans out to children
```

**Field semantics:**

| Field | Required | Notes |
|---|---|---|
| `template` | yes | logical role slug; ADR §9 format `^[a-z][a-z0-9_]{0,62}[a-z0-9]$` |
| `kind` | yes | `leaf` or `module` |
| `equipment_id` | iff kind=leaf | 1:1 link to `edp-module-assemblies/equipment/<id>/spec.yaml` |
| `vendor`, `model` | iff kind=leaf | metadata |
| `contains[]` | iff kind=module | child template refs; `qty:` is `scalable` (any count ≥1, deployment-resolved) or a positive integer (exact count) |
| `measurements{}.binding` | when the gateway publishes the measurement | present → gateway codegens `x-source` and polls; absent → measurement is derived/rollup and `publisher:` is required |
| `measurements{}.publisher` | when `binding` is absent | enum: `line_controller`, `analyst` |
| `commands{}.binding` | when the gateway dispatches the command | present → gateway emits the protocol write; absent → `fanout:` is required |
| `commands{}.fanout` | when `binding` is absent | enum: `line_controller` |

**Validation rules:**
- Each template file MUST declare at least one of `measurements:` or `commands:`.
- Every `binding.protocol` value MUST match the equipment spec's `control.protocol`.
- Every `unit` MUST be in the ADR-002 §3 enum-locked vocabulary.
- `contains[].template` references MUST resolve at edp-api startup walk.
- Every measurement MUST have either `binding:` or `publisher:` (never both, never neither).
- Every command MUST have either `binding:` or `fanout:` (never both, never neither).

### Canonical DTM

```python
# Shared schema. edp-api emits, device-api receives. One source of truth.

class Connection(BaseModel):
    """Per-device runtime connection params."""
    host: str                                # may be PROVISIONED_AT_COMMISSIONING
    port: ProvisionedInt                     # int | sentinel
    unit_id: str | None = None              # modbus unit_id, dnp3 outstation_addr, etc.

class Device(BaseModel):
    """One device instance — leaf or module per its template's kind."""
    device_id: str                           # snake_case slug per ADR §9
    template: str                            # ref into Dtm.templates_used
    parent: str | None = None               # FK to another Device.device_id
    display_name: str | None = None
    connection: Connection | None = None    # required for gateway-bound leaves
    blocking: list[BlockingKind] = [BlockingKind.LIVE_MODE]
    extra_measurements: dict[str, Measurement] | None = None  # ADR §7 escape hatch

class BusMember(BaseModel):
    device_id: str
    port: str | None = None                 # references device's port id

class Bus(BaseModel):
    bus_id: str
    type: Literal["dc", "ac"]
    members: list[BusMember]

class Dtm(BaseModel):
    deployment_uuid: UUID
    ems_mode: EmsMode = EmsMode.SIM
    sizing_ref: str | None = None           # optional traceback to sizing run
    sizing_params: SizingParams
    devices: dict[str, Device]              # keyed by device_id
    buses: list[Bus]
    templates_used: dict[str, DeviceTemplate]   # keyed by template slug

    @computed_field
    def pending_devices(self) -> list[Device]: ...
```

**Notable shape decisions:**
- `dict[str, Device]` keyed by `device_id` (not `list`) — clean parent/bus FKs.
- `device_id: str` slug per ADR-002 §9. UUIDs go away in the DTM.
- No `dtm_version`, no `generated_at`. DTM is versioned externally by the device-api topology row that holds it, plus AsyncAPI semver.
- Modules from the current edp-api dissolve — `compute_container_1` becomes a Device with `template: compute_module, parent: null`.
- `extra_measurements` is per-Device; per ADR-002 §7.
- Commissioning fields preserved: `ems_mode` at top, per-Device `blocking`, computed `pending_devices`.

### Slug generation rules

Rule: `{template}_{index_within_site}`.

- One revenue meter at a site → `revenue_meter` (no index needed when count=1, but generator always writes the index for symmetry → `revenue_meter_1`).
- Multiple BESS racks → `bess_rack_1`, `bess_rack_2`, `bess_rack_3`.
- Module devices → `compute_module_1`, `grid_module_1`.
- ADR-002 §3 example `meter_01` matches this style.

Index numbering is contiguous, 1-based, deterministic from the deployment resolution. Never reused after a delete (advances monotonically over the device-api's lifetime; this is enforced in sub-project C).

### Bus authoring

`buses:` is a section inside the per-assembly `topology.yaml` (alongside `devices:`). Members are declared by template-pattern, expanded by `dtm_generator` per the deployment resolution.

```yaml
# edp-module-assemblies/assemblies/grid-container/commercial-ac/topology.yaml
devices:
  - device_type: switchgear
    ...
buses:
  - bus_id: ac_main
    type: ac
    members:
      - { device_template: switchgear, port: line }
      - { device_template: revenue_meter, port: voltage_in }
      - { device_template: dnp3_master_external, port: line }
```

Where `qty` is scalable (e.g. BESS racks), the generator adds one bus member per instance.

### Templates — bundling and runtime

- **Authoring:** `edp-api/device_templates/leaf/<name>.yaml` and `edp-api/device_templates/module/<name>.yaml`. PR-gated.
- **Validation:** edp-api walks the directory at startup, validates each file against the template schema, builds an in-process catalog. Validation failures → process exit. Drift surfaces at deploy time, not runtime.
- **Emission:** `dtm_generator` walks the resolved devices, looks up each one's template by slug, copies the verbatim template into `Dtm.templates_used`. The DTM is self-describing per ADR-002 §7; device-api never reads templates from disk.
- **Bundling for device-api dynamic CRUD:** at device-api build time, CI snapshots `edp-api/device_templates/` into the device-api Docker image (e.g., copies into `/app/device_templates/`). At runtime, dynamic CRUD operations that reference a template not present in the current `templates_used` resolve from this bundled catalog. Per ADR-002 §14: templates referenced by CRUD must already exist in the bundled set; runtime template authoring is rejected.

### Referential integrity

Every DTM mutation (POST /topology bulk, or per-device CRUD) MUST validate:

1. **Template ref resolves.** Every `device.template` exists either in `templates_used` (POSTed bulk) or the bundled catalog (dynamic CRUD).
2. **Parent chain acyclic and well-formed.** Every `device.parent` is either `null` (root) or a `device_id` present in `devices`. No cycles. Module-template devices may have null parent; leaf-template devices typically have a parent (module) — but not required.
3. **Bus members resolve.** Every `bus.members[].device_id` exists in `devices`.
4. **Connection presence.** A leaf device whose template has any binding must declare `connection`. Module devices and binding-free leaves may have `connection: null`.
5. **`contains:` consistency.** Module-template devices in the DTM MUST have child devices whose template matches their template's `contains[]` declaration. Quantities respected: `qty: scalable` admits any count ≥ 1; `qty: <int>` admits exactly that count.

Any violation rejects the mutation atomically — DTM-level for POST /topology, per-operation for CRUD.

### What edp-api looks like after migration

- `src/shared/schemas/dtm.py` — replaces current schema with the canonical one above. Drops `Module`, drops `device_uuid` from `Device`. Adds `templates_used`, `buses`, `extra_measurements`.
- `src/shared/schemas/template.py` (new) — Pydantic model for the template schema described above.
- `src/dtm/template_loader.py` (new) — walks `device_templates/`, validates, builds catalog.
- `src/dtm/dtm_generator_service.py` — assigns slugs, dissolves modules, looks up templates, embeds `templates_used`, expands bus member patterns.
- `src/dtm/topology_yaml.py` — `TopologyYaml` gains a `buses:` section; `TopologyDeviceSpec` gains a port catalog (so bus members can reference ports by name).
- 15 template files added under `edp-api/device_templates/leaf/` (one per current `equipment_id`).
- 3 module template files added under `edp-api/device_templates/module/` (compute_module, grid_module, bess_module).

### What ems-device-api looks like after migration

- `src/topology/dtm.schema.ts` — replaces with the canonical schema (Zod mirror of edp-api's Pydantic).
- `src/templates/template.schema.ts` — replaces with the full template schema described above.
- Template catalog loaded from `/app/device_templates/` at startup.
- Existing `topology.controller.ts` (POST/GET /topology) keeps the same shape but rejects DTMs that fail referential integrity.
- AsyncAPI generator (`src/asyncapi/`) consumes templates_used + devices + buses to produce the spec. Out of scope for this sub-project; sub-project C completes that.

## Migration

This breaks three contracts:

1. **edp-api → device-api DTM** (the on-the-wire schema between the two services)
2. **edp-api ↔ edp-module-assemblies `topology.yaml` shape** (the only assemblies-side contract that changes)
3. **edp-api internal Pydantic schema**

No production deployments exist, so no backwards-compat shim. All three break atomically across one PR cycle.

### What does NOT change in edp-module-assemblies

- `manifest.yaml` — URL-based, content-agnostic. Stays as `{version, specs, geometry, assemblies, plates, profiles}` per `src/manifest_models.py`.
- `equipment/<id>/spec.yaml` — equipment hardware datasheets. Templates reference these by `equipment_id` (1:1) but spec.yaml shape is unchanged.
- `assemblies/.../bom.yaml` — `{parts: [{equipment_id, qty}], plates, deployment_context?}` stays. bom_generator unaffected.
- STEP / GLB / envelope artifacts — unaffected.
- `src/assemblies/{compute,grid}_container.py` — emit bom.yaml programmatically; do not emit topology.yaml. Unaffected.

### What DOES change in edp-module-assemblies

Only `assemblies/<type>/<variant>/topology.yaml`:

| Field | Before | After |
|---|---|---|
| `devices[].device_type` | free string (e.g. `switchgear`) | renamed to `devices[].template` (template slug) |
| `devices[].host`, `devices[].port` | top-level | nested under `devices[].connection` |
| `devices[].protocol_config` | full per-device protocol block with register maps | **removed** — register maps live in templates |
| `devices[].connection.unit_id` | (was inside protocol_config.unit_id) | only per-instance addressing remains here |
| `buses` | absent | new top-level section with template-pattern members |

The only existing topology.yamls today are `compute-container/commercial-ac/topology.yaml` and `grid-container/commercial-ac/topology.yaml` (plus the dnp3_master_external entry I just added). Migration touches exactly two files in the assemblies repo. Other grid-container variants (defense-ac, defense-dc-ext, no-bess, defense-no-bess, commercial-dc-ext) don't have topology.yamls today — `dtm_generator` warns + skips them per current behavior.

### Order — three PRs

**PR 1 — edp-api templates foundation** *(standalone, mergeable any time)*
- `src/shared/schemas/template.py` — Pydantic template schema
- `src/dtm/template_loader.py` — walks `device_templates/`, validates at startup, builds in-process catalog
- 18 YAML files: `device_templates/leaf/*.yaml` (15, one per current `equipment_id`) + `device_templates/module/*.yaml` (3: compute_module, grid_module, bess_module)
- Tests for catalog walk, validation rules, lookup
- No downstream consumers yet — dtm_generator is unaware. Safe to land independently.

**PR 2 — edp-api DTM rework ⇄ edp-module-assemblies topology.yaml rewrite** *(lock-step)*

*edp-api PR:*
- Replace `src/shared/schemas/dtm.py` with canonical schema (Connection, Device, BusMember, Bus, Dtm; dict-keyed by slug; `templates_used` embedded)
- Update `src/dtm/topology_yaml.py` — new shape with `template`, `connection`, `buses` section
- Update `src/dtm/dtm_generator_service.py` — assign deterministic slugs, dissolve modules into Devices, look up templates from PR 1's catalog, embed `templates_used`, expand bus member patterns
- Migrate tests

*edp-module-assemblies PR:*
- Rewrite `assemblies/compute-container/commercial-ac/topology.yaml` to new shape (drop `protocol_config`, rename `device_type` → `template`, nest `host`/`port` under `connection`, add `buses:` section)
- Rewrite `assemblies/grid-container/commercial-ac/topology.yaml` likewise
- Both files in one PR
- Other variants (defense-*, no-bess, *-dc-ext) have no topology.yaml today; out of scope

*Coordination:* both PRs merge in the same calendar window. Each repo's main branch is broken against the other otherwise. Given zero production deployments, this is acceptable; no feature flag.

**PR 3 — ems-device-api migration** *(after PR 2)*
- Replace `src/topology/dtm.schema.ts` with canonical Zod (mirror of edp-api Pydantic)
- Replace `src/templates/template.schema.ts` with full template schema
- CI step: snapshot `edp-api/device_templates/` into `/app/device_templates/` at device-api image build
- Wire bundled-template loader at startup
- Validate referential integrity on POST /topology (template ref resolves, parent acyclic, bus members exist, connection presence rule, contains[] consistency)
- Update tests

**End-to-end smoke after PR 3:**
- edp-api generates DTM for commercial-ac
- POST to device-api → expect 201 with all referential-integrity checks passing

### ADR-002 follow-up — landed 2026-05-09

ADR-002 §7 was amended alongside this spec to drop template `vN` versioning for MVP. DTMs reference templates by bare slug; git history is the version source. The `vN` suffix is reserved for future fleet-management work. §14 implications also updated: HMI is read-only and not a CRUD client.

## Testing

- **edp-api template loader:** unit tests for catalog walk, schema validation, catalog lookup, validation-failure-on-bad-template.
- **edp-api DTM generator:** unit tests for slug assignment, module dissolution, templates_used embedding, bus member expansion.
- **edp-api integration:** generate a DTM for commercial-ac, validate against the canonical schema, assert templates_used includes every referenced template.
- **device-api schema:** Zod schema accepts a known-good DTM, rejects every referential-integrity violation.
- **Cross-repo:** snapshot test — emit DTM from edp-api, POST it to device-api, expect 201.

## Open Questions

None at design time. All key decisions resolved during brainstorming:
- Template ↔ equipment relationship: template references equipment_id, owns bindings.
- Template naming: logical role slug, no `vN` for MVP.
- Multi-vendor: out of scope for MVP.
- Module templates: first-class with `contains:` and rollup measurements.
- Slug rule: `{template}_{index_within_site}`.
- Bus authoring: `buses:` section in `topology.yaml`.
- Templates_used: embedded in DTM per ADR §7.
- Bundling: CI snapshot of `edp-api/device_templates/` into device-api image at build time.
- Versioning: AsyncAPI semver and topology row history kept; DTM payload self-version dropped; template `vN` dropped for MVP.

If new questions surface during writing-plans, fold them back here.
