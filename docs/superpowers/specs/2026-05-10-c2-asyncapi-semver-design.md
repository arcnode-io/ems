# Sub-Project C2 — AsyncAPI Semver Tracking Design

**Status:** Draft
**Date:** 2026-05-10
**Sub-project:** C2 of {C1: dropped (full-load pattern), C2: semver tracking, C3: MQTT broadcast}
**Depends on:** Sub-project A foundation (PRs 1–3 shipped), sub-project B day-1 boot (PR 4 shipped)

## Context

Sub-project A landed canonical Dtm + templates + catalog validation. Sub-project B added boot-time DTM seeding from S3. The AsyncAPI generator at `ems-device-api/src/asyncapi/` already builds a spec from the latest persisted DTM and serves it at `GET /asyncapi`. What's missing: the spec carries a **fixed `info.version`** (currently hardcoded to `1.0.0`) regardless of how many times the DTM has been mutated.

Per [ADR-002 §10](../../../topic_structure_adr.md), the AsyncAPI spec uses semver:

> "Patch = metadata fix; minor = additive (new channel, new device); major = breaking (rename, remove, unit change). `system/topology_changed` carries only `{ts, version}` — consumers re-fetch the full spec and compute the diff client-side."

> "Operator change-management benefits from operator-readable version strings (`2.4.7 → 3.0.0` reads as 'expect breakage')."

This sub-project lands the version-bumping pipeline:

1. **Diff function** — given prev DTM + next DTM, classify the change as `major | minor | patch | none`.
2. **Per-row version persistence** — extend the Topology TypeORM entity with a `version: string` column.
3. **Version computation on save** — `TopologyService.save` looks up the latest row, runs diff, computes new version, persists.
4. **Spec emits the version** — `spec-generator.ts` reads from the topology row instead of hardcoding `"1.0.0"`.

C1 was reframed during brainstorming. The original "per-device CRUD endpoints" was deferred to a future sub-project; clients use `GET /topology` (full-load) for navigation. The remaining C1 cleanup folds into this PR: update `ems-device-api/readme.md` to drop stale references to `/devices-0/:id/devices-1/:id` (which never existed in code post-foundation).

C3 (MQTT broadcast of `topology_changed { ts, version }`) builds on C2's version field.

## Goals

1. `Topology` entity gains `version: string` (semver, e.g., `2.4.7`). Default `1.0.0` for the first row.
2. `TopologyService.save` runs the diff against the latest existing row and computes the new version per the diff rules table below.
3. AsyncAPI generator emits the actual version in `info.version`.
4. Identical-DTM writes are still persisted (audit lineage) but the version is unchanged.
5. Diff function is pure, testable, and returns a structured reason.
6. Existing endpoints unchanged in behavior except `GET /asyncapi` now reflects real version.
7. Stale readme references to `/devices-0/...` removed.

## Non-goals

- Per-device CRUD endpoints (deferred — sub-project's original C1 scope; revisit when device count or operator pain warrants).
- MQTT broadcast (sub-project C3).
- Audit-log queries — Postgres rows are already append-only; query the DB directly if needed.
- Template-body diffing across catalog versions (templates are PR-gated per ADR-002 §7; if the bundled catalog changes, that's a deploy-time concern, not a runtime diff).

## Design

### Diff rules

Pure function `diffDtm(prev: Dtm | null, next: Dtm): DtmDiff` where:

```typescript
interface DtmDiff {
  bump: "major" | "minor" | "patch" | "none";
  reasons: string[];   // human-readable, e.g. "device added: bess_cell_42"
}
```

Rule table — applied in order; the highest-severity match across all changes wins:

| Change in `next` vs `prev` | Bump |
|---|---|
| `prev` is null (first DTM) | `none` (seed at 1.0.0) |
| `next` deep-equals `prev` | `none` |
| Device added | minor |
| Device removed | **major** |
| Device.template changed | **major** (binding shape changes) |
| Device.parent changed | minor (subtree shifts; channels still valid) |
| Device.display_name changed | patch |
| Device.connection changed (host/port/unit_id) | patch (`x-source` binding metadata only) |
| Device.blocking changed | patch (commissioning state) |
| Device.extra_measurements added | minor |
| Device.extra_measurements removed | **major** |
| Bus added | minor |
| Bus removed | **major** |
| Bus.members added | minor |
| Bus.members removed | **major** |
| Bus.type changed (dc ↔ ac) | **major** |
| `templates_used[slug]` body changed | **major** (codegen contract changes) |
| `templates_used[slug]` added | minor |
| `templates_used[slug]` removed | **major** |
| `sizing_params` changed | patch |
| `ems_mode` changed (sim ↔ live) | patch |
| `sizing_ref` changed | patch |
| `deployment_uuid` changed | **major** (different deployment entirely) |

**Severity hierarchy:** `major` > `minor` > `patch` > `none`. If multiple changes happen in one write, the result is the highest severity. Reasons accumulate so an operator can read all the changes.

**Examples of operator-visible diff output:**

```
{ bump: "minor", reasons: ["device added: bess_cell_42"] }
{ bump: "major", reasons: ["device removed: bess_rack_2", "bus member removed: ac_main/bess_rack_2"] }
{ bump: "patch", reasons: ["device display_name changed: bess_cell_1"] }
{ bump: "none", reasons: [] }
```

### Version computation

```typescript
function nextVersion(prev: string | null, diff: DtmDiff): string {
  if (prev === null) return "1.0.0";   // bootstrap seed
  if (diff.bump === "none") return prev;
  const [major, minor, patch] = prev.split(".").map(Number);
  if (diff.bump === "major") return `${major + 1}.0.0`;
  if (diff.bump === "minor") return `${major}.${minor + 1}.0`;
  return `${major}.${minor}.${patch + 1}`;
}
```

Pure, deterministic, testable.

### Persistence

The `Topology` TypeORM entity gains:

```typescript
@Column({ type: "varchar", length: 32 })
version!: string;
```

Migration is automatic via `synchronize: true` (per existing `app.module.ts`). The new column defaults to `1.0.0` for any pre-existing rows; in practice no production rows exist yet.

### `TopologyService.save` flow

```
async save(dtm: DtmType): Promise<Topology> {
  const prev = await this.repo.findOne({ where: {}, order: { receivedAt: "DESC" } });
  const prevDtm = prev?.dtm ?? null;
  const prevVersion = prev?.version ?? null;
  const diff = diffDtm(prevDtm as DtmType | null, dtm);
  const version = nextVersion(prevVersion, diff);
  const row = this.repo.create({ dtm, version });
  return await this.repo.save(row);
}
```

The service log line for each save includes the bump + reasons:

```
INFO seeded topology v1.0.0 (initial)
INFO updated topology v1.0.0 → v1.1.0 (minor: device added: bess_cell_42)
INFO updated topology v1.1.0 → v1.1.0 (none: identical DTM)
INFO updated topology v1.1.0 → v2.0.0 (major: device removed: bess_rack_2; bus member removed: ac_main/bess_rack_2)
```

### AsyncAPI generator integration

Currently `spec-generator.ts` hardcodes `info.version: "1.0.0"`. Wire it through:

```typescript
// spec-generator.ts
export function buildSpec(dtm: Dtm, version: string): AsyncApiSpec {
  return {
    asyncapi: "3.0.0",
    info: {
      title: "ARCNODE EMS Device API",
      version,           // ← was hardcoded "1.0.0"
      description: "..."
    },
    // ...
  };
}
```

`AsyncapiService.generateSpec` fetches both DTM and version from the same row:

```typescript
async generateSpec(): Promise<Record<string, unknown> | null> {
  const row = await this.topology.getLatestRow();    // new method, returns Topology|null
  if (!row) return null;
  return buildSpec(row.dtm as Dtm, row.version) as unknown as Record<string, unknown>;
}
```

`TopologyService` gains:

```typescript
async getLatestRow(): Promise<Topology | null> {
  return this.repo.findOne({ where: {}, order: { receivedAt: "DESC" } });
}
```

`getLatest` (returning DTM only) stays for backwards compat; the new method exposes the row including version.

### Readme cleanup

`ems-device-api/readme.md` currently mentions `/devices-0/:device0Id` and `/devices-0/:device0Id/devices-1/:device1Id` endpoints. Code never had them post-foundation. Delete the section. Add a brief note:

> "**Device navigation.** Clients fetch the full DTM via `GET /topology` and walk `device.parent` chains client-side. See sub-project C1 design for rationale (full-load pattern; per-device endpoints deferred until device counts cross 1k)."

### File structure

**Create:**
- `src/topology/dtm_diff.ts` — pure `diffDtm` + `nextVersion` functions. ≤ 200 lines.
- `src/topology/dtm_diff.test.ts` — unit tests for every rule in the table above.

**Modify:**
- `src/topology/topology.entity.ts` — add `version` column.
- `src/topology/topology.service.ts` — wire `diffDtm` + `nextVersion` into `save`. Add `getLatestRow`.
- `src/topology/topology.service.test.ts` — extend with version-bump tests.
- `src/asyncapi/spec-generator.ts` — accept `version` param.
- `src/asyncapi/asyncapi.service.ts` — fetch version + DTM together.
- `tests/topology.test.ts` — extend integration to assert version is in `info.version` and bumps correctly across multiple POSTs.
- `readme.md` — drop stale `/devices-0` text, add full-load note.

## Test plan

### Unit (src/topology/dtm_diff.test.ts)

One test per row in the diff rules table. Each constructs a minimal prev/next pair that triggers exactly that rule. Examples:

```typescript
it("device added → minor", () => {
  // Arrange
  const prev = baseDtm();
  const next = { ...prev, devices: { ...prev.devices, new_dev: makeDevice("new_dev") } };
  // Act
  const diff = diffDtm(prev, next);
  // Assert
  assert.equal(diff.bump, "minor");
  assert.ok(diff.reasons.some((r) => r.includes("device added: new_dev")));
});

it("device removed → major", () => {
  const prev = baseDtm();
  const next = { ...prev, devices: { revenue_meter_1: prev.devices.revenue_meter_1 } };
  const diff = diffDtm(prev, next);
  assert.equal(diff.bump, "major");
});

// ... one per rule
```

Plus severity-hierarchy test:

```typescript
it("multiple changes — highest severity wins", () => {
  // device added (minor) + device removed (major) → major
  const diff = diffDtm(prev, withAddAndRemove);
  assert.equal(diff.bump, "major");
  assert.equal(diff.reasons.length, 2);
});
```

Plus `nextVersion` tests for bootstrap, none, patch, minor, major.

### Unit (src/topology/topology.service.test.ts)

- First save → row.version === "1.0.0"
- Second save with no change → row.version stays "1.0.0"
- Second save with patch-only change → "1.0.1"
- Second save with new device → "1.1.0"
- Second save with removed device → "2.0.0"
- Service logs the bump reason (mock logger, assert log content)

### Integration (tests/topology.test.ts)

Extend the existing topology integration test:

```typescript
test("POST /topology bumps version on each change; GET /asyncapi reflects", async () => {
  // POST initial DTM
  await client.post("/topology", { dtm: BASE_DTM });
  let spec = await client.get("/asyncapi");
  assert.equal(spec.info.version, "1.0.0");

  // POST same DTM (no-op)
  await client.post("/topology", { dtm: BASE_DTM });
  spec = await client.get("/asyncapi");
  assert.equal(spec.info.version, "1.0.0");

  // POST with new device → minor
  await client.post("/topology", { dtm: { ...BASE_DTM, devices: { ...BASE_DTM.devices, new_dev: ... } } });
  spec = await client.get("/asyncapi");
  assert.equal(spec.info.version, "1.1.0");

  // POST with removed device → major
  await client.post("/topology", { dtm: { ...BASE_DTM, devices: {} } });
  spec = await client.get("/asyncapi");
  assert.equal(spec.info.version, "2.0.0");
});
```

## Migration

No data migration needed. New `version` column gets `synchronize: true` schema update. Pre-existing rows (none in production) would default to NULL; if encountered, `getLatestRow` reads as NULL and `nextVersion(null, ...)` returns `"1.0.0"` — degrades gracefully.

The change to `buildSpec(dtm, version)` is a breaking signature change for any internal caller. There's only one caller (`AsyncapiService.generateSpec`) — updated in lock-step.

## Open Questions / Risks

1. **Identical-DTM writes still create rows.** Current behavior persists every POST regardless of content. C2 keeps this — version stays the same but a new row is created. Audit lineage benefits ("operator POST'd at time T but didn't change anything") outweigh row count cost.

2. **Diff is structural, not semantic.** A device with the same `device_id` but renamed via "delete + add" would be classified as `major` (remove) + `minor` (add) → `major`. That matches operator intent ("identifier was lost, downstream codegen breaks"). But two operators independently swapping the connection of the same device by deep-equal-different but functionally-equivalent values would still bump patch. Acceptable.

3. **Template body changes mid-deployment.** If the bundled catalog updates (e.g., `bess_module` template adds a new measurement) and the next DTM POST embeds the new template body, `templates_used[slug] body changed` triggers `major`. This is correct — codegen contracts changed.

4. **No rollback support.** Once `2.0.0` is emitted, there's no `POST /topology/rollback`. Consumers see `2.0.0` and re-fetch. To "rollback" the operator must POST the prior DTM, which (per diff rules) is itself a `major` change forward → `3.0.0`. Versions only go up. This is per ADR-002 §10.

5. **Pre-1.0.0 semantics.** ADR-002 §10 doesn't define a beta phase. We use `1.0.0` for the bootstrap seed, treating the first POST as "release v1." If the deployment lifecycle wants `0.x.y` for "in commissioning," that's a future extension.

## Implementation Order (Hint to writing-plans)

Roughly:
1. `dtm_diff.ts` + tests (pure function; foundation for everything else)
2. `topology.entity.ts` adds `version` column
3. `topology.service.ts` wires `diffDtm` + `nextVersion` into save; adds `getLatestRow`
4. `spec-generator.ts` accepts `version`
5. `asyncapi.service.ts` fetches version
6. Integration tests
7. Readme cleanup
8. Push + CI

Single PR. ~7 commits. The plan from writing-plans will detail step-by-step.
