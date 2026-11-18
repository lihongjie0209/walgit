# Alibaba Cloud OSS: exclusive bucket ownership

Select `store.backend = "oss"` and configure the OSS endpoint, region and
credential environment variables under `[store.s3]` in
[walgit.example.toml](../walgit.example.toml). The binary needs the `s3` feature.

OSS rejects the S3 conditional PutObject headers used for manifest CAS. This
backend uses `x-oss-forbid-overwrite: true` for atomic Create, and emulates
Update with HEAD/ETag comparison followed by PUT under a lock. The lock belongs
to one `S3Store` instance. It is not a bucket lease or a cross-process fence.
Never enable or suspend bucket versioning: OSS ignores forbid-overwrite in
both states. See the [OSS API contract](https://www.alibabacloud.com/help/en/oss/developer-reference/completemultipartupload).

All writes to the bucket must go through the same store instance in exactly
one process. This restriction applies to the entire bucket, even with
separate repository assignments or prefixes. Another writer can change an
object between HEAD and PUT; both updates can appear successful while an
acknowledged push is lost. Config validation cannot discover other processes,
CLI invocations or external writers, so operators must enforce this boundary.
Use S3 or GCS when concurrent writers or rolling deployments are required.

| Operation | Supported OSS deployment |
|---|---|
| `serve`, `maintain`, `events` roles | Enable the desired roles together in one process. Do not split roles into separate services. `serve` alone can write pushes, LFS and shared render caches. |
| Restart or upgrade | Stop the old process and wait for it to exit completely before starting its replacement. SIGTERM starts draining and does not immediately stop writes. Readiness changes and a fixed delay are not proof of exit. Accept downtime; no rolling, blue/green or autoscaling overlap. |
| `walgit import`, `walgit compact` (including `--base`), `walgit wal add-pack/annotate-pack`, `walgit repo create`, `walgit repo settings set/clear`, `walgit repo policy set/clear` | Stop the server and every other bucket writer first. Run one CLI command at a time, wait for it to exit, then restart the server. |
| Settings/policy edits while serving | Use the running server's authenticated HTTP API, which shares its store instance. |
| `config check/dump`, local `synth` | May run alongside the server; these do not write the bucket. |
| External tools and embedded callers | No external bucket writes while walgit runs. Embedded callers must share one store instance; two instances in the same process have independent locks. |

Create and Overwrite uploads do not acquire the update lock or HEAD the
destination. Compose copies its parts without this lock; an Update locks only
the final destination HEAD and completion. Conditional deletes share the update
lock. Walgit uses Create for immutable objects, Update for CAS-controlled mutable
objects, and Overwrite only where no CAS is required. Callers must not mix
unconditional writes/deletes with CAS operations on the same mutable key.

The opt-in real OSS contract test is described in
[`crates/walgit-store/tests/contract.rs`](../crates/walgit-store/tests/contract.rs).
Use an isolated test bucket with no running server or other writer. A contract
pass validates operations within one store instance; it does not certify
multi-process CAS safety.
