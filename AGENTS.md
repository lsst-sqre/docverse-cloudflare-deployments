# docverse-cloudflare-deployments

## Overview

docverse-cloudflare-deployments is a repository that provides specific Cloudflare deployments for Rubin Observatory's Docverse service. It provides configurations for the both the Docverse deployments in production (roundtable.lsst.cloud/docverse/api) and development (roundtable-dev.lsst.cloud/docverse/api) environments.
These deployments layer on the core Cloudflare configuration in Docverse's [cloudflare-worker](https://github.com/lsst-sqre/docverse/tree/main/cloudflare-worker) directory.

### Related information

For more background on Docverse's architecture, see [SQR-112](https://sqr-112.lsst.io). For the Docverse codebase see [lsst-sqre/docverse](https://github.com/lsst-sqre/docverse).

## File structure

- `wrangler.toml` — Wrangler configuration with base settings and per-environment bindings.
- `package.json` — Root `devDependency` on `wrangler` so `npx wrangler deploy` works.
- `worker/` — Populated at deploy time by `docverse deploy-worker`; gitignored.

## wrangler.toml conventions

### Base config

The top-level section defines the worker name (`docverse`), the entry point (`worker/src/index.ts`), compatibility date, and flags. There are **no top-level bindings** — all bindings live in environment blocks. Testing with local bindings happens in the docverse source repo's own `wrangler.toml`.

### Environment naming

Environments follow the `{tier}-{org}` pattern:

- `tier` — deployment tier (`dev`, `production`)
- `org` — the Docverse organization slug (e.g., `jsc-test-20260409`)

Wrangler automatically prepends the worker name, so `env.dev-jsc-test-20260409` deploys as `docverse-dev-jsc-test-20260409`.

### Required bindings per environment

Each environment must define:

| Binding | Type | Description |
|---------|------|-------------|
| `EDITIONS_KV` | KV namespace | Maps edition slugs to build IDs |
| `BUILDS_R2` | R2 bucket | Stores build artifacts (HTML, assets) |
| `URL_SCHEME` | Variable | `"path-prefix"` or `"subdomain"` |
| `PATH_PREFIX` | Variable (optional) | Root path prefix for path-prefix routing |

## Resource naming

Cloudflare resources follow predictable names derived from the environment:

- KV namespace: `docverse-{tier}-{org}-editions`
- R2 bucket: `docverse-{tier}-{org}-builds`

## Deployment process

Deployment is handled by the `docverse deploy-worker` CLI, shipped by the [`docverse` client package](https://github.com/lsst-sqre/docverse/tree/main/client):

1. `npm pack` the `cloudflare-worker/` directory in the docverse repo
2. Extract the tarball into `worker/` in this repo
3. Copy `package-lock.json` in alongside it — `npm pack` always excludes lockfiles
4. `npm ci --omit=dev` in `worker/` to install runtime dependencies
5. `npx wrangler deploy --env <env>` from this repo root

The `worker/` directory and `node_modules/` are gitignored.
