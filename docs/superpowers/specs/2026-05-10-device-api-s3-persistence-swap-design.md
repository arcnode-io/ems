# Device-API Persistence Swap (Postgres → S3) — Design

**Goal:** Swap `ems-device-api`'s persistence backend from Postgres + TypeORM to S3. Monotonic version counter moves into the DTM JSON itself.

**Stays stateful.** Device-api still has an in-memory DTM cache and serves a versioned topology. Only the persistence backend changes.

**Why:** Audit history was silently introduced in PRs 1–5 via the `topology` table's row-per-save pattern — unapproved scope that justified Postgres, migrations, repository mocks, and a testcontainer. Removing it eliminates the entire database layer.

**Out of scope:**
- `edp-api` — untouched; tests assume DTM in S3 carries a `version` field (default `"1.0.0"` if absent)
- Per-device CRUD endpoints — deferred
- Multi-replica device-api — single replica in v1
- Audit history — explicit non-goal
- Dynamic schema generation — Zod schemas in `src/topology/dtm.schema.ts` are baked at build time and evolve with arcnode product releases, not at runtime

---

## Architecture

```
                S3 (canonical store)
                └─ dtm.json (contains `version` field)
                    ▲
            edp-api │  (writes once at build time — ISO/CFN; no runtime connection)
                    │
                    ▼
               device-api (stateful, S3-backed)
                 ├─ in-memory cache of current DTM
                 ├─ POST /topology → fetch + bump + PUT + cache + broadcast
                 ├─ GET /topology  → cache
                 ├─ GET /asyncapi  → derive from cache
                 └─ MQTT pub system/topology_changed { ts, version }
                    │
                    ▼
              gateway + line-controller + hmi
```

**Writer model:** edp-api writes once at build time. At runtime, device-api is the **sole writer** of the S3 object. Cache is authoritative between writes by construction.

---

## State elimination

| File / concern | Action |
|---|---|
| `src/topology/topology.entity.ts` | Delete |
| `TypeOrmModule.forFeature([Topology])` in `topology.module.ts` | Remove |
| `TypeOrmModule.forRoot({...})` in `app.module.ts` | Remove |
| `synchronize: true` | Remove |
| `@nestjs/typeorm`, `typeorm`, `pg` in `package.json` | Drop |
| Postgres testcontainer in `tests/fixtures/containers.ts` | Drop |
| `postgresHost` / `postgresPort` in `cfg.yml` | Drop |
| `POSTGRES_PASSWORD` in `template-secrets.env` | Drop |
| `Repository<Topology>` mocks across `*.test.ts` | Replace with S3 client mocks |
| Audit history (row-per-save) | Gone |

**What stays:** `topology.service.ts` (signature unchanged, internals rewritten), `src/asyncapi/`, Day-1 boot flow, MQTT broadcast, template-catalog validation, all HTTP endpoint signatures, LocalStack testcontainer.

---

## Save flow (POST /topology)

```
TopologyService.save(dtm):
  1. validateAgainstCatalog(dtm)
  2. headObject(s3Key)
       → { etag: e1, currentDtm } on hit
       → NoSuchKey → bootstrap: currentDtm = null, etag = null
  3. priorVersion = currentDtm?.version ?? "1.0.0"
  4. nextVersion  = nextMonotonicVersion(priorVersion)   // existing helper
  5. newDtm = { ...incomingDtm, version: nextVersion }
  6. putObject(s3Key, newDtm,
       IfMatch: e1  if not bootstrap,
       no IfMatch   if bootstrap)
  7. cache.set(newDtm)
  8. mqttClient.publishTopologyChanged(nextVersion)
  9. return newDtm
```

**Failure responses:**

| Cause | HTTP |
|---|---|
| Catalog validation fails | 400 |
| S3 returns 412 (concurrent writer) | 409 Conflict |
| S3 unreachable | 503 (fail loud, no in-process retry) |
| MQTT publish fails | log warning, continue (producer fails loud, consumer owns resilience) |

No save history, no rollback, no retry loops.

---

## Boot + read flow

**Boot** (extends existing PR 4 / ADR-003 behavior):

```
on bootstrap:
  1. dtm = fetchDtmFromS3()                   // may throw → fatal exit, orchestrator restarts
  2. if dtm.version is missing → "1.0.0"
  3. cache.set(dtm)
  4. HTTP server starts accepting traffic
```

**Reads:**

```
GET /topology  → cache.get()
GET /asyncapi  → generateAsyncApiSpec(cache.get())   // info.version = dtm.version
```

Cache is coherent with S3 by construction (single writer = device-api itself, updates cache atomically with PUT). No re-fetch on read.

**Runtime S3 outage:**

| Scenario | GET /topology | GET /asyncapi | POST /topology |
|---|---|---|---|
| S3 unreachable, DTM still in S3 | ✅ cache | ✅ cache | ❌ 503 |
| Pod restart while S3 unreachable | ❌ boot fatal → restart loop | same | same |

Canonical behavior in [ADR-003 failure-modes table](../../../boot_strategy_adr.md).

---

## Test setup

**Device-api repo:**

| Container | Action |
|---|---|
| Postgres | Drop |
| LocalStack | Keep (already from PR 4) |
| emqx | Keep (PR 6) |

Test changes:
- Unit tests: `Repository<Topology>` mocks → S3 client mocks (`getObject`, `putObject`, `headObject`).
- Integration tests: POST DTM → assert lands in LocalStack S3 with bumped `version`, assert beacon fires.
- New test: ETag concurrency — two parallel POSTs, one wins (200), one loses (409).
- Existing Day-1 boot tests: unchanged semantics.

**Gateway stub e2e test:** see [gateway stub spec](2026-05-10-gateway-stub-contract-validation-design.md). Spec needs minor revision after this refactor lands (drop Postgres from its container list; net 4 containers: LocalStack + device-api + emqx + mock-modbus-server).

---

## Libraries

- `@aws-sdk/client-s3` — already in use from PR 4 Day-1 boot; reuse existing wrapper.
- ETag handling — native S3 response headers; no extra dep.
- In-memory cache — plain class field on `TopologyService`; no cache lib.

---

## Migration

No production deployments exist. Single PR lands the refactor end-to-end. Post-merge: rebuild + publish `ems-device-api` Docker image to GitLab registry (gateway stub will pull it in the next sub-project).

---

## Net impact

- Removed: ~108 LOC, 3 npm packages, 1 testcontainer
- Added: ~70 LOC, 0 new deps, 0 new testcontainers
- Net: ~38 LOC deletion + one fewer collaborator in `TopologyService`
