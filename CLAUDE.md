# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

The **MerryPatch app store for Runtipi** — a custom app store whose purpose is to publish [Bramblekeep](https://github.com/merrypatch/bramblekeep) (self-hosted single-binary workspace, Rust + SQLite, repo at `../hub`) so Runtipi users can install and try it in one click.

It is a *data* repository, not an application: the payload is app definitions under `apps/`, which a Runtipi instance clones. There is no build, no server, no bundler. The only executable code is a validation test suite and a Renovate helper script.

Written work (code, config, commits, docs) is in **English**. `apps/whoami/` is the upstream template's test app, kept as a smoke-test target.

## Commands

Runtime is **Bun** (not Node, despite `@types/node`).

```bash
bun install              # deps
bun test                 # full validation suite
bun test -t "bramblekeep"  # single test / subset by name pattern
bunx tsc --noEmit        # typecheck (no build script exists; tsconfig outDir is unused)
```

## App definition contract

Each directory under `apps/` is one installable app. `__tests__/apps.test.ts` enumerates `apps/*/` at runtime and asserts every app has `config.json`, `docker-compose.json`, `metadata/logo.jpg` and `metadata/description.md`. Adding an app = adding a directory; tests auto-discover it.

**Ship both compose files.** Runtipi reads `docker-compose.json`, validates it, builds the real compose (traefik labels, networks, ports) and *overwrites* `docker-compose.yml` in the installed copy — so the JSON is the source of truth. The `.yml` is kept in sync anyway because the docs point new authors at it and older Runtipi versions fall back to it. This mirrors the official store (250 of 269 apps ship both). Note the two use different conventions: JSON is camelCase with a `services` **array** (`schemaVersion`, `internalPort`, `isMain`, `environment: [{key, value}]`, `volumes: [{hostPath, containerPath}]`); YAML is ordinary compose with a `services` **map** plus `x-runtipi: {is_main, internal_port}` per service and `x-runtipi: {schema_version: 2}` at the root.

### Validation is the contract

Schemas are **not defined in this repo**. They come from `@runtipi/common/schemas` and are [arktype](https://arktype.io) validators (not zod — `zod-validation-error`'s `fromError` is only borrowed to pretty-print arktype's `type.errors`). To learn the real shape of a file, read the schema source in `node_modules/@runtipi/common/dist/schemas/`:

- `app-info.js` — `appInfoSchema`, enumerated `APP_CATEGORIES`, `ARCHITECTURES`, `FIELD_TYPES`, `formFieldSchema`
- `dynamic-compose-ark.js` — `serviceSchemaArk` (every service key Runtipi accepts), `dynamicComposeSchemaArk`
- `dynamic-compose.js` — `MIN_SCHEMA_VERSION = 1`, `CURRENT_SCHEMA_VERSION = 2`

Bumping `@runtipi/common` can break every app at once. Run `bun test` after any dependency change.

Runtipi also hosts the same schemas as JSON Schema — `https://schemas.runtipi.io/v2/app-info.json` and `.../v2/dynamic-compose.json`, referenced from each file's `$schema`. Validating against those with ajv catches drift between the installed `@runtipi/common` and what live Runtipi actually enforces; worth doing when a file changes shape.

`dynamicComposeSchemaYaml` from the same package is **not** the validator for a store's `.yml` — it rejects the root `x-runtipi: {schema_version: 2}` that the docs and the official store both use. Don't validate store YAML against it.

## What Runtipi does at install time

Read from `runtipi/runtipi`, `packages/backend/src/modules/`, in this order (`app-lifecycle/commands/install-app-command.ts` → `commands/command.ts#ensureAppDir`):

1. Copies the app folder into the installed apps dir.
2. `apps/app.helpers.ts#generateEnvFile` writes `app.env` **into the app-data dir** and always sets `APP_PORT`, `APP_URN`, `APP_ID`, `APP_NAME`, `APP_STORE_ID`, `ROOT_FOLDER_HOST`, `APP_DATA_DIR`, plus `APP_DOMAIN` / `APP_HOST` / `APP_PROTOCOL`. Those last three are always populated: the real domain + `https` when exposed, `<subdomain>.<LOCAL_DOMAIN>` + `https` for a local subdomain, otherwise `<internalIp>:<port>` + `http`.
3. `copyDataDir` copies `apps/<app>/data/**` into `${APP_DATA_DIR}/data/` — but **only if that destination does not already exist**. Files ending in `.template` are rendered against the env map.
4. `ensureAppDir` generates the compose file, then runs `chmod -Rf a+rwx` on the app-data dir.
5. `docker compose up`.

Consequences worth remembering:

- **Never bind-mount `${APP_DATA_DIR}` itself** — `app.env` lives there, secrets included. Mount `${APP_DATA_DIR}/data` or a subpath.
- **A non-root image needs a shipped `data/.gitkeep`.** Step 3 is the only thing that creates `${APP_DATA_DIR}/data` before the chmod in step 4; without it Docker creates the bind source itself at `up` time as `root:root 0755`, and a container that dropped privileges cannot write. Bramblekeep runs as uid 10001, hence `apps/bramblekeep/data/.gitkeep`.
- **Empty form fields are omitted, not blanked.** Step 2 only sets a variable when the submitted value is truthy (or a real boolean), so an untouched optional field leaves `${VAR}` unset and Docker interpolates an empty string. Use `${VAR:-default}` in compose whenever the app treats empty and unset differently — Bramblekeep's `PUBLIC_BASE_URL` and `SMTP_FROM` read with `unwrap_or_else`, so an empty value is *not* replaced by their default, while `SETUP_CODE` and the other SMTP vars filter empties themselves.
- `uid` / `gid` in `config.json` are metadata only; nothing chowns from them. A container's user comes from the image or an explicit `user:` in the compose definition.
- Per-app overrides users can apply without touching the store live in `<data>/user-config/<store>/<app>/app.env`.

## Bramblekeep specifics

- Image `ghcr.io/merrypatch/bramblekeep`, tags carry **no `v` prefix** (`0.12.0`, not `v0.12.0`) even though git tags do. Multi-arch amd64 + arm64.
- Listens on 8080 inside the container; host port 8390 in `config.json`. Data (SQLite DB + uploads) in `/data`, runs as uid 10001 / gid 999.
- `PUBLIC_BASE_URL` is derived as `${APP_PROTOCOL:-http}://${APP_DOMAIN}`, the same idiom 36 official apps use — no form field needed.
- **No Watchtower sidecar here**, unlike upstream's `docker-compose.yml`. Bramblekeep's in-app update button drives Watchtower, which would fight Runtipi over the same container; omitting `WATCHTOWER_URL` hides the button and leaves upgrades to Runtipi.
- `COOKIE_SECURE` has to be a form field: it must be on for HTTPS and off for plain-http IP access, and compose interpolation cannot derive a boolean from `APP_PROTOCOL`.

Smoke-testing a change to the compose definition, without a Runtipi instance: copy `apps/bramblekeep/docker-compose.yml` to a scratch dir, add an override with a published port, write a `.env` containing only `APP_DATA_DIR` / `APP_DOMAIN` / `APP_PROTOCOL` (deliberately *not* the `BRAMBLEKEEP_*` vars, to exercise the unset path), `docker compose up -d`, then check `curl localhost:<port>/api/health`, that the DB appears in the bind mount owned by 10001, and `docker logs` for permission errors.

## Renovate → config.json version bumping

`renovate.json` sets `enabledManagers: ["regex"]` — the only thing tracked is container image tags, via two custom managers, one matching `"image": "<dep>:<version>"` in `docker-compose.json` and one matching `image: <dep>:<version>` in `docker-compose.ya?ml`. Both are needed or the two compose files drift. Database images (`mariadb`, `mysql`, `monogdb` [sic], `postgres`, `redis`) are deliberately disabled.

On each upgrade, `postUpgradeTasks` runs `scripts/update-config.ts <packageFile> <newVersion>`, which rewrites the `config.json` sitting next to the changed compose file: `tipi_version += 1`, `version = <newVersion>`, `updated_at = Date.now()`. `tipi_version` is Runtipi's update counter — clients only see an update when it increments, so never hand-edit it backwards. Because both compose files carry the same image, one upgrade runs the script twice and `tipi_version` moves by 2; harmless.

`config.js` at the repo root is Renovate's self-hosted `allowedCommands` allowlist. **Any new `postUpgradeTasks` command must be added there verbatim or Renovate silently skips it.**

CI: `.github/workflows/test.yml` runs `bun install && bun test` on push/PR to `main`; `renovate.yml` runs Renovate nightly at 02:00 UTC.

## Attribution

Commits and pushes for this repo are attributed to **MerryPatch** (`305806950+merrypatch@users.noreply.github.com`), matching the Bramblekeep repo. The identity is set locally in `.git/config`.
