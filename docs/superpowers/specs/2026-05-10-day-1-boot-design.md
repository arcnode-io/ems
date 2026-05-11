# Sub-Project B — Day-1 Boot Design

**Status:** Draft
**Date:** 2026-05-10
**Sub-project:** B of {A: Foundation (done), B: Day-1 boot, C: Dynamic CRUD}
**Depends on:** Sub-project A foundation (PRs 1-3 shipped), ADR-003 Boot DTM Source Strategy

## Context

Sub-project A landed the canonical Dtm shape, the templates layer, and catalog-aware validation on `POST /topology`. What's missing: when an operator boots an ISO, deploys a CFN/EC2 stack, or runs `docker compose up`, `ems-device-api` starts with an empty topology table. `GET /topology` returns 404. The gateway, line-controller, and HMI fetch `/asyncapi` and find nothing useful.

This sub-project closes that gap by fetching a DTM from an S3-compatible endpoint at startup. Per [ADR-003 Boot DTM Source Strategy](../../../boot_strategy_adr.md), **one mechanism (S3 GET with configurable endpoint URL)** covers cloud (real S3), on-prem ISO (minio), and dev (LocalStack). The behavior matrix and failure modes are normative there; this spec is the implementation contract for ems-device-api.

Out of scope:
- Dynamic CRUD per ADR-002 §14 (sub-project C)
- AsyncAPI semver bump or `system/topology_changed` broadcast on boot seed (sub-project C)
- Mounted-file fallback or bundled default DTM — explicitly rejected by ADR-003

## Goals

1. ems-device-api fetches a JSON-serialized canonical Dtm from `cfg.boot_dtm_s3_url` at startup using `cfg.s3_endpoint_url` (null = real AWS S3).
2. Validates (Zod + bundled catalog) and persists to the topology table.
3. Behavior matches ADR-003 §"Behavior matrix" exactly:
   - URL set + table empty → fetch + parse + validate + seed
   - URL set + table populated → fetch + skip seed (don't overwrite)
   - URL unset → graceful empty start
   - Any fetch/parse/validate/catalog error when URL set → fatal exit
4. Existing `POST /topology` path stays untouched.

## Non-goals

- Generating the boot DTM. Production: edp-api's emit pipeline writes to `s3://arcnode-artifacts/...`. Dev: developer uploads sample DTM to LocalStack.
- Fallback layers (env var, file mount, bundled default) — ADR-003 explicitly rejects these.
- Re-seed semantics for an already-populated table.

## Design

### Configuration

Add to `cfg.yml`:

```yaml
local:
  # ... existing keys
  bootDtmS3Url: ~                          # null in dev → skip fetch
  s3EndpointUrl: http://localhost:4566     # LocalStack
beta:
  # ... existing keys
  bootDtmS3Url: ~                          # set per-deployment (CFN parameter, k8s env, etc.)
  s3EndpointUrl: ~                         # null → real AWS S3
```

Add to `src/config.ts` Zod schema:

```typescript
const Config = z.object({
  // ... existing
  bootDtmS3Url: z.string().nullable(),
  s3EndpointUrl: z.string().nullable(),
});
```

### S3 client

Add `@aws-sdk/client-s3` dependency. Use existing `manifest_client.py` pattern from edp-api as the reference:

- Construct S3Client with optional `endpoint` override
- Force path-style addressing when endpoint is set (LocalStack/minio compatibility)
- Standard credential chain (IAM role on EC2, env vars on dev)

### Boot flow

The seeding step lives in `bootstrap()` in `src/main.ts`, between `NestFactory.create(...)` and `app.listen(...)`. Sequence:

```
1. cfg = loadConfig()
2. setupLogger(cfg.logLevel)
3. app = await NestFactory.create(AppModuleWithDatabase)
4. await seedFromS3(app, cfg.bootDtmS3Url, cfg.s3EndpointUrl, logger)   ← new
5. SwaggerModule.setup(...)
6. await app.listen(cfg.port, cfg.host)
```

The seed step runs synchronously before the HTTP server listens. If it throws (fatal cases per ADR-003), the process exits. Docker restart policy (via docker-compose) handles retry.

### `seedFromS3` contract

New module: `src/seed/seed_from_s3.ts` (free function, not NestJS service — one-shot, doesn't need DI lifecycle).

```typescript
export async function seedFromS3(
  app: INestApplicationContext,
  url: string | null,
  endpointUrl: string | null,
  logger: Logger,
): Promise<void>;
```

Behavior:

```
1. if url is null:
     logger.info("no boot_dtm_s3_url configured; starting empty")
     return

2. parse url → { bucket, key }
   - format: s3://<bucket>/<key>
   - reject non-s3:// schemes → fatal

3. client = new S3Client({
     endpoint: endpointUrl ?? undefined,
     forcePathStyle: !!endpointUrl,
   })

4. response = await client.send(new GetObjectCommand({ Bucket: bucket, Key: key }))
   - 404 / NotFound → fatal (URL configured but object missing)
   - auth/network/other errors → fatal

5. body = await streamToString(response.Body)

6. raw = JSON.parse(body)
   - SyntaxError → fatal

7. dtm = Dtm.parse(raw)
   - ZodError → fatal (schema drift)

8. service = app.get(TopologyService)
9. service.validateAgainstCatalog(dtm)
   - BadRequestException → fatal (catalog drift)

10. existing = await service.getLatest()
11. if existing !== null:
      logger.info(`topology already populated; skipping seed from ${url}`)
      return
12. await service.save(dtm)
13. logger.info(`seeded topology from ${url}`)
```

All "fatal" paths propagate the throw. `bootstrap()` doesn't catch; process exits with nonzero; container restart loop until config/state is fixed.

### File location summary

- **Config:** `src/config.ts` gains `bootDtmS3Url` + `s3EndpointUrl`; `cfg.yml` gains both keys under both environments.
- **Seed function:** `src/seed/seed_from_s3.ts` (new file).
- **S3 dependency:** `@aws-sdk/client-s3` added to `package.json`.
- **Tests:** `src/seed/seed_from_s3.test.ts` (colocated unit), `tests/seed.test.ts` (integration with LocalStack testcontainer).
- **Main wiring:** `src/main.ts` adds the await call to `seedFromS3(...)`.

### Test plan

#### Unit (src/seed/seed_from_s3.test.ts)

Mock `S3Client` (the SDK is mockable; or use a `S3Client`-shaped interface for DI):

- url is null → returns silently with info log
- url malformed (not s3://) → throws
- S3 returns NotFound → throws
- S3 returns auth error → throws
- Object body invalid JSON → throws SyntaxError
- Object valid JSON but Zod fails → throws ZodError
- Object valid + Zod passes + catalog rejects → throws BadRequestException
- All valid + table empty → service.save called with parsed Dtm
- All valid + table populated → service.save NOT called

#### Integration (tests/seed.test.ts)

Use `@testcontainers/localstack` to spin up LocalStack with S3 enabled. Upload a known-good Dtm fixture to LocalStack. Use a parallel pattern to existing `tests/topology.test.ts`:

- Empty DB + url set + valid object in S3 → service starts; `GET /topology` returns the seeded DTM
- Empty DB + url unset → service starts; `GET /topology` returns 404
- Populated DB + url set with new DTM → service starts; `GET /topology` returns the EXISTING DTM (not the S3 object)
- Empty DB + url set + S3 NotFound → service crashes (fatal)

Existing tests use `overrideProvider(TEMPLATE_CATALOG)`. Add parallel `overrideProvider(<S3_CLIENT_TOKEN>)` if we DI the S3 client; otherwise pass mock client via the `seedFromS3` arg.

#### CI integration

The `check` stage in `.gitlab-ci.yml` runs unit + integration. Both new tests run there. LocalStack testcontainer is the standard pattern (already used by edp-api).

### Documentation deliverables

Add a "Day-1 Boot" section to `ems-device-api/readme.md`:

- Cloud (CFN + EC2 + docker-compose): set `bootDtmS3Url` to `s3://arcnode-artifacts/deployments/<id>/dtm.json`. EC2 instance IAM role grants `s3:GetObject`. `s3EndpointUrl` unset.
- On-prem ISO: set `bootDtmS3Url` to the canonical key. `s3EndpointUrl` to the on-prem minio (`http://minio:9000`). minio populated at imaging time.
- Dev (docker-compose): LocalStack container in compose. Sample DTM uploaded via `aws --endpoint-url=http://localhost:4566 s3 cp ...` at startup or via init container.
- Tests/CI: leave `bootDtmS3Url` unset; tests POST DTMs explicitly.

Total addition is ~50 lines. Companion to existing "Diagrams" section.

## Migration

No DB migration. No breaking change to `POST /topology`. The seed path is purely additive — when `bootDtmS3Url` is unset, behavior is identical to today.

Existing tests continue to pass:
- Their `bootstrap`-equivalent doesn't go through `main.ts`'s new seed step (NestJS Test module assembly bypasses it).
- The integration tests already mount/override `TEMPLATE_CATALOG`; we add a parallel override path for the S3 client if we DI it.

## Risks

| Risk | Mitigation |
|---|---|
| S3 outage during boot | Docker restart policy on the device-api container; if S3 is down, deployment has bigger issues |
| Stale `bootDtmS3Url` pointing at missing object | Fatal exit, error names the URL; operator updates deployment |
| ISO deployment without minio populated | Fatal exit; docs in readme + ISO bake-time runbook ensure minio is seeded |
| LocalStack credential mismatch in dev | Standard `AWS_ACCESS_KEY_ID=test`/`AWS_SECRET_ACCESS_KEY=test` pattern; documented in readme |
| Catalog drift between bundled templates and S3 DTM | Fatal exit at `validateAgainstCatalog`; PR 3 already names the unknown slug |

## Implementation Order (Hint to writing-plans)

Roughly:
1. cfg.yml + Config schema add `bootDtmS3Url` + `s3EndpointUrl`
2. Add `@aws-sdk/client-s3` dependency
3. `seedFromS3` function with unit tests (mocked S3)
4. Wire into `main.ts`
5. Integration test with LocalStack testcontainer
6. Update readme with deployment recipes
7. Push + CI

Single PR. ~6 commits. The plan from writing-plans will detail step-by-step.
