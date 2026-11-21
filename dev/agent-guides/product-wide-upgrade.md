# Product-wide dependency upgrade procedure

Use this procedure for Retrack, Secutils, tool apps, and the sibling `secutils-dev` repository.
Run commands from the Secutils root unless a step names another repository.
Use [dependency upgrades](dependency-upgrades.md) for the required stage gates and compatibility rules.
Complete each stage before the next stage starts.

## Before the upgrade

1. Inspect Git status in Secutils, its Retrack submodule, and the sibling `secutils-dev` checkout.
2. Read each repository's `AGENTS.md` and the applicable task guides.
3. Record unrelated changes and existing development stacks that must remain intact.
4. Inventory Cargo manifests, NPM manifests, lockfiles, runtime declarations, Dockerfiles, Makefiles, and CI workflows.
5. Inspect source code, Markdown, tool HTML, Docusaurus output, and the sibling promo site.
6. Inspect `https://secutils.dev/llms.txt` and `https://secutils.dev/api-docs/openapi.json` for published contracts and generated content.
7. Research current releases and breaking changes for each dependency group.
8. Select the newest compatible versions across their actual consumers.
9. Record each compatibility hold with its dependency constraint or reproduced failure.

## Upgrade sequence

### 1. Retrack Rust

- Update the Retrack workspace, `retrack-types`, and applicable benchmark manifests.
- Refresh `Cargo.lock` with `cargo update`.
- Migrate affected API consumers before introducing alternative methods.
- After SQLx macro changes, regenerate the offline query metadata against development PostgreSQL.
- Run the Retrack Rust stage gate before changes to its Node packages.

### 2. Retrack Node

- Select the Node runtime in Retrack's `.nvmrc`.
- Activate that runtime in the shell before package installation or builds.
- Align declared Node engines and `@types/node` with the selected major version.
- Keep the scraper's target runtime distinct from host build-tool requirements.

### 3. Retrack NPM

- Update root tooling and scraper dependencies from the Retrack workspace root.
- Refresh the shared NPM lockfile.
- Select an exact scraper `playwright-core` version compatible with Camoufox's Python Playwright package.
- Inspect worker isolation and sandbox behavior after Node or Playwright changes.

### 4. Retrack Docker and Camoufox

- Update the API and scraper base images after their manifests and lockfiles are ready.
- Refresh image digests and applicable UPX pins.
- Select the Python runtime, Camoufox wrapper, Playwright package, and browser build as a compatibility set.
- Keep scraper and Camoufox Playwright minor versions aligned.
- Build the images with their existing dependency and browser caches.
- Run package consistency checks and real Chromium/Gecko extraction before accepting the images.

### 5. Kratos

- Review Ory release notes for authentication schema and flow changes.
- Update the Compose Kratos image and Web UI client package together where an upgrade is compatible.
- Run registration, sign-in, and recovery checks against the selected server/client combination.

### 6. Secutils Rust, JWT tools, and Retrack pointer

- Update the parent submodule pointer to the exact Retrack revision from the completed Retrack stages.
- Update Secutils, JWT-tools, and applicable benchmark Cargo manifests.
- Refresh the root Cargo lockfile.
- Migrate affected consumers and review Deno runtime snapshots after runtime changes.
- Regenerate SQLx metadata where macro queries or SQLx changes require it.
- Run the root Rust stage gate with `cargo test`, rather than `cargo test --workspace`.

### 7. Secutils Node and root tooling

- Align the root Node major with the selected Retrack runtime where compatible.
- Activate that Node version before installation, builds, or deployment scripts.
- Update root tooling and refresh the affected root, Web UI, docs, and E2E lockfiles.
- Keep package-specific compiler and lint versions within their consumer peer requirements.

### 8. Web UI packages and screenshots

- Review EUI, React, Router, Parcel, and lint-plugin compatibility before package updates.
- Update compatible packages and the exact Web UI Playwright pin.
- Inspect EUI icon removals and migrate every affected application reference.
- Build the Web UI and review its bundle changes.
- Regenerate docs screenshots after EUI changes.
- Review intentional visual changes separately from raster differences.
- Preserve the existing screenshot tolerance during that review.

### 9. Docusaurus and generated Markdown

- Update docs packages and the Markdown build pipeline.
- Inspect actual consumers before adding overrides for blocked transitive security fixes.
- Build the docs and review generated Markdown, LLMS files, links, and content types.
- After SSG worker dependency changes, exercise both inline and pooled builds.

### 10. Secutils Docker and local PostgreSQL

- Update API, Web UI, and docs base images after their dependency stages.
- Refresh base-image digests, including unchanged rolling tags.
- Update PostgreSQL pins consistently across Compose and CI.
- Refresh Buildx and BuildKit configuration where compatible.
- Rebuild the actual Compose services without deleting existing database volumes.
- Inspect API/database status, Web UI assets, docs, LLMS content, and MIME handling.

### 11. E2E harness

- Update Playwright, its browser binaries, and compatible harness tooling.
- Coordinate the final Playwright pins across scraper, Camoufox, Web UI, and E2E consumers.
- Run standalone tests first, then application E2E, then docs screenshot checks.
- Compare additional failures against the original dependency baseline before treating them as regressions.

### 12. Security-tool libraries and JOSE

- Inspect CDN dependencies in tool HTML, including private tools.
- Select immutable release URLs and integrity hashes from known-good release assets.
- After a library changes its distribution format, migrate its consumers through the existing bundle mechanism.
- Rebuild JOSE and exercise JWT signing, verification, and malformed inputs.
- Run the affected certificate and SAML browser flows against the assembled pages.

### 13. PDF Extractor and Forecast

- Update each app's isolated build package and lockfile.
- Inspect browser adapters and stubs before accepting native-package or WASM major changes.
- Inspect all consumers before overriding vulnerable transitive dependencies.
- Rebuild both bundles and exercise real PDF extraction and Forecast calculations.
- Exercise PDF-to-MARS handoff in the supported browser engines.

### 14. MARS

- Update editor, Markdown renderer, sanitizer, and build dependencies.
- Keep Mermaid, highlighting, and math assets consistent between the viewer and exports.
- Update Transformers.js in both the application loader and evaluation package.
- Run every configured model/dtype evaluation without changing the quality thresholds to hide regressions.
- Exercise editing, rendering, imports, sharing, exports, and real GPU smart search.
- Diagnose browser harness failures separately from application failures.

### 15. Remaining bundles and shared tool checks

- Inspect the shared 3MF importer for compatible dependency updates.
- Run its importer checks even when its versions remain current.
- Run `make tools-bundles` to assemble every discovered bundle.
- Run `make tools-verify` on the assembled tool sources.
- Run `make tools-check` with `SECUTILS_TOOLS_PROMO_HOME_INDEX` pointed at the sibling promo source.

### 16. Sibling `secutils-dev`

- Run the sibling guide's sequence: Rust → Node → Web UI → Docker → E2E.
- Refresh Cargo and NPM lockfiles in their corresponding stages.
- Align runtime declarations and Docker pins with the compatible product-wide choices.
- Regenerate its agent-skills index against the local Secutils source files.
- Build the promo API and site images after their dependency stages.
- Exercise the sibling stack without removing the main Secutils development data.

### 17. Final review, requested commits, and CI

- Review source changes, lockfiles, generated artifacts, runtime pins, and remaining advisories.
- Keep each completed stage individually committable.
- When commits are requested, use conventional commits in each repository.
- Before each requested push, fetch the target remote and rebase unpublished commits onto its latest branch.
- After a Retrack rebase, record its final SHA in the parent submodule pointer.
- Wait for the latest applicable CI workflows rather than accepting an older successful run.
- After CI failures, diagnose the failed step before changing dependencies or reducing checks.
- For Buildx release-index lag, use the supported release-tag Git source rather than a bare commit reference.

## Publish and restart only when requested

Use [image publishing](image-publishing.md) for registry configuration, build routing, backup requirements, and rollout checks.

1. Establish dev/prod scope, PostgreSQL restart permission, and tool publication permission.
2. Inspect live workloads, service dependencies, image tags, platforms, running digests, and persistent storage.
3. Identify shared services before the first restart.
4. Save rollback image references and current responder configuration.
5. Before PostgreSQL restarts, create fresh backups of globals and every database.
6. Publish development images through the existing Makefile targets.
7. Record the published index and platform digests.
8. Update dev dependencies first: PostgreSQL, Kratos, Camoufox, scraper, Retrack API, then Secutils API and static sites.
9. Require healthy replacement workloads and actual HTTP/browser behavior before production promotion.
10. Preserve unique rollback tags before replacing production tags.
11. Promote the exact development-tested digests to the configured production tags.
12. Update production PostgreSQL and applications in the same dependency order.
13. When tool publication is requested, run `make deploy-tools ARGS="<selected tools>"` after the API rollout.
14. Resolve missing responder IDs with the user instead of creating unrequested responders.
15. Exercise the deployed tools, companion skills, discovery indexes, and generated docs against their live hosts.
16. Use isolated MARS test scopes and close those scopes after the browser checks.
17. Inspect final image digests, application status, logs, database versions, and preserved storage bindings.
18. Record live image pins for any separately maintained infrastructure manifests.
19. Remove only temporary resources created for this upgrade.

## Handoff

Record the selected versions, compatibility holds, repository SHAs, image digests, and remaining work.
Explain the actions and their order rather than using test totals as the workflow.
Distinguish dependency changes, image publication, tool publication, live restarts, and infrastructure-manifest updates.
Keep private configuration, credentials, rollback inventory, and deployment logs in ignored local files.
