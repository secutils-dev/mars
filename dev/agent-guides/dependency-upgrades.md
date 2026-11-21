# Dependency upgrades

Run commands from the Secutils root unless a stage names another directory.
For changes inside Retrack, read [its agent guide](../../components/retrack/AGENTS.md) first.
Tool apps and `dev/tools/shared/libs/*` pin their own bundle dependencies. Read the [bundle guide](../tools/shared/guides/platform.md#embedded-js-bundles-data-su-bundle).

## Upgrade sequence

For a product-wide upgrade, include Retrack, Secutils, tool apps, and the sibling `secutils-dev` repository.
Find the sibling checkout from the parent directory or its configured remote.
Read its `AGENTS.md` before changes there.
Inspect repository status and existing development data before the first stage.
Preserve unrelated files and database volumes throughout the upgrade.

Inventory manifests, lockfiles, Makefiles, Dockerfiles, CI workflows, Markdown, and tool HTML before selecting versions.
Inspect Docusaurus output and the published `https://secutils.dev/llms.txt` and `https://secutils.dev/api-docs/openapi.json`.
Use the newest compatible releases rather than incompatible major versions.
Record each compatibility hold with the consumer requirement or reproduced error.

Use these stages in order.
For a targeted upgrade, use the applicable stages and preserve their prerequisite order.
Each stage must pass its gate before the next stage starts.
Keep each stage individually committable. Commit only when the user requests it.

Retrack is a Rust path dependency. Its API changes must precede root crate updates.
Node and NPM lockfiles must precede Docker runtime rebuilds.
Kratos server and client changes share one stage because they affect the same auth flows.

| Stage             | Changes                                                                      | Gate                                                                                                                                                                                                                                                           |
|-------------------|------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1. Retrack Rust   | Workspace manifests and lockfile                                             | Retrack dependency guide, stage 1                                                                                                                                                                                                                              |
| 2. Retrack Node   | `.nvmrc`, declared Node engines, and `@types/node`                           | Retrack dependency guide, stage 2                                                                                                                                                                                                                              |
| 3. Retrack NPM    | Root and scraper packages. Record the selected Playwright version.           | Retrack dependency guide, stage 3                                                                                                                                                                                                                              |
| 4. Retrack Docker | Base images, UPX, and Camoufox triple                                        | Retrack dependency guide, stage 4                                                                                                                                                                                                                              |
| 5. Kratos         | Compose `oryd/kratos` image and Web UI `@ory/kratos-client-fetch` together   | `make e2e-up BUILD=1 && make e2e-test ARGS="tests/registration.spec.ts"`                                                                                                                                                                                       |
| 6. Root Rust      | Retrack pointer first, then root/JWT-tools/perf manifests and `cargo update` | `cargo +nightly fmt --all -- --check && cargo clippy --workspace --all-targets -- -D warnings && cargo sqlx prepare --check && cargo test`                                                                                                                     |
| 7. Root Node      | Same major as stage 2. Refresh root, Web UI, docs, and E2E lockfiles.        | Web UI lint/test/build, docs typecheck/test/build, and E2E lint                                                                                                                                                                                                |
| 8. Web UI NPM     | EUI, React, Parcel, and exact `playwright-core` pin                          | `npm --prefix components/secutils-webui run lint && npm --prefix components/secutils-webui run test && npm --prefix components/secutils-webui run build && npm --prefix components/secutils-webui run analyze`. For an EUI bump, also `make docs-screenshots`. |
| 9. Docs NPM       | Docusaurus and Markdown pipeline                                             | `npm --prefix components/secutils-docs run typecheck && npm --prefix components/secutils-docs test && npm --prefix components/secutils-docs run build && npm --prefix components/secutils-docs run check-llms`                                                 |
| 10. Root Docker   | API, Web UI, and docs base images/UPX                                        | `make docker-api && make docker-webui && make docker-docs && make e2e-up BUILD=1`, then HTTP smoke checks for `/api/status`, `/`, and `/docs/`                                                                                                                 |
| 11. E2E harness   | Playwright, ESLint, TypeScript-ESLint, and globals                           | Install Chromium from `e2e/`, then `make e2e-standalone-test && make e2e-test && make docs-screenshots`                                                                                                                                                        |
| 12. Security tools | CDN libraries and the embedded JOSE bundle | Tool guide checks, JOSE unit tests, and affected browser specs |
| 13. PDF/Forecast | App bundles, build tooling, and transitive security fixes | Reproducible installs, typechecks, builds, audits, and real PDF extraction in Chromium, Firefox, and WebKit |
| 14. MARS | Editor, renderers, sanitizer, and Transformers.js consumers | MARS guide checks, all model/dtype evaluations, affected browser suites, and real GPU smart-search checks |
| 15. Other bundles | The shared 3MF importer and every discovered bundle | Importer tests, `make tools-bundles`, `make tools-verify`, and `make tools-check` with the sibling promo path |
| 16. Sibling product | Rust → Node → Web UI → Docker → E2E | The sibling guide's stage gates, full-stack E2E, and agent-skill digests against local Secutils sources |
| 17. Final synchronization | Generated artifacts, requested commits/pushes, and CI | Reviewed diff, final fetch/rebase in each repository, exact Retrack pointer, and passing latest CI |

Read Ory release notes for registration, login, and recovery schema changes at stage 5.
Read EUI, React, and Parcel release notes at stage 8.
Read [docs output](docs-output.md) at stage 9 and for docs-serving changes at stage 10.
Read [E2E](e2e.md) before stack verification and [screenshots](docs-screenshots.md) before screenshot regeneration.
The final Playwright gate requires all three full suites. Partial suites miss network and DNS regressions.

### Final record and optional publication

Record an ordered table of changes, commands, results, compatibility holds, and final repository SHAs.
Separate pre-existing failures from regressions through baseline comparisons.
Record remaining advisories instead of reporting a zero audit count for every package.
For step-by-step actions, read [the product-wide upgrade procedure](product-wide-upgrade.md).

Image publication, tool publication, and live restarts are separate operations.
When the user requests them, use [image publishing](image-publishing.md) and the tool deployment guide.
Ask whether live restarts include shared PostgreSQL and whether publication includes responder-backed tools.
Complete the upgrade record even when publication remains outside the requested scope.

### Retrack pointer: stage 6

Use the verified Retrack revision from stages 1–4 before `cargo update` in the root.
When those stages produce commits, use their final exact SHA rather than an unspecified remote tip.
When only a pointer upgrade is requested, inspect the target revision and its API compatibility first.
Include `components/retrack` in the requested parent commit so CI uses the verified revision.
When commits are requested for the upgrade sequence, update the parent pointer only in stage 6.
Without requested commits, report the uncommitted Retrack changes and pending parent pointer update.

### Playwright coordination

Inspect the actual pins in scraper/Web UI `playwright-core`, E2E `@playwright/test`, and Camoufox's PyPI `playwright`.
Use exact pins. Keep the scraper on Camoufox's minor: its Playwright server rejects other minors with HTTP 428.
When the `camoufox` wrapper caps Playwright, Web UI and E2E may use a newer minor. Otherwise keep all four on one minor.
Use the selected patch across consumers where compatible. Record any required patch difference.
Update consumers in stages 3, 4, 8, and 11 before completion.
Run the standalone suite first after the coordinated bump.
Its codegen test catches boilerplate changes that affect the Web UI transformer.

## Rust

- After a `deno_core` bump, review the `can_access_deno_apis` snapshot against the changed API surface.
  Its transitive exact `deno_error` pin can require a matching workspace patch.
  If `cargo outdated` cannot resolve that pin, query the crates.io index.
- After SQLx macro query changes or a SQLx bump, regenerate `.sqlx/` against development Postgres:

  ```bash
  docker compose -f dev/docker/docker-compose.yml up -d secutils_db
  cargo sqlx prepare
  cargo sqlx prepare --check
  ```

  Runtime queries do not require offline-cache updates.
- Performance deltas are advisory. A failed harness run still requires investigation.
  Read [performance](performance.md) before analysis or history changes.

## Node and Web UI compatibility

Read versions from manifests instead of copying version inventories into this guide.

- Match `.nvmrc` to declared top-level `engines.node` and `@types/node` majors.
  Preserve `targets.retrack.engines.node` in Web UI `package.json` during toolchain upgrades.
  That key controls Parcel output compatibility, not the host Node version.
- For scraper worker changes during Node upgrades, read Retrack's dependency guide for sandbox requirements.
- Before an ESLint major bump, inspect plugin peer ranges per package.
  Web UI's React and EUI plugins constrain its ESLint major independently of E2E and scraper.
  With `preserve-caught-error`, rethrow with `{ cause: err }`.
- Before a TypeScript major bump, establish compatibility with Docusaurus and EUI consumers.
  For TypeScript 6, migrate deprecated `baseUrl`/`paths` imports to direct relative imports.
- For debounced callbacks under React Hooks 7, use `useMemo(() => debounce(...), [])`.
- Preserve Parcel's `@parcel/resolver-default.packageExports: true` for exports-only subpaths.
  `@peculiar/x509` v2 also needs `reflect-metadata`. Resolve both requirements before that major bump.
- Keep a CommonJS-compatible `http-proxy-middleware` for `.proxyrc.ts`. Version 4 is ESM-only.

### EUI icons

Read the full EUI changelog for token, required-prop, and accessible-label changes.
Icon removals can pass lint, unit tests, builds, and bundle analysis.
The imported `IconType` can degrade to `string` through an unpublished EUI path.

1. Compare `typeToPathMap` in both versions' `es/components/icon/icon_map.js`.
2. Search Web UI `src/` for every quoted occurrence of each removed name.
   Include helpers, ternaries, and empty states, then classify non-icon homonyms.
3. For a removed alias, use its surviving canonical target when available.
4. For other replacements, use monochrome glyphs that obey `currentColor`.
   `logo*` and `*App` glyphs carry fixed fills and can fail on dark backgrounds.
5. If the replacement changes the visual metaphor, obtain the user's choice from a screenshot comparison.
   Use `npm pack` of the previous version for the old asset.
   Read asset paths from their icon-map loaders, such as `securityApp` → `assets/app_security.js`.
   Show the real background and preserve each candidate's native fills.
6. Regenerate docs screenshots during stage 8.

Completion requires an accounted-for replacement for every used removed icon.
The `no-deprecated-icon-aliases` rule covers JSX aliases, not removals or helper return values.

## Docker

Re-pin root image digests with `./dev/scripts/docker-pin-digests.sh`, including unchanged rolling tags.
Use Retrack's guides for scraper workspace layout and Camoufox builds.
If the Rust image build fails with `No space left on device`, run `make docker-prune` before retry.
That target deletes dangling images and BuildKit cache.
Completion requires successful builds and smoke checks against the rebuilt Compose services.

For Buildx CI failures, inspect the action's release index before changing the selected version.
A published release can precede its entry in that index.
The supported Git source URL bypasses the index.
Preserve its release tag so that Compose receives a semantic version rather than a bare commit hash.
Run the resulting binary's version command and an actual Compose build before accepting this workaround.
