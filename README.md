# docverse-cloudflare-deployments

Cloudflare Worker deployment configurations for [Rubin Observatory's Docverse](https://github.com/lsst-sqre/docverse) service.

The Worker source code lives in the [lsst-sqre/docverse](https://github.com/lsst-sqre/docverse) monorepo under `cloudflare-worker/`.
This repository provides per-environment Wrangler configuration (`wrangler.toml`) and the root `package.json` needed to run `npx wrangler deploy`.
At deploy time the `docverse deploy-worker` CLI packs the worker source into this repo's `worker/` directory (gitignored) and invokes Wrangler.

For architecture details see [SQR-112](https://sqr-112.lsst.io).

## Repository structure

```
wrangler.toml   # Wrangler config: base settings + per-environment bindings
package.json    # Root devDependency on wrangler
AGENTS.md       # Claude Code context
CLAUDE.md       # Points to AGENTS.md
```

The `worker/` directory is populated at deploy time and is not checked in.

## Naming conventions

Environments and resources follow the pattern `{tier}-{org}`:

| Resource | Pattern | Example |
|----------|---------|---------|
| Wrangler environment | `{tier}-{org}` | `dev-jsc-test-20260409` |
| Deployed worker name | `docverse-{tier}-{org}` | `docverse-dev-jsc-test-20260409` |
| KV namespace | `docverse-{tier}-{org}-editions` | `docverse-dev-jsc-test-20260409-editions` |
| R2 bucket | `docverse-{tier}-{org}-builds` | `docverse-dev-jsc-test-20260409-builds` |

## Creating Cloudflare resources

Before deploying a new environment you need to create its KV namespace and R2 bucket.

### KV namespace

```bash
npx wrangler kv namespace create "docverse-{tier}-{org}-editions"
```

Copy the returned namespace ID into the environment's `[[env.*.kv_namespaces]]` block in `wrangler.toml`, replacing `PLACEHOLDER_KV_NAMESPACE_ID`.

### R2 bucket

```bash
npx wrangler r2 bucket create "docverse-{tier}-{org}-builds"
```

The bucket name in `wrangler.toml` must match exactly.

## Deploying

Deployment is handled by the `docverse deploy-worker` CLI from the [`docverse` client package](https://github.com/lsst-sqre/docverse/tree/main/client):

```bash
docverse deploy-worker \
  --docverse-repo /path/to/docverse \
  --deployments-repo /path/to/docverse-cloudflare-deployments \
  --env dev-jsc-test-20260409
```

Add `--dry-run` to build the bundle without deploying.

The CLI:
1. Runs `npm pack` in the docverse `cloudflare-worker/` directory
2. Extracts the tarball into `worker/` in this repo
3. Copies `package-lock.json` in alongside it — `npm pack` always excludes lockfiles
4. Runs `npm ci --omit=dev` in `worker/` to install runtime dependencies
5. Runs `npx wrangler deploy --env <env>` from this repo root

## URL routing

Each environment sets a `URL_SCHEME` variable that controls how the worker routes requests:

- **`path-prefix`** — docs are served from the domain root (or an optional `PATH_PREFIX`)
- **`subdomain`** — each product is a subdomain

The current PoC environments use `path-prefix` with no `PATH_PREFIX`, serving docs directly from the workers.dev URL root.
