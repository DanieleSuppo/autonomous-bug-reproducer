# Shared ABR Run/Artifact Store: current zero-cost offerings

**Research date:** 2026-08-18  
**Scope:** A trusted, shared Store used by ABR on developer laptops and CI runners. Runners must be able to create immutable Run Records and artifact references from different machines. Each logical ABR project needs its own partition. End-user authorization is explicitly out of scope. AWS is not evaluated: the operator already has an AWS account and does not expect new-account free-tier eligibility.

This is deployment research only. It does not change the SQLite-first Store named by the current specification. The specification requires sanitized immutable Run Records, Artifact References, explicit retention, and no persisted credentials. [1]

## Decision constraints

| Constraint | Implication |
| --- | --- |
| Shared writers | The database/object store, not a synced SQLite file, coordinates concurrent laptop and CI writes. |
| Logical project partition | Make `project_key` required in every record and collection/table query; prefix every object key with `abr/v1/<project_key>/`. This is not authorization. |
| No user authorization yet | Trusted runners may hold a shared, least-privilege machine credential. Do not expose that credential to browsers or untrusted repositories. A future hosted service replaces it. |
| Artifact handling | Keep metadata and lifecycle state with the Run Record; store bytes in an object store where available. Use immutable digest-addressed keys and conditional/idempotent writes. |
| Zero-cost claim | "Zero-cost" means documented use remains within the listed permanent free allocation. It does not mean that a provider cannot require a billing account or charge an overage. |

## Comparison

| Offering | Exact free allocation and expiry | Billing or card requirement | Can trusted runners write directly? | Artifact suitability | Deployment and operations |
| --- | --- | --- | --- | --- | --- |
| **Cloudflare D1 + R2** | D1 on Workers Free: 5 million rows read/day, 100,000 rows written/day, and 5 GB total D1 storage; limits reset at 00:00 UTC. A free D1 database is at most 500 MB, with 7-day Time Travel and 10 databases/account. R2 Standard: 10 GB-month storage, 1 million Class A operations/month, 10 million Class B operations/month, and free egress/month. R2's allowance is permanent/monthly; it does not apply to Infrequent Access. [2][3][4] | Workers Free has a $0 plan. R2 is usage-billed: Cloudflare requires R2 to be purchased before issuing its S3 API token, and may preauthorize a valid payment method for usage-billed services. Therefore this is $0-usage-capable, but **not cardless for direct R2 access**. [4][5] | **R2: yes.** Give trusted runners an S3 token scoped to one bucket and Object Read & Write. **D1: use a small Worker API.** Cloudflare documents normal D1 query execution through a Worker binding, rather than a conventional remote SQLite/Postgres driver; expose narrow Store operations from a Worker and keep D1 credentials/bindings there. [6][7] | **Strong.** R2 is S3-compatible, has strongly consistent direct reads/writes/lists, lifecycle rules, up to 5 TiB per object, and is the appropriate byte store. D1 is metadata only: its 2 MB maximum row/BLOB makes it unsuitable for general artifacts. [4][8][9] | One Cloudflare account, D1 schema/migrations, R2 bucket/lifecycle policy, and a Worker Store API. More application-specific work than a direct Postgres client, but runners receive a small HTTP interface instead of provider database credentials. A D1 database serializes queries one at a time, so use short indexed transactions and retry overload responses. [3][6] |
| **Google Cloud Firestore Standard + Cloud Storage** | Firestore: one free database/project, 1 GiB stored data, 50,000 document reads/day, 20,000 writes/day, 20,000 deletes/day, and 10 GiB/month outbound transfer. Cloud Storage Always Free: 5 GB-month Standard regional storage, 5,000 Class A operations/month, 50,000 Class B operations/month, and 100 GB/month North American egress; only `us-east1`, `us-west1`, and `us-central1` qualify. Both are permanent monthly/daily allocations, not trials. [10][11][12] | **Required.** An active Google Cloud billing account is required for Free Tier. New Free Trial signup requires a credit card or other valid payment method; a paid billing account can bill overages. Google says the Free Tier has no end date but can change with 30 days' notice. [10] | **Yes, for trusted runners.** Firestore server libraries/REST use IAM; Cloud Storage supports client libraries, CLI, and REST using Application Default Credentials. Give CI a federated or service-account identity and developers user ADC or impersonation. A Store API is not technically required before user auth, but avoids distributing broad credentials and becomes preferable once untrusted callers exist. [13][14] | **Strong for bytes, weak for records.** Cloud Storage has strong global read-after-write/list consistency and atomic individual-object uploads; use it for artifacts. Firestore works for document-shaped Run Record metadata and atomic transactions/batches, but it is not relational SQL and queries/indexes must be designed around its document model. [15][16] | Two Google APIs, IAM roles/credential delivery, Firestore collection/index management, a Cloud Storage bucket/lifecycle policy, and cost controls. This is the most operationally involved direct-writer option; its small permanent quotas are adequate only for a bounded evaluation corpus. |
| **Google Cloud SQL for PostgreSQL + Cloud Storage** | **No permanent Cloud SQL Always Free allocation.** The product offers one free trial instance per project lifecycle: 30 days, preset 8-vCPU/64-GB/100-GB configuration, then it stops serving requests. Data remains only for a further 90-day grace period before scheduled deletion. Data transfer outside the region/public Internet and final backup/storage can still be charged. Cloud Storage retains the permanent quotas above. [10][17] | **Required.** Existing users must enable Cloud Billing; new users must create a billing account with a credit card or other payment method. [17] | **Yes during the trial.** Normal PostgreSQL clients can connect through public IP plus a Cloud SQL connector, or directly with appropriate network/TLS configuration. No service is required for trusted runners, but the connection/network setup is heavier than Neon. [18] | **Strong technically.** PostgreSQL suits records and Cloud Storage suits artifacts, but the database trial and deletion schedule make it unsuitable as the authoritative persistent Store. [17][18] | **Reject for this requirement.** It has the highest operational footprint (instance, networking, connector/TLS or IP allowlists, billing) and no lasting zero-cost database path. |
| **Neon Free Postgres** | **Verified current:** Free is a permanent $0 plan with no credit card. It includes 100 projects, 10 branches/project, 100 CU-hours/project/month, 0.5 GB storage/project, 5 GB public egress/month, 6 hours of restore history (up to 1 GB-month of changes), and one manual snapshot. Free compute scales to zero after five inactive minutes. Compute/egress reset each billing period; storage is a continuing limit. [19][20] | **No card required** for the Free plan. The documented free limits are hard operational boundaries: exhausting compute or egress suspends compute until the next billing period; writes that grow past 0.5 GB fail until space is freed or the plan is upgraded. [19][20] | **Yes.** Runners use the standard TLS Postgres connection string; use the pooled endpoint for many short-lived CI/laptop clients and direct connections for migrations/export. This requires no Store service while runners remain trusted. [21][22] | **Records and small bounded artifacts only.** The 0.5 GB project storage limit makes it a poor home for archives, logs, traces, or audit bundles. Neon Object Storage is beta, advertised as no-charge during beta with unspecified usage limits; do not make it the authoritative artifact plan. Pair Neon metadata with R2 or Cloud Storage when an artifact bucket and its billing requirement are acceptable. [19][20] | Lowest operational complexity for structured metadata: one project, schema migrations, connection secret, and scheduled logical exports. Manage scale-to-zero wake-up/retry and free-tier limits. It does not itself provide a durable zero-cost general object store. |
| **Local SQLite + filesystem fallback** | No provider quota, expiry, billing account, or card. Capacity and retention are limited by the machine/disk and local backup policy. | None. | **No, not across machines.** Each runner has an independent Store; only a single runner, a shared mounted volume with its own concurrency/reliability assumptions, or explicit export/import can make history visible elsewhere. | Good for local files and bounded SQLite BLOBs. It is the existing first persistent Store direction, not a shared artifact service. [1] | Lowest complexity and the only fully cardless option. Use as the default/offline mode and for tests, but do not present it as satisfying the shared laptop/CI requirement. |

## Operational shape for every hosted option

1. Use one Store account/project for the ABR deployment, not one provider project per logical ABR project.
2. Require a stable, non-secret `project_key` in every Run Record, database query, and object key. For example: `abr/v1/<project_key>/<run_id>/<sha256>`.
3. Insert a Run Record with a generated run ID and uniqueness constraint or transaction; make retried terminal writes idempotent.
4. Upload an artifact under its immutable digest key, then commit its Artifact Reference. Record an explicit pending/failed upload state if the process ends between the two operations.
5. Sanitize before persistence, pass credentials by runtime configuration only, scope each credential to the Store/bucket, and establish a scheduled export/restore test. Provider free quotas are not a backup policy.

## Ranked recommendation

1. **Neon Free for Run Records and Artifact References, plus Cloudflare R2 for artifact bodies when a payment method is acceptable.** Neon is the only evaluated permanent, cardless, conventional relational metadata database. R2 has the largest documented artifact allocation and S3 compatibility. This is $0 within quotas, but R2 token issuance requires a purchased, usage-billed R2 account; it is not a cardless deployment. Keep a local SQLite/filesystem mode.
2. **Cloudflare D1 + R2 with a narrow Worker Store API** if one Cloudflare deployment is more valuable than direct database access. It has generous combined free quotas and a natural artifact store, but needs R2 billing setup and a Worker layer because D1's documented application interface is a Worker binding.
3. **Firestore + Cloud Storage** only if the team already operates Google Cloud IAM/billing. It supports direct trusted runners safely and Cloud Storage is a strong artifact store, but requires a billing account, has small permanent quotas, and adds document-model/index/IAM complexity.
4. **Local SQLite + filesystem** as the pure local fallback and default for offline/single-runner use. It is the only fully cardless/costless option, but does not meet the shared-store requirement without external synchronization.
5. **Do not use Cloud SQL trial as the Store of record.** It is technically capable but has a 30-day serving window, a deletion path, billing requirements, and no permanent free tier.

## Sources

All provider claims below are primary documentation current on the research date.

1. [ABR specification: Run Record, security, retention, and adapters](../../SPEC.md#run-record-evidence-security-and-retention)
2. [Cloudflare D1 pricing](https://developers.cloudflare.com/d1/platform/pricing/)
3. [Cloudflare D1 limits](https://developers.cloudflare.com/d1/platform/limits/)
4. [Cloudflare R2 pricing](https://developers.cloudflare.com/r2/pricing/)
5. [Cloudflare R2 authentication and billing policy](https://developers.cloudflare.com/r2/api/tokens/) and [usage-based billing policy](https://developers.cloudflare.com/billing/understand/billing-policy/)
6. [Cloudflare D1 Workers Binding API](https://developers.cloudflare.com/d1/worker-api/)
7. [Cloudflare D1 read replication and REST API limitations](https://developers.cloudflare.com/d1/best-practices/read-replication/)
8. [Cloudflare R2 consistency](https://developers.cloudflare.com/r2/reference/consistency/)
9. [Cloudflare R2 limits](https://developers.cloudflare.com/r2/platform/limits/)
10. [Google Cloud Free Program and Free Tier limits](https://cloud.google.com/free/docs/free-cloud-features)
11. [Firestore pricing](https://cloud.google.com/firestore/pricing)
12. [Cloud Storage pricing and Always Free limits](https://cloud.google.com/storage/pricing)
13. [Firestore server client authentication](https://cloud.google.com/firestore/docs/manage-data/add-data)
14. [Cloud Storage authentication](https://cloud.google.com/storage/docs/authentication)
15. [Cloud Storage consistency](https://cloud.google.com/storage/docs/consistency)
16. [Firestore transactions and batched writes](https://cloud.google.com/firestore/docs/manage-data/transactions)
17. [Cloud SQL for PostgreSQL free trial instance](https://cloud.google.com/sql/docs/postgres/free-trial-instance)
18. [Cloud SQL PostgreSQL connection options](https://cloud.google.com/sql/docs/postgres/connection-options)
19. [Neon pricing](https://neon.com/pricing)
20. [Neon plans](https://neon.com/docs/introduction/plans)
21. [Neon connection pooling](https://neon.com/docs/connect/connection-pooling)
22. [Neon connections from any application](https://neon.com/docs/connect/connect-from-any-app)
