# Sub-Project C2 — AsyncAPI Monotonic Version Tracking Design

**Status:** Accepted (revised 2026-05-10 — diff approach dropped, monotonic counter)
**Date:** 2026-05-10
**Sub-project:** C2 of {C1: dropped (full-load pattern), C2: monotonic version, C3: MQTT broadcast}
**Depends on:** Sub-project A foundation (PRs 1–3 shipped), sub-project B day-1 boot (PR 4 shipped)

## Revision history
- 2026-05-10 — Initial draft: full prev→next DTM diff with major/minor/patch classification.
- 2026-05-10 — Revised mid-implementation: diff approach was over-engineered for MVP. Skip-codegen-on-patch was theoretical; consumers re-fetch on every change anyway. Replaced with monotonic counter (every save bumps patch by 1). Major/minor classification deferred until consumers actually need the distinction.

## Context

Sub-project A landed canonical Dtm + templates + catalog validation. Sub-project B added boot-time DTM seeding from S3. The AsyncAPI generator at `ems-device-api/src/asyncapi/` already builds a spec from the latest persisted DTM and serves it at `GET /asyncapi`. What's missing: the spec carries a **fixed `info.version`** (currently hardcoded `"1.0.0"`) regardless of how many times the DTM has been mutated.

[ADR-002 §10](../../../topic_structure_adr.md) mandates a semver-shaped version field. The original draft of this spec implemented a 300-line prev→next diff that classified changes as patch/minor/major. During implementation we observed:

- **Skip-codegen-on-patch is theoretical.** Consumers (gateway, HMI, line-controller) all re-codegen unconditionally on `system/topology_changed` per ADR-002 §10. No consumer actually inspects the version-component pattern to skip work.
- **Diff complexity dominated.** Six per-section helper files, deep-equality JSON-stringify pitfalls (Postgres JSONB key reordering broke identity detection), severity hierarchies — disproportionate to the value delivered.
- **YAGNI applies.** The change signal that consumers actually need is "did the version string change?" — that works with any monotonic source. Operator-readable major/minor/patch can return when an operator shows up asking for it.

This sub-project replaces the pipeline with a 4-line monotonic counter:

1. **Per-row version persistence** — extend the Topology TypeORM entity with a `version: string` column.
2. **Version computation on save** — `TopologyService.save` reads the latest row's version, increments the patch component, persists.
3. **Spec emits the version** — `spec-generator.ts` accepts a `version` parameter and threads it into `info.version`.

C1 was reframed during brainstorming. The original "per-device CRUD endpoints" was deferred; clients use `GET /topology` (full-load) for navigation. The remaining C1 cleanup folds into this PR: update `ems-device-api/readme.md` to drop stale references to `/devices-0/:id/devices-1/:id` (which never existed in code post-foundation).

C3 (MQTT broadcast of `topology_changed { ts, version }`) builds on C2's `version` field.

## Goals

1. `Topology` entity gains `version: string` (semver-shaped, e.g., `1.0.42`). Default `1.0.0` for the first row.
2. `TopologyService.save` reads the latest row's version, computes the next via `nextMonotonicVersion`, persists alongside the DTM.
3. AsyncAPI generator emits the persisted version in `info.version`.
4. Every save increments — no diff, no no-op detection, no severity classification.
5. Existing endpoints unchanged in behavior except `GET /asyncapi` now reflects real version.
6. Stale readme references to `/devices-0/...` removed.

## Non-goals

- Diff-based major/minor/patch classification. Deferred until consumer demands it.
- Per-device CRUD endpoints. Deferred to future sub-project.
- MQTT broadcast (sub-project C3).
- Audit-log queries — Postgres rows are append-only; query the DB directly if needed.

## Design

### Version computation

```typescript
function nextMonotonicVersion(prev: string | null): string {
  if (prev === null) return "1.0.0";
  const [major, minor, patch] = prev.split(".").map(Number);
  return `${major}.${minor}.${patch + 1}`;
}
```

Pure, deterministic, four lines. Bootstrap → `1.0.0`. Subsequent saves → patch + 1.

### Persistence

The `Topology` TypeORM entity gains:

```typescript
@Column({ type: "varchar", length: 32, default: "1.0.0" })
version!: string;
```

`synchronize: true` (per `app.module.ts`) auto-migrates the schema on next boot. No data migration; no production rows yet.

### `TopologyService.save` flow

```typescript
async save(dtm: DtmType): Promise<Topology> {
  const prior = await this.repo.findOne({
    where: {},
    order: { receivedAt: "DESC" },
  });
  const priorVersion = prior?.version ?? null;
  const version = nextMonotonicVersion(priorVersion);
  if (priorVersion === null) {
    this.logger.log(`seeded topology v${version} (initial)`);
  } else {
    this.logger.log(`updated topology v${priorVersion} → v${version}`);
  }
  const row = this.repo.create({
    dtm: dtm as unknown as Record<string, unknown>,
    version,
  });
  return await this.repo.save(row);
}
```

Log lines per save:

```
INFO seeded topology v1.0.0 (initial)
INFO updated topology v1.0.0 → v1.0.1
INFO updated topology v1.0.1 → v1.0.2
```

### AsyncAPI generator integration

`buildSpec(dtm, version)` accepts the version as a parameter:

```typescript
export function buildSpec(dtm: DtmType, version: string): AsyncApi3Spec {
  return {
    asyncapi: SPEC_VERSION,
    info: {
      title: `ARCNODE EMS — ${dtm.deployment_uuid}`,
      version,
      description: "...",
    },
    // ...
  };
}
```

`AsyncapiService.generateSpec` fetches the latest row and threads both DTM and version through:

```typescript
async generateSpec(): Promise<Record<string, unknown> | null> {
  const row = await this.topology.getLatestRow();
  if (!row) return null;
  return buildSpec(row.dtm as DtmType, row.version) as unknown as Record<string, unknown>;
}
```

`TopologyService` exposes `getLatestRow(): Promise<Topology | null>` to return the full row including version (`getLatest` continues to return DTM-only).

### Readme cleanup

`ems-device-api/readme.md` mentions `/devices-0/:device0Id` and `/devices-0/:device0Id/devices-1/:device1Id` endpoints that never existed in code post-foundation. Replace with:

> "**Device navigation.** Clients fetch the full DTM via `GET /topology` and walk `device.parent` chains client-side. See sub-project C2 design for rationale (full-load pattern; per-device endpoints deferred until device counts cross 1k)."

### File structure

**Modify:**
- `src/topology/topology.entity.ts` — add `version` column.
- `src/topology/topology.service.ts` — add `nextMonotonicVersion` helper, wire into `save`, add `getLatestRow`.
- `src/topology/topology.service.test.ts` — tests for monotonic bump + getLatestRow.
- `src/asyncapi/spec-generator.ts` — accept `version` param.
- `src/asyncapi/asyncapi.service.ts` — fetch row and pass version through.
- `tests/topology.test.ts` — integration test asserts version bumps across multiple POSTs.
- `readme.md` — drop stale `/devices-0` text, add full-load note.

No new files. No new dependencies.

## Test plan

### Unit (`src/topology/topology.service.test.ts`)

- First save → `row.version === "1.0.0"`
- Second save → `row.version === "1.0.1"`
- 41st save → `row.version === "1.0.41"` (Nth save → patch + (N-1))
- `getLatestRow()` returns full row with version
- `getLatestRow()` returns `null` when no rows exist

### Integration (`tests/topology.test.ts`)

```
1. POST /topology → GET /asyncapi  → info.version === "1.0.0"
2. POST /topology (same DTM) → GET → info.version === "1.0.1"  (monotonic, no diff)
3. POST /topology (changed) → GET → info.version === "1.0.2"
```

## Migration

No data migration. New `version` column adds via `synchronize: true`. Pre-existing rows (none in production) would default to NULL; if encountered, `getLatestRow` reads as NULL and `nextMonotonicVersion(null)` returns `"1.0.0"` — degrades gracefully.

## Open Questions / Risks

1. **Patch component overflow.** A 32-character `varchar` holds `1.0.<int>` up to ~10^28. Practical bound is unbounded.

2. **Operator readability.** `1.0.247` doesn't tell an operator what kind of change it was. Acceptable for MVP — audit lineage lives in commit messages and the topology table itself (query `SELECT id, version, receivedAt, jsonb_pretty(dtm) FROM topology ORDER BY id`). When an operator complains, revisit and add diff classification at that time.

3. **Consumer churn.** Every save bumps version, so every save triggers consumer re-fetch via `system/topology_changed` (when C3 lands). Identical-DTM POSTs would still trigger re-fetch — wasteful but cheap. Consumers absorb a re-fetch in <100ms; not a real concern.

4. **Consumer diff capability.** Per ADR-002 §10, consumers compute the diff client-side after re-fetching the spec. If a consumer wants to discriminate "this was just a patch" without diffing client-side, they're stuck. None do today.

## Implementation Order

1. cfg/entity: `Topology` gains `version` column.
2. `nextMonotonicVersion` helper + tests in `topology.service.ts`.
3. `TopologyService.save` wires the helper + adds `getLatestRow`.
4. `buildSpec` accepts `version`; `AsyncapiService.generateSpec` threads it.
5. Integration test for end-to-end version bump.
6. Readme cleanup.
7. Push + CI.

Single PR. ~6 commits. Cleaner than the original diff-based plan.
