# Device-API Persistence Swap (Postgres → S3) — Design

**Goal:** Swap `ems-device-api`'s persistence backend from Postgres + TypeORM to S3 (already used for Day-1 boot per ADR-003). Monotonic version counter moves into the DTM JSON itself.

**Terminology note:** Device-api remains a *stateful* service — in-memory DTM cache, versioned topology. What changes is *where* persistence lives: S3 instead of Postgres. This is a storage backend swap, not a statelessness conversion.

**Why this exists:** Audit history was silently introduced in PRs 1–5 via the `topology` table's row-per-save pattern. That unapproved scope justified a Postgres dependency, which justified migration plumbing, repository mocks, and a testcontainer that didn't need to exist. S3-backed persistence aligns with what device-api actually is: a transform/cache layer over an S3 artifact owned by `edp-api`.

**Order of operations rationale:** Refactor lands BEFORE any consumer (gateway stub, line-controller, hmi) is built against the current shape. Architecture cleanup compounds when consumers couple to a wrong shape.

**Out of scope (do NOT touch):**
- `edp-api` — someone else is working on it. Tests assume DTM in S3 already carries a `version` field (default `"1.0.0"` if absent for bootstrap).
- Per-device CRUD endpoints (deferred per existing memory; bulk POST /topology only)
- Multi-replica device-api (single replica in v1; multi-replica cache invalidation deferred)
- Audit history of any kind — explicit non-goal

---

## Architecture

```
                      S3 (canonical store)
                      └─ dtm.json (contains `version` field)
                          ▲
                  edp-api │  (writes once at build time — ISO/CFN. No runtime connection.)
                          │
                          ▼
                     device-api (stateful service, S3-backed persistence)
                       ├─ in-memory cache of current DTM (refreshed on boot)
                       ├─ POST /topology → fetch S3 → bump version → PUT S3 → broadcast
                       ├─ GET /topology → serve from cache
                       ├─ GET /asyncapi → derive from cached DTM
                       └─ MQTT pub system/topology_changed { ts, version } on save
                          │
                          ▼
                      gateway + line-controller + hmi (consumers)
```

**Sources of truth:**
- DTM content: S3 (single object per deployment)
- Version counter: inside the DTM JSON's `version` field
- AsyncAPI spec: pure function of DTM, not persisted

**Writer model:**
- **Build time:** `edp-api` writes initial DTM (with `version: "1.0.0"`) to S3 via ISO bake or CloudFormation init. No runtime connection between edp-api and device-api.
- **Runtime:** `device-api` is the sole writer. Operator POSTs /topology → device-api fetches current, bumps patch, PUTs back. Re-deployment cycle = new ISO/CFN with new artifact; runtime mutations correctly lost (intent of re-deploy).

---

## State elimination

**What dies:**

| File / concern | Action |
|---|---|
| `src/topology/topology.entity.ts` | Delete entirely |
| `TypeOrmModule.forFeature([Topology])` in `topology.module.ts` | Remove |
| `TypeOrmModule.forRoot({...})` in `app.module.ts` | Remove |
| `synchronize: true` schema setup | Remove |
| `@nestjs/typeorm`, `typeorm`, `pg` packages | Drop from `package.json` |
| Postgres testcontainer in `tests/fixtures/containers.ts` | Drop |
| Postgres-related cfg keys (`postgresHost`, `postgresPort`) | Drop from `cfg.yml` |
| `POSTGRES_PASSWORD` env var | Drop from `template-secrets.env` |
| `Repository<Topology>` mocks across `*.test.ts` | Replace with S3 client mocks |
| Save history (row-per-save) | Unapproved scope — gone |

**What stays:**

- `src/topology/topology.service.ts` — external signature unchanged; internals rewritten to use S3 client + cache
- `src/asyncapi/` — pure function of DTM; no change
- Day-1 S3 boot flow (PR 4) — now refreshes cache instead of seeding DB
- `src/mqtt/` — MQTT broadcast on save (PR 6); unchanged
- Template catalog validation
- All HTTP endpoints (external contract unchanged)
- LocalStack testcontainer (already in use for PR 4)

---

## Save flow

### POST /topology behavior

```
TopologyService.save(dtm):
  1. validateAgainstCatalog(dtm)
  2. headObject(s3Key) → { etag: e1, currentDtm }
     If S3 returns NoSuchKey → bootstrap path: currentDtm = null, etag = null
  3. priorVersion = currentDtm?.version ?? "1.0.0"
  4. nextVersion = nextMonotonicVersion(priorVersion)   // existing helper
  5. newDtm = { ...incomingDtm, version: nextVersion }
  6. putObject(s3Key, newDtm, IfMatch: e1) on update path,
     or putObject(s3Key, newDtm) on bootstrap path (no IfMatch)
       → 200 on success
       → 412 Precondition Failed → 409 Conflict to client
  7. cache.set(newDtm)
  8. mqttClient.publishTopologyChanged(nextVersion)
  9. return newDtm
```

### Concurrency model

S3 optimistic locking via ETag + `If-Match` header. v1 single-replica device-api with edp-api as build-time-only writer = effectively zero contention. ETag is cheap insurance for:
- Two simultaneous operator POSTs (extremely rare; surfaced as 409 → client retries)
- Future multi-replica scenarios (not v1)

### Failure modes

- S3 unreachable → 503 Service Unavailable. Fail loud, let the runtime stack trace.
- 412 Precondition Failed → 409 Conflict to client.
- Catalog validation fails → 400 Bad Request (unchanged).
- MQTT broadcast fails → log warning + continue (per saved feedback: producer publishes don't retry, consumer owns resilience).

**No defensive scope:** no save history, no rollback path, no retry loops on the S3 PUT.

---

## Boot + read flow

### Boot

```
on bootstrap:
  1. dtm = fetchDtmFromS3()    # may throw if S3 unreachable or NoSuchKey
  2. if dtm.version is missing → default to "1.0.0"
  3. cache.set(dtm)
  4. HTTP server starts accepting traffic
```

If S3 unreachable at boot → fail fast (existing PR 4 / ADR-003 behavior).

### Read endpoints

```
GET /topology:
  return cache.get()

GET /asyncapi:
  dtm = cache.get()
  return generateAsyncApiSpec(dtm)    # info.version = dtm.version
```

### Why no S3 refetch on every read

- Cache invalidation handled by write path (`save()` updates cache atomically with successful PUT)
- No external writers at runtime (edp-api is build-time only)
- v1 single-replica → cache always coherent with S3

**Future-proofing (NOT v1):** multi-replica would refetch on receipt of `system/topology_changed` beacon from sibling replicas. Cheap to add later.

---

## Test setup

### Device-api repo

| Container | Change |
|---|---|
| Postgres testcontainer | **Drop** |
| LocalStack | Stays (already in use from PR 4) |
| emqx | Stays (PR 6 broadcast tests) |

Tests rewritten:
- **Unit tests** (`topology.service.test.ts`): replace `Repository<Topology>` mocks with S3 client mocks (`getObject`, `putObject`, `headObject`). AAA pattern preserved.
- **Integration tests** (`tests/topology.test.ts`): bootstrap device-api against LocalStack + emqx. POST a DTM, assert it lands in S3 with bumped version, assert MQTT beacon fires.
- **New integration test**: ETag concurrency — two parallel POSTs, one wins, one gets 409.
- **Existing Day-1 boot test**: unchanged semantics; behavior matrix still applies (`bootDtmS3Url` set + empty cache → fetch + seed).

### Gateway repo (deferred until after this refactor)

Gateway stub spec (`2026-05-10-gateway-stub-contract-validation-design.md`, committed locally) needs revision before execution to reflect:
- Drop Postgres from container list
- 4 containers: LocalStack + device-api + emqx + mock-modbus-server
- Real device-api in the test (locked at user direction)

Revision happens after this refactor lands.

---

## Library choices

| Concern | Library | Notes |
|---|---|---|
| S3 client | `@aws-sdk/client-s3` | Already in use for PR 4 Day-1 boot; reuse the existing wrapper |
| ETag handling | Native S3 response | No extra lib |
| In-memory cache | Plain class field on `TopologyService` | YAGNI on cache lib |

---

## Migration / rollout

- No production deployments exist yet — no data migration needed.
- One PR lands the refactor end-to-end.
- Post-merge: rebuild + publish device-api Docker image to GitLab registry (so gateway test can pull it in the next sub-project).

---

## Net LOC impact

- Removed: ~108 LOC + 3 npm packages + 1 testcontainer
- Added: ~70 LOC + 0 new deps + 0 new testcontainers (LocalStack already present)
- Net: **~38 LOC deletion + simpler mental model** (one collaborator instead of two)
