# Image publishing

## Configuration and discovery

Read `deploy.env` for private registry, SSH, tags, and per-image routing configuration.
Use `deploy.env.example` for the supported keys.
The repository ignores `deploy.env` and `.deploy/`.
Keep private host addresses and timing reports in these local files.

The `deploy-*` targets build and push images.
Deployment restarts require a separate user request.
Responder-backed tools require separate publication through `make deploy-tools`.
Read [the tool guide](../tools/AGENTS.md) before that publication.

## Procedure

1. Read the applicable dependency guide before a Docker build.
2. Run `make deploy-health` before a remote build.
3. Run the selected target with `DRY_RUN=1` to inspect its route and tags.
4. Run the selected target without `DRY_RUN` to build and push.
5. Run `make deploy-stats` to inspect the duration and result.
6. Complete the image smoke checks from the applicable dependency guide.

For a local override, use `DEPLOY_BUILD_ROUTE=local`:

```bash
make deploy-dev-api DEPLOY_BUILD_ROUTE=local
```

For a remote override, use `DEPLOY_BUILD_ROUTE=remote`:

```bash
make deploy-camoufox DEPLOY_BUILD_ROUTE=remote
```

The automatic route sends images in `DEPLOY_REMOTE_IMAGES` to the SSH host.
Other images use the current local Docker builder.
An empty remote image list preserves local builds for every image.

## SSH builder

The local machine requires Bash, jq, Git, tar, and Docker Buildx.
The SSH host requires Bash, tar, Docker Engine, Buildx, and passwordless sudo.
SSH uses the local SSH configuration and agent.
Run `docker login` for the configured registry on the local machine.

The script uploads tracked build-context files, including current edits and initialized submodule files.
Untracked files require a local build or explicit Git staging before upload.
BuildKit applies the Dockerfile's ignore rules after upload.

The script uses Docker Buildx's `docker-container` driver on the SSH host.
This driver keeps its cache in a dedicated Docker volume.
The script preserves the named builder between builds.
The script stops the builder after each run and preserves its cache volume.
Run one publish at a time for each remote builder.
Each run uses a temporary source directory and registry-only credential file.
The script removes both files after the run.

Resource limits apply when the script first creates the builder.
To change these limits, remove that named builder before the next run.
Use `docker buildx rm` with the same remote Docker and `BUILDX_CONFIG` configuration.
The script keeps builder metadata under `DEPLOY_REMOTE_DIR/buildx`.
For builder-specific disk cleanup, run `docker buildx prune --builder <name>` through the same remote configuration.
Avoid global Docker pruning on a shared host.

If SSH fails during cleanup, remove the reported run directory after the host recovers.
Registry credentials can remain in that directory until cleanup succeeds.

Docker also supports [SSH contexts](https://docs.docker.com/engine/security/protect-access/).
Contexts require direct Docker socket access for the SSH user.
This script supports sudo-only hosts without changes to Docker socket permissions.
The [container driver](https://docs.docker.com/build/builders/drivers/docker-container/) documents the resource limits and cache behavior.

## Capacity and timing

The health command reports free disk space, available RAM, swap, CPU count, architecture, and Docker storage usage.
The remote disk threshold is configurable.
Remote API builds require at least 4 GiB available RAM.
Available disk space alone does not establish sufficient build capacity.

Small hosts with fast network access suit browser-image builds.
Rust builds benefit from multiple CPUs, RAM, and an existing compilation cache.
Select the route from measurements on the actual machines.

Each publish records total duration, context-upload duration, build-plus-push duration, and BuildKit's reported push duration.
Total duration includes preflight and builder creation.
The report includes remote capacity before and after the build, plus the initial cache inventory.
Push duration includes layer and manifest operations.
Concurrent BuildKit operations can overlap.
The durations do not measure network throughput directly.

Compare the same image, source revision, architecture, and cache state.
Distinguish fresh builds from cached builds.
Use existing publish requests for measurements because repeated pushes replace registry tags.
Read `.deploy/history.jsonl` for raw records and `.deploy/*.log` for BuildKit output.

## Live rollout

1. Establish the requested environments and restart scope.
   Ask whether the scope includes shared PostgreSQL and responder-backed tools.
   Single-replica PostgreSQL restarts cause a brief outage for every consumer of that database.
2. Inspect the current Kubernetes context and the exact workloads in the requested namespaces.
   Record container names, image references, running digests, replicas, storage, and health probes under `.deploy/`.
   Identify services that both environments share before the first restart.
3. Preserve rollback references before replacing mutable registry tags.
   Record the old running digests, not only the configured tags.
   For tools, save each current responder's configuration before replacing its `settings`.
4. Before a requested PostgreSQL restart, create fresh backups of globals and every database.
   Use the existing backup credentials and storage without changing the scheduled job.
   Require successful dumps, readable dump catalogs, a stored snapshot, and a passing integrity check.
   Preserve existing snapshots and all database volumes.
5. Publish the selected development images with the procedure above.
   Run one publish at a time per remote builder.
   Record the published index digest and the digest for the selected platform.
6. Update development dependencies first: PostgreSQL, auth, Camoufox, scraper, Retrack API, then Secutils API and static sites.
   Wait for each rollout before the next dependent service.
   Keep the existing PostgreSQL distribution and major version for a patch update.
   A shared scraper rollout also affects production consumers.
7. Inspect the actual HTTP responses, database version, startup logs, and running image digests.
   Kubernetes readiness alone is insufficient for workloads without HTTP probes.
   Target the ready replacement pod for port-forward checks.
   A service port-forward can select an old terminating pod after a successful rollout.
   Require operational API/database status, auth health, scraper extraction, and served UI/docs assets.
8. Promote the exact development-tested image digests to the configured production tags.
   `docker buildx imagetools create --tag <production-reference> <repository>@<tested-index-digest>` preserves the tested image.
   Rebuilding production is acceptable only with another digest comparison and complete smoke checks.
9. Update production PostgreSQL and applications in the same dependency order.
   Preserve unrelated infrastructure and the saved rollback inventory.
10. When requested, publish the changed tools and companion skills after the API rollout.
    Each tool publication also refreshes discovery aggregates.
    Run the affected specs against the live host, including MARS and real PDF extraction.
    Use isolated MARS test scopes with the operator credential in the Node process only.
11. Record backups, image digests, rollout outcomes, HTTP checks, browser results, and any persistent manifest drift.
    Keep private inventory, logs, and responder backups under `.deploy/`.

Completion requires ready replacement pods, matching platform digests, successful live checks, and preserved storage.
If a gate fails, stop before the next dependent rollout.
Use saved image digests for application rollback.
Before a database downgrade or restore, assess migration compatibility and obtain approval for any data-destructive operation.
