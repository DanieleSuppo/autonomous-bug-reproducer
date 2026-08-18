# Shared Run/Artifact Store: free-tier provider research

**Research date:** 2026-08-18  
**Scope:** A shared, persistent ABR Run/Artifact Store used from developer laptops and CI. This is deployment research only; it does not select an implementation or change the current SQLite-first MVP design.

## Decision summary

Use **Neon Free Postgres for structured Run Records and Artifact References**, with an **optional Cloudflare R2 bucket for artifact bodies**. Start with a single shared Neon project and partition records by an immutable `project_key`; do not create a database project per ABR project unless separate lifecycle or credentials become necessary. Store artifact bytes in R2 from the outset if retention could exceed Neon's 0.5 GB project limit; otherwise bounded small artifacts can remain in the database while the Store contract exposes a locator either way.

This has a $0 documented usage path for a small MVP, works from laptops and CI through standard Postgres/S3 credentials, and keeps the future hosted-service path conventional: PostgreSQL for metadata plus S3-compatible object storage for larger immutable content. It also avoids adopting user authorization or a multitenant API before ABR needs either.

**Important:** Free tiers are not a backup policy. The recommended MVP must export or otherwise copy authoritative metadata and artifact manifests on a defined schedule before treating retained history as irreplaceable.

## ABR constraints considered

The repository requires sanitized, immutable structured Run Records and Artifact References; artifact references include a content digest, media type, size, sensitivity, and optional Store locator. It also requires an explicit per-Run retention policy and prohibits retaining credentials in records or artifacts. The present specification identifies SQLite as the first persistent adapter while reserving PostgreSQL and object storage as future Store adapters. [1]

The PRD explicitly keeps canonical Run/Artifact persistence separate from result publication and does not mandate a storage technology. [2]

This research therefore evaluates a shared store on these axes:

| Requirement | Interpretation for this decision |
| --- | --- |
| Shared history | One reachable provider account/project for laptops and CI, not local files synchronized ad hoc. |
| Per-project partitioning | A required `project_key` on every logical record and in artifact key prefixes; no end-user authorization is needed yet. |
| Low/zero MVP cost | A permanent free tier, or an explicitly rejected time-limited free tier. |
| Artifact retention | Metadata must remain queryable; byte retention must have known capacity, lifecycle controls, and an escape path when artifacts grow. |
| Credentials from config | Provider secrets are supplied at runtime (for example, `DATABASE_URL` and S3 credential variables), never persisted by ABR. |
| Hosted-service path | Standard Postgres and S3-compatible interfaces are preferred over provider-specific application APIs. |

## Current offerings

| Provider | Free allocation and availability | Artifact fit | Connectivity constraints | Assessment |
| --- | --- | --- | --- | --- |
| **Neon Postgres Free** | Permanent, no-card $0 plan: 100 projects, 100 CU-hours/project/month, 0.5 GB storage/project, 5 GB public egress/month; compute scales to zero after five inactive minutes. Exceeding a free monthly compute or egress limit suspends compute until the next billing period. [3] | 0.5 GB is viable only for records plus small/bounded artifacts. One manual snapshot and six hours of point-in-time history are not a sufficient artifact-retention strategy. [4] | Native TLS Postgres; connection string is intended for an environment variable. The built-in transaction pooler accepts up to 10,000 client connections, but actual direct connections depend on compute size: 0.25 CU permits 104 total, seven reserved. Transaction pooling cannot support session state, `LISTEN`/`NOTIFY`, or SQL `PREPARE`; use direct connections for migrations and `pg_dump`. [5][6] | **Recommended metadata database.** Standard Postgres, generous project count, and pooling are a good laptop/CI fit. Capacity and weak free backup history are material risks. |
| **Supabase Postgres Free** | Two active free projects; each has a 500 MB Nano database. The Free organization has 1 GB Storage, 5 GB egress, and 5 GB cached egress. A free project pauses after one week of inactivity; free has no automatic backups or PITR. [7] | The included 1 GB organization-wide Storage is convenient for artifacts, but it caps each upload at 50 MB. [7] | Nano has 60 direct DB connections and 200 pooler clients. On Free, direct Postgres is IPv6-only; shared Supavisor session/transaction poolers are IPv4. Transaction mode does not support prepared statements. [8][9] | **Viable all-in-one alternative**, especially for small artifacts. The inactivity pause, 500 MB database, lack of automated backups, and IPv4/direct-connection split make it a weaker CI history store. |
| **Render Postgres Free** | One active free database/workspace, 1 GB capacity. It expires 30 days after creation, is inaccessible unless upgraded, and is deleted after a further 14-day grace period. [10] | The capacity is usable in isolation, but the mandatory expiration contradicts persistent shared run history. | Free databases have no backups or managed connection pooling; Render may restart or perform maintenance at any time. Its documented under-8-GB connection limit is 100. [10][11] | **Rejected.** The 30-day deletion path makes it unsuitable for an authoritative persistent ABR Store. |

### Optional artifact object store: Cloudflare R2

R2 is an appropriate companion to Neon when artifact bytes outgrow the database limit. Its documented Standard free tier includes 10 GB-month storage, one million Class A operations, ten million Class B operations, and free direct egress each month. Free-tier use is Standard storage only. R2 uses an S3-compatible API, so a Store adapter can avoid a Cloudflare-specific data API. [12][13]

R2 has lifecycle rules that can expire objects by prefix, which maps cleanly to the ABR retention policy. Objects normally disappear within 24 hours of their scheduled expiry; an object is no longer billable once deleted. R2 reads, writes, deletes, and lists are strongly consistent when accessed directly through its API. [14][15]

R2 API credentials need an R2 API token; tokens can be scoped to one or more buckets with object read/write or read-only permission. Cloudflare states that R2 must be purchased before generating its API token, so confirm the billing-account requirement even when expected usage remains within the free allowance. [16]

## Recommended MVP shape

### Storage layout

1. Retain sanitized Run Records, retention metadata, and Artifact References in one Neon Postgres database.
2. Partition every table by a required `project_key`; make a project-local Run identifier unique with `UNIQUE (project_key, run_id)`. This is logical partitioning, not end-user tenancy or authorization.
3. For R2-backed bodies, use immutable object names such as `abr/v1/<project_key>/<run_id>/<sha256>`. Persist the digest, size, media type, sensitivity, and object locator in Postgres only after a successful upload.
4. Apply R2 lifecycle rules per key prefix only when an ABR retention policy calls for expiration. Preserve the database tombstone/digest when the contract requires it.
5. Keep bounded small artifact bodies in Postgres only if the aggregate database footprint is comfortably below Neon's 0.5 GB cap. Move large archives, traces, and audit bundles to R2 rather than allowing an unbounded Postgres blob table.

### Runtime configuration

Use one trusted service credential set, supplied independently to each laptop and CI runner:

| Purpose | Configuration value | Constraint |
| --- | --- | --- |
| Postgres runtime | `DATABASE_URL` | Use Neon's pooled URL for normal concurrent run writes; use the direct URL for migrations, export, and administration. [5][6] |
| Artifact runtime | `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` | Scope the R2 token to the one ABR artifact bucket with Object Read & Write. [16] |
| Partitioning | `ABR_PROJECT_KEY` | Treat this as required run configuration and copy it into the sanitized record, not as a secret. |

This is deliberately a trusted-operator design. There is no browser-exposed database credential, end-user identity, or row-level authorization layer. If ABR becomes a hosted service, replace the broad runtime credentials with a service backend and introduce authorization at that boundary; the Postgres schema and S3 object keys remain usable.

## Risks and operating limits

| Risk | Effect | MVP mitigation |
| --- | --- | --- |
| Neon free storage is 0.5 GB/project. | Writes that grow storage beyond the free cap fail; unbounded traces or archives can exhaust it quickly. [4] | Use R2 for large bodies, enforce artifact-size/retention policy, and monitor database size. |
| Neon free compute or public egress exhausts. | Compute is suspended until the next billing period. [3] | Use a small bounded client connection pool, retry transient wake-up failures, and keep bulky artifact reads on R2. |
| Neon Free has six-hour history and one manual snapshot. | Provider recovery does not satisfy long-term backup expectations. [4] | Schedule a credential-safe logical export of metadata and a manifest/object inventory to a separately controlled location. Test restore. |
| Supabase Free pauses after one week inactive. | The first CI/laptop access after idle time can be unavailable until the project is restored; it is unsuitable where continuous immediate availability matters. [7] | Prefer Neon for the authoritative shared store, or explicitly tolerate/manual-test Supabase recovery. |
| Supabase Free lacks automated backups and PITR. | Retention does not equal recoverability. [7] | Treat it as an alternative only with independent exports. |
| Supabase Free direct connections are IPv6-only. | IPv4-only CI or laptop networks must use the shared pooler, which changes connection semantics. [8] | Use Supavisor session mode for persistent IPv4 clients and transaction mode only where its prepared-statement limitation is compatible. |
| Render Free expires and deletes the database. | Historical run evidence disappears on a fixed schedule. [10] | Do not use as an ABR Store of record. |
| R2 free-tier usage can exceed quotas; token creation requires purchasing R2. | A billing account may be required and costs can begin beyond free allocations. [12][16] | Set account alerts/budget controls outside ABR, restrict credentials, and account for operations as well as GB-months. |
| Shared trusted credentials provide no user isolation. | Anyone holding them can read/write every MVP project's records and artifacts. | Limit distribution to trusted developers and CI, rotate credentials on staff/repository changes, and add a backend authorization boundary before external users are introduced. |

## Why not select Supabase as the default

Supabase's 1 GB integrated Storage makes it the simplest one-provider option, and it remains a reasonable choice if the team values that convenience above idle availability. However, its free Nano database is the same 500 MB order of magnitude as Neon, it pauses after one inactive week, has no automated backups or point-in-time recovery on Free, and requires pooler selection for common IPv4-only CI environments. [7][8][9] Neon plus optional R2 makes availability and artifact capacity independent, while retaining standard database/object-store interfaces for a hosted evolution.

## Source notes

All external sources below are first-party provider pricing or documentation. Product limits and pricing can change; recheck them when provisioning or graduating from the MVP.

1. [ABR specification: Run Record, security, retention, and adapters](../../SPEC.md#run-record-evidence-security-and-retention)
2. [ABR PRD: Persistence vs publication](../PRD.md#104-persistence-vs-publication)
3. [Neon: Pricing plans](https://neon.com/pricing)
4. [Neon: Plans](https://neon.com/docs/introduction/plans)
5. [Neon: Connection pooling](https://neon.com/docs/connect/connection-pooling)
6. [Neon: Connect from any application](https://neon.com/docs/connect/connect-from-any-app)
7. [Supabase: Pricing](https://supabase.com/pricing)
8. [Supabase: Connect to your database](https://supabase.com/docs/guides/database/connecting-to-postgres)
9. [Supabase: Compute and Disk](https://supabase.com/docs/guides/platform/compute-and-disk)
10. [Render: Deploy for Free](https://render.com/docs/free)
11. [Render: Create and Connect to Render Postgres](https://render.com/docs/postgresql-creating-connecting)
12. [Cloudflare R2: Pricing](https://developers.cloudflare.com/r2/pricing/)
13. [Cloudflare R2: Get started with S3-compatible tools](https://developers.cloudflare.com/r2/get-started/s3/)
14. [Cloudflare R2: Object lifecycles](https://developers.cloudflare.com/r2/buckets/object-lifecycles/)
15. [Cloudflare R2: Consistency model](https://developers.cloudflare.com/r2/reference/consistency/)
16. [Cloudflare R2: Authentication](https://developers.cloudflare.com/r2/api/tokens/)
