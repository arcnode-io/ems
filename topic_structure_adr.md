# ADR-002: MQTT Topic Structure and Payload Conventions

**Status:** Accepted (revised 2026-05-09 — see §14)
**Date:** 2026-04-25
**Decision Makers:** Development Team
**Consulted:** Energy Domain SMEs, Customer Operators
**Informed:** EMS Subsystem Maintainers, Course Participants

## Revision History
- 2026-04-25 — Initial acceptance.
- 2026-05-09 — §7 scope clarified; §14 added. Dynamic device-level CRUD via
  `ems-device-api` is now in scope. Template *authoring* remains PR-gated and
  out of runtime scope. Earlier "topology changes only via redeployment" stance
  is superseded.
- 2026-05-09 — §7 template versioning (`vN`) dropped for MVP. Templates are
  PR-gated; git history is the version source. `vN` reserved for future
  fleet-management work. §14 implications updated: HMI is read-only and not
  a CRUD client.

## Context

ADR-001 commits to MQTT as the central event bus. That decision did not specify topic structure, payload format, unit handling, or naming conventions. Without a concrete contract, every subsystem (gateway, line-controller, HMI, analyst) would invent its own — and existing draft readmes already showed three incompatible topic shapes (`sites/.../measurements/{unit}/{value}` in gateway, `sites/.../calculations/...` in PST, `arcnode/{deployment_uuid}/...` in device-api).

The contract must satisfy:

1. **Customer-owned naming** — operators may run multiple sites and don't want a vendor prefix in their topics
2. **Stable topics under firmware/spec evolution** — adding a measurement should not invalidate existing subscribers
3. **Operator debuggability** — `mosquitto_sub -t '#'` at 3am should be readable
4. **Industrial gear realities** — Modbus/DNP3/SNMP carry raw integers with vendor-specific scaling; raw values must not leak onto the wire
5. **MVP-scoped vocabulary** — defer optional fields until scoped code needs them
6. **Codegen drives consumers** — gateway, line-controller, HMI all generate types from a single AsyncAPI v3 spec served by `ems-device-api`

## Decision

A two-family topic model with fixed-depth segments per family, strict `{ts, value}` payloads, and unit-encoded topic slots backed by an enum-locked vocabulary in the AsyncAPI spec. Device-class templates own the canonical vocabulary; the per-deployment DTM overlays customer-facing display names.

### Key Architectural Decisions

#### 1. Two Families: `measurements` and `commands`

**Decision**: Every application-level message belongs to exactly one of two families. `measurements/` carries everything a device emits (sensor readings, computed values, device health, alarm state, command acks). `commands/` carries everything sent to a device.

**Rationale**:
- Mirrors Modbus read/write and DNP3 master/outstation flow
- Removes the gray zone between "measurement," "state," "status," and "calculation" — they are all things the device emits, so they are all measurements
- Two parsers, two type-safe handlers downstream

**Implications**:
- Boolean and enum readings (rain, tap_position, alarm_state) are measurements with `unit=none`, not a separate family
- Derived quantities (DLR dynamic rating) are measurements — the consumer doesn't care that the publisher computed them
- A third top-level prefix `system/` carries broker-internal events (topology changed, etc.); not a third family

#### 2. Fixed-Depth Topics with Per-Family Parser

**Decision**: Measurements are 6 segments. Commands are 7 segments. Different depths, different parsers, no shared overload.

```
sites/{site_id}/devices/{device_id}/measurements/{measurement}/{unit}        # 6
sites/{site_id}/devices/{device_id}/commands/{verb}/{target}/{unit}          # 7
system/{event_type}                                                          # 2
```

**Rationale**:
- Variable-depth wildcards become parsing nightmares; fixed depth makes broker-level subscriptions predictable
- Splitting commands as `verb/target/unit` gives broker-level ACL hooks (e.g. allow `set/+/+`, deny `reset/+/+`) without payload introspection
- The parser switches on slot 4 (`family ∈ {measurements, commands}`) and dispatches

**Implications**:
- Unitless measurements use a literal `none` sentinel in the unit slot (e.g. `measurements/rain/none`)
- `none` chosen over `null` to avoid JSON-coercion accidents in deserializers
- Verb is enum-locked: `[set, reset, clear, start, stop, enable, disable]`

#### 3. Unit Vocabulary in Topic, Locked by Enum in Spec

**Decision**: Slot 6 of measurements (and of commands when applicable) carries the engineering unit. The AsyncAPI parameter schema locks the enum.

**Rationale**:
- Operator can `mosquitto_sub` and read units inline
- Broker-level filters work directly: `+/+/measurements/+/celsius`
- Enum-locked vocabulary prevents `degC` vs `celsius` divergence — typo at codegen time, build fails

**MVP unit vocabulary** (extend by AsyncAPI spec change → patch-level version bump):

```
volts | amps | watts | vars | voltamperes | watt_hours | hertz |
celsius | percent | watts_per_m2 | meters_per_second |
bar | liters_per_minute | none
```

**Naming rules**: lowercase, plural where natural (`volts` not `volt`), short compounds (`watts_per_m2` not `watts_per_meter_squared`), `vars` for reactive power (`var` collides with reserved words).

**Implications**:
- Adding a unit is a spec change → `system/topology_changed` → consumers re-fetch — not free, but additive
- SI-prefixed duplicates rejected: `watt_hours` only, no `kilowatt_hours`. JSON float64 has range to spare.

#### 4. Gateway Converts Raw → Engineering Units at Boundary

**Decision**: Industrial protocols (Modbus, DNP3, SNMP, CAN) carry raw integers with vendor-specific scaling. The gateway applies scale and offset; MQTT carries the engineering value. Scaling metadata lives in the AsyncAPI spec under `x-source` per channel for audit.

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

**Rationale**:
- Industry standard: gateway is the unit-translation boundary
- Downstream consumers (HMI, analyst, line-controller) stay simple — no scaling logic replicated three times
- Spec records the math, answering "where did this number come from" for regulatory audit

**Implications**:
- Drift between gateway code and spec is the only failure mode — mitigated because gateway code is itself codegenned from the spec

#### 5. Strict `{ts, value}` Payload, Quality via Time-Joined Measurement

**Decision**: Every sample payload is `{ts, value}`. No quality, no source flags, no correlation IDs. Quality information lives in a separate `status` measurement, joined by timestamp at query time.

**Rationale**:
- Wire stays minimal; payload doesn't repeat metadata that doesn't change per sample
- Quality is recoverable post-hoc via `LATERAL` join of status ≤ sample.ts — same pattern PI/Ignition use internally
- MQTT LWT handles publisher-level death; per-device `status` measurement handles sub-device degradation

**Implications**:
- Industrial-historian egress (PI, Ignition) needs an adapter that does the time-join. Acceptable trade for wire minimalism.
- `ts` is RFC3339/ISO8601 string with `Z` suffix — TimescaleDB / TigerData ingest natively into `TIMESTAMPTZ` with no conversion code

#### 6. Sample Schemas Aligned with Family Semantics

**Decision**: Four sample schemas in the spec.

```
FloatSample    { ts, value: number }
BooleanSample  { ts, value: boolean }
EnumSample     { ts, value: string }   # value constrained to enum:[...] in the channel schema
TriggerSample  { ts }                  # commands only — reset, clear, fire-and-forget
```

The `EnumSample` channel schema in the generated AsyncAPI spec sets `value.enum` to the ordered list of label keys from the class `values:` block (e.g. `["AUTO", "MANUAL", "RUNPQ"]`). The gateway translates raw register integers to the matching label before publish; the HMI renders the label it receives — no int-to-string logic in any consumer.

**Rationale**: Symmetric across measurements and commands where shape matches; trigger is command-only (a `reset` has no payload value).

**Verb → schema mapping**:

| Verb | Schema |
|---|---|
| set | Float / Boolean / Enum (depending on target) |
| start, stop, enable, disable | Boolean (persistent state) |
| reset, clear | Trigger |

#### 7. Device Templates as Canonical Vocabulary Source

**Decision**: Measurements, commands, units, and protocol bindings for each device type live in a **device template** (e.g. `bess_module`, `dlr_sensor`). The DTM references templates by bare slug; it does not redefine measurements per deployment. Template authoring is PR-gated; git history is the version source. A `vN` suffix is reserved for future fleet-management work where multiple firmware revisions of the same model are in flight simultaneously — out of scope for MVP.

We use *template* rather than *class* deliberately — "class" is OOP jargon that doesn't translate to industrial operators or integrators, while *template* is the standard SCADA/HMI term for "a definition you instantiate." OPC UA's *ObjectType*, IEC 61850's *Logical Node Type*, BACnet's *ObjectType*, and Sparkplug's *Device Definition* all model the same concept; we picked the term that reads cleanly in operator-facing UI, ADR text, and code without overloading either *type* or *profile* (the latter collides with IEC 61850 conformance profiles).

**Positioning — ARCNODE owns the inside of every container we ship and the integration of every BESS we support.** The three module types are `compute_module` and `grid_module` (arcnode-fabricated containers — GPU servers, NVLink switches, DLC pumps + plate heat exchangers; AC switchgear, transformer, PCS, metering relays) and `bess_module` (BYO — Tesla Megapack, Tesla Megablock, CATL EnerOne). Templates are not vendor-agnostic placeholders. For arcnode-fab containers, templates encode our hardware design end-to-end. For BESS modules, templates encode the supported vendor's protocol surface and register map, authored by the ARCNODE team. Templates are first-class, opinionated definitions, PR-gated like source code. Software architecture leans on this — codegen targets are specific (we know exactly what a `bess_module` exposes), HMI views are designed against known shapes (not generic "render whatever the device says"), and template authoring discipline is rigorous (every PR is a real hardware decision). User-defined *template authoring* is explicitly **out of scope** for MVP — new templates land via PR to `edp-api/device_templates/`, not at runtime. Per-template extensibility within an existing template lives at the per-device `extra_measurements:` escape hatch in the DTM. *Device-level* dynamic CRUD — instantiating an existing template at a new device-id, removing a device, updating a device's display name or connection — is in scope and lives in `ems-device-api`; see §14.

**Rationale**:
- Bikeshedding ("`voltage_dc` here, `dc_voltage` there") resolved once at template-design time
- Aligns with the modular hardware product surface — three module-level templates as roots (`compute_module`, `grid_module`, `bess_module`), equipment templates beneath, all curated to real datasheets
- Git history is the version source for MVP; template-file PRs carry the audit trail. Explicit `vN` adds discipline cost without buying anything until multiple firmware revisions of the same model are in flight simultaneously.

**Implications**:
- Templates are the single PR-gated vocabulary surface. Canonical home is **`edp-api/device_templates/`** — `edp-api` is both author and sole runtime reader, no separate distribution repo. Promote to a dedicated repo when template count exceeds ~12 or external integrators need consumption-only access.
- `edp-api` walks its `device_templates/` at startup, validates every YAML, and uses the templates for sizing.
- DTMs are **self-describing**: `edp-api` embeds a `templates_used: Record<templateRef, DeviceTemplate>` map in every DTM payload it emits, containing exactly the templates referenced by the `devices` block. Downstream consumers (`ems-device-api`, `ems-hmi`, analyst, gateway) receive everything they need in one POST — no separate template-catalog endpoint, no separate fetch.
- Per-deployment customizations of canonical measurements use a per-device `extra_measurements:` escape hatch in the DTM. Use sparingly — adding to a template is preferred over a one-off override.

**Template hierarchy mirrors real hardware:** the three module types from the product surface (`compute_module`, `grid_module` — arcnode-fab 10ft ISO containers; `bess_module` — BYO third-party BESS) are module-level templates — they sit at the root of the device tree. Equipment within each module is represented as descendant devices at any depth. Real industrial hierarchies are typically 3+ levels deep — a battery module contains racks which contain BMS / inverter / cells; a compute module contains chassis which contain servers which contain GPUs; a grid module contains switchgear which contains breakers and metering CTs. Each template declares a `contains:` block listing its direct child templates (required and scalable). Containment chains arbitrarily — depth is unbounded. The DTM expresses this as a parent-chain tree: every device has an optional `parent: device_id` pointer. Topic addressing stays flat — `device_id` in the topic is the leaf identifier, not a path; depth lives in the parent chain, walked by HMI/analyst codegen at render time. IEC 61850 alignment is carried as optional metadata (`iec_61850.logical_nodes`, `iec_61850_ref` per measurement) for the MVP — strict structural superset is a v1.x goal once an SCD import path lands.

Example tree (BESS-shaped, identical pattern for compute and grid):

```
bess_module_1             template: bess_module,   parent: null
├── bess_rack_1           template: bess_rack,     parent: bess_module_1
│   ├── bess_bms_1        template: bess_bms,      parent: bess_rack_1
│   ├── bess_inverter_1   template: bess_inverter, parent: bess_rack_1
│   └── bess_cell_1..N    template: bess_cell,     parent: bess_rack_1
├── bess_rack_2
└── ...
```

Each level can declare its own measurements/commands. Module-level rollups (whole-module SOC), mid-level rollups (per-rack spread), and leaf reads (per-cell voltage) are independent channels, all flat in the topic, related only by `parent` links in the DTM.

#### 8. Two-Layer Naming: Canonical (Stable) + Display (Customer-Owned)

**Decision**: Topic, code, and spec use the canonical name (e.g. `voltage_dc`). The DTM carries a `display_name` override per site, device, and measurement, propagated to the AsyncAPI spec as `x-display-name` and rendered by the HMI.

**Rationale**:
- Customer can call it "Pile A DC Bus" without breaking topic stability
- Renaming a label is a metadata-only change → patch-level spec bump → re-fetch but no resubscribe
- Codegen still produces deterministic Rust struct fields and TypeScript constants from canonical names

**Resolution order**: DTM override → class default → humanized canonical.

#### 9. Identifier Format: Snake-Case Slugs, Immutable Post-Creation

**Decision**: All identifiers (`site_id`, `device_id`, `measurement`, command `target`) are snake_case slugs matching `^[a-z][a-z0-9_]{0,62}[a-z0-9]$`. They are immutable once a device is provisioned. Display names are mutable, identifiers are not.

**Rationale**: UUID identifiers destroy operator readability in topics; renaming would invalidate every subscriber.

**Uniqueness scope**: `site_id` unique within deployment, `device_id` unique within site, `measurement` unique within device class.

#### 10. Spec Versioning: Semver with Diff-Free Topology Events

**Decision**: AsyncAPI spec uses semver. Patch = metadata fix; minor = additive (new channel, new device); major = breaking (rename, remove, unit change). `system/topology_changed` carries only `{ts, version}` — consumers re-fetch the full spec and compute the diff client-side.

**Rationale**:
- Spec is source of truth — diff computation in the event payload could lie or get stale
- Consumers always re-fetch unconditionally; version is primarily an audit trail
- Operator change-management benefits from operator-readable version strings (`2.4.7 → 3.0.0` reads as "expect breakage")

#### 11. QoS, Retain, and LWT Per Family

**Decision**:

| Family | QoS | Retain | Why |
|---|---|---|---|
| measurements | 0 | true | High-rate telemetry; new subscriber sees latest immediately |
| commands | 1 | **false** | Imperative-at-time; replay on broker restart is a safety hazard |
| system | 1 | false | Transient triggers; old events should not fire on new connect |
| status (LWT) | 1 | true | Liveness state must persist for the device offline-flag pattern |

**Rationale**: Industrial-conventional. Retain on commands is the dangerous default — explicitly off so a setpoint replay does not unexpectedly spin a BESS.

#### 12. AsyncAPI Spec Written from Device Perspective

**Decision**: The AsyncAPI v3 spec served by `ems-device-api` is written from the perspective of a deployed device (or device-like publisher: gateway, line-controller). `action: send` means the device publishes; `action: receive` means the device subscribes. Topology-update events on `system/topology_changed` appear as `action: receive` since devices subscribe to them.

Consumers that are not devices — `ems-hmi`, `ems-analyst-server`, `platform-api` — invert this during their own codegen pass: what the device sends, the consumer subscribes to; what the device receives, the consumer publishes.

**Rationale**: device codegen is the most performance-sensitive path (Rust on ESP32 / Pi, embedded constraints) and benefits from a literal mapping. Inverting in TypeScript / Python codegen for HMI / analyst is cheap.

**Implications**: `family` (slot 4 of every device-scoped topic) carries the family classification (`measurements` | `commands` | `system`) — orthogonal to the `action` axis. Both are needed: `action` distinguishes publisher from subscriber; `family` distinguishes data direction (device-emitted vs broker-imperative vs broker-internal). MQTT bindings (QoS, retain) are inferred from `family` per §11; per-channel `bindings.mqtt` blocks override only for exceptions.

#### 13. Basic Auth for MVP, Role-Based Deferred

**Decision**: MVP uses a single MQTT username/password from `template-secrets.env`. Topic ACLs are open. Roles, mTLS, and JWT auth are deferred to v2.

**Rationale**: Broker is per-deployment private (per ADR-001 — CFN or air-gapped ISO), bounding blast radius. Role-based separation can be added without changing topic structure or payload format.

**Implications**:
- A compromised device could publish fake telemetry from another device. Acceptable for MVP private-broker deployments. Not acceptable once a multi-tenant or partner-integration scenario emerges — at which point ADR-003 supersedes section 13.

#### 14. Dynamic Device CRUD via `ems-device-api`

**Decision**: `ems-device-api` exposes runtime device CRUD endpoints. Adding, removing, or updating a device — instantiating an existing template at a new `device_id`, swapping a `connection`, retitling a `display_name`, or deleting a device — is a device-api operation, not a redeployment. Each successful mutation regenerates the AsyncAPI v3 spec, bumps semver per §10, persists the new spec, and publishes `system/topology_changed { ts, version }`. Bulk delivery of a fresh DTM via `POST /topology` (platform-api on first install or major upgrade) remains supported and is the same regenerate-and-broadcast operation applied to a wholesale replacement.

**Rationale**:
- Operators commission, repair, and replace equipment at runtime. Forcing every change through a redeploy cycle adds operational friction without a safety benefit, since the spec-regen + `topology_changed` mechanism (§10) already handles consumer reconciliation correctly.
- Template *authoring* is the part that needs PR-gating (§7) — adding a new measurement vocabulary item is an engineering change. Template *instantiation* (using a template that already exists) is a deployment-state change and belongs in the runtime surface.
- The two operations (POST /topology blob and per-device PUT/DELETE/POST) collapse to the same downstream effect: persist the new graph, regenerate the spec, broadcast. Symmetry simplifies consumers — they only react to `topology_changed`.

**Scope of dynamic operations**:

| Operation | Allowed dynamically | Notes |
|---|---|---|
| Add device using existing template | yes | Must reference a template already present in `templates_used` or in the bundled template set |
| Remove device | yes | Cascades to children when the device is a parent (§7); refuses if the device is referenced by a `bus.members[].device_id` until the bus is updated |
| Update `display_name` | yes | Patch-level spec bump (§10) |
| Update `connection` (host/port/unit_id) | yes | Patch-level — bindings live in `x-source` per §4; spec regenerates with new metadata |
| Update `parent` (re-parent) | yes | Validates parent-chain has no cycles |
| Add/remove `extra_measurements` on a device | yes | Minor spec bump (additive measurement on that device's channels) |
| Introduce a new template at runtime | **no** | Template authoring is PR-gated (§7); runtime CRUD references templates by bare slug and rejects unknown refs |
| Modify a template (add a measurement, change unit) | **no** | Same — template edits land via PR to `edp-api/device_templates/` |

**Implications**:
- `ems-device-api` owns referential integrity at runtime: every CRUD validates template ref, parent chain (acyclic), and `bus.members[]` consistency.
- AsyncAPI semver discipline (§10) applies per CRUD operation. A single PUT can be patch (rename) or minor (add device); the regen pipeline computes the bump from the diff.
- `system/topology_changed` is fired exactly once per successful mutation; transactional CRUD groups multiple changes into one broadcast.
- **`ems-hmi` is not a CRUD client.** It remains read-only against `/topology` and `/asyncapi` for the module browser, SLD rendering, and live telemetry. It does not call POST/PUT/DELETE on devices. Dynamic CRUD endpoints are invoked by `platform-api` (bulk delivery), commissioning workflows, and integrator/operator tooling outside the HMI surface. Authorization model belongs in a future ADR-004.
- Persistence layer must retain history: every CRUD produces a new versioned topology row so the AsyncAPI version monotonically increases and audit reconstruction is possible.

## Consequences

### Positive
- Single contract end-to-end — gateway, line-controller, HMI, analyst all generate from one AsyncAPI spec
- Topic strings stay stable across firmware iteration, customer renames, and unit-vocabulary extensions
- Operator debuggability matches industrial-SCADA expectation
- Two-family model collapses three previously incompatible draft conventions into one

### Negative
- Industrial-historian egress (PI, Ignition) needs a quality-stitching adapter
- Adding a unit is not free — spec change → `topology_changed` → consumer re-fetch
- Class-template governance becomes a real engineering surface (PR review, versioning discipline)

### Risks
- Drift between gateway code and `x-source` binding metadata in spec — mitigated by gateway-codegen-from-spec
- Class-vocabulary bikeshedding moves from per-PR to per-class-design — same problem, different cadence
- MVP basic auth is a known short-term gap; ADR-003 owes the role-based design

## Alternatives Considered

### Single deeply-nested topic with `arcnode/{deployment_uuid}/...` prefix
- **Rejected**: customer-owned topic namespace beats vendor prefix; per-deployment broker means `deployment_uuid` is redundant on the wire

### Unit metadata only in spec (`x-unit`), topic carries no unit
- **Rejected**: operator debuggability lost; SCADA convention favors unit-in-topic; broker-level filtering by unit becomes consumer-side after spec lookup

### Per-sample quality field in payload (OPC UA pattern)
- **Rejected**: quality is recoverable via time-joined `status`; wire bytes saved on every message

### Variable-depth topics (different shapes per measurement family)
- **Rejected**: parser nightmare for wildcard subscriptions; broker ACL hooks become payload-dependent

### Free-form measurement vocabulary, recorded in spec
- **Rejected**: bikeshedding multiplies; no single canonical source for downstream codegen

## Review

This ADR should be reviewed:
- **When the unit vocabulary needs extension** — add via patch-level spec bump, document the new unit
- **When a new device class is proposed** — class-template governance kicks in, not this ADR
- **When auth scope changes** — new tenant model, partner integration, or compliance requirement triggers ADR-003 supersession of section 13
- **When industrial-historian egress is implemented** — verify the time-join quality pattern holds in practice

## References

- [ADR-001: System Architecture](system_adr.md)
- [AsyncAPI 3.0 Specification](https://www.asyncapi.com/docs/reference/specification/v3.0.0)
- [TimescaleDB Best Practices for Time Columns](https://docs.timescale.com/use-timescale/latest/schema-management/about-hypertables/)
- [Sparkplug B Specification](https://www.eclipse.org/tahu/spec/sparkplug_spec.pdf) — referenced for SCADA-vocabulary alignment
