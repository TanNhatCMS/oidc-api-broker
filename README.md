# OIDC API Broker

A Cloudflare Worker that authenticates GitHub Actions OIDC tokens and brokers requests to upstream APIs with injected API keys. CI workflows call APIs without storing long-lived secrets in GitHub.

**Multi-provider**: routes to **Anthropic** or **Xiaomi MiMo** based on the `model` field in the request body.

> **Forked from [aaif-goose/goose/oidc-proxy](https://github.com/aaif-goose/goose/tree/main/oidc-proxy)** — see [Attribution](#attribution) below.

## Architecture

```
GitHub Actions ──(OIDC token)──► Worker ──(validate JWT, inject API key)──► Anthropic API
                                                          │
                                                          ├─ model: claude-* → Anthropic
                                                          └─ model: mimo-*  → Xiaomi MiMo
```

1. GitHub Actions mints an OIDC token with a configured audience
2. Workflow sends requests to this proxy, passing the OIDC token as the API key
3. Worker validates JWT against GitHub's JWKS, checks issuer / audience / age / repo
4. Reads `model` from request body → routes to matching upstream
5. Valid requests forwarded with the real API key injected

## Supported Providers

### Anthropic (default)

| Model examples | Endpoint |
|---|---|
| `claude-sonnet-4-20250514`, `claude-3.5-haiku`, etc. | `https://api.anthropic.com` |

### Xiaomi MiMo

| Model | Notes |
|---|---|
| `mimo-v2.5-pro` | Latest flagship |
| `mimo-v2.5` | Standard |
| `mimo-v2.5-pro-claude` | Claude-compatible |
| `mimo-v2-pro` | ⚠️ Deprecated June 30, 2026 — auto-routes to V2.5 |
| `mimo-v2-omni` | ⚠️ Deprecated June 30, 2026 — auto-routes to V2.5 |

Endpoints:
- **Primary**: `https://api.xiaomimimo.com/anthropic/v1/messages`
- **Token plan (SGP)**: `https://token-plan-sgp.xiaomimimo.com/anthropic/v1/messages`

Auth: `api-key` header (raw key) or `Authorization: Bearer <key>`

## Setup

```bash
npm install
```

## Configuration

Edit `wrangler.toml`:

### Core

| Variable | Description |
|---|---|
| `OIDC_ISSUER` | `https://token.actions.githubusercontent.com` |
| `OIDC_AUDIENCE` | Audience your workflow requests (e.g. `oidc-api-broker`) |
| `MAX_TOKEN_AGE_SECONDS` | Upper bound on `iat` age (default `1200` = 20 min). Applied *in addition to* IdP's `exp` claim. |
| `MAX_REQUESTS_PER_TOKEN` | Max requests per OIDC token (default `200`) |
| `RATE_LIMIT_PER_SECOND` | Max requests/sec per token (default `2`) |
| `ALLOWED_REPOS` | *(optional)* Comma-separated `owner/repo` list |
| `ALLOWED_REFS` | *(optional)* Comma-separated allowed refs |

### Anthropic Upstream (default)

| Variable | Description | Default |
|---|---|---|
| `UPSTREAM_URL` | Anthropic API base URL | `https://api.anthropic.com` |
| `UPSTREAM_AUTH_HEADER` | Auth header name | `x-api-key` |
| `UPSTREAM_AUTH_PREFIX` | *(optional)* Prefix before key | *(unset — raw key)* |

### Xiaomi MiMo Upstream

| Variable | Description | Default |
|---|---|---|
| `MIMO_UPSTREAM_URL` | MiMo API base URL | `https://api.xiaomimimo.com/anthropic` |
| `MIMO_UPSTREAM_AUTH_HEADER` | Auth header name | `api-key` |
| `MIMO_UPSTREAM_AUTH_PREFIX` | *(optional)* Prefix before key | *(unset)* |
| `MIMO_MODELS` | *(optional)* Override built-in model list | `mimo-v2.5-pro,mimo-v2.5,mimo-v2-pro,mimo-v2-omni,mimo-v2.5-pro-claude` |

### CORS

| Variable | Description |
|---|---|
| `CORS_ORIGIN` | *(optional)* Allowed CORS origin |
| `CORS_EXTRA_HEADERS` | *(optional)* Additional CORS allowed headers |

### Secrets

```bash
# Anthropic API key (required for Anthropic models)
npx wrangler secret put UPSTREAM_API_KEY

# Xiaomi MiMo API key (required for MiMo models; falls back to UPSTREAM_API_KEY if unset)
npx wrangler secret put MIMO_UPSTREAM_API_KEY
```

### Example `wrangler.toml` — Anthropic only

```toml
UPSTREAM_URL = "https://api.anthropic.com"
UPSTREAM_AUTH_HEADER = "x-api-key"
CORS_EXTRA_HEADERS = "anthropic-version"
```

### Example `wrangler.toml` — Anthropic + Xiaomi MiMo

```toml
# Anthropic (default)
UPSTREAM_URL = "https://api.anthropic.com"
UPSTREAM_AUTH_HEADER = "x-api-key"
CORS_EXTRA_HEADERS = "anthropic-version"

# Xiaomi MiMo (mimo-* models)
MIMO_UPSTREAM_URL = "https://api.xiaomimimo.com/anthropic"
MIMO_UPSTREAM_AUTH_HEADER = "api-key"
```

### Example `wrangler.toml` — Xiaomi MiMo only (token plan)

```toml
# Point default upstream to MiMo — all requests go there
UPSTREAM_URL = "https://token-plan-sgp.xiaomimimo.com/anthropic"
UPSTREAM_AUTH_HEADER = "api-key"

# No need for MIMO_* vars when using MiMo as the only upstream
```

## Usage in GitHub Actions

```yaml
permissions:
  id-token: write

steps:
  - name: Get OIDC token
    id: oidc
    uses: actions/github-script@v7
    with:
      script: |
        const token = await core.getIDToken('goose-oidc-api-broker');
        core.setOutput('token', token);
        core.setSecret(token);

  - name: Call Anthropic through proxy
    env:
      ANTHROPIC_BASE_URL: https://oidc-api-broker.your-subdomain.workers.dev
      ANTHROPIC_API_KEY: ${{ steps.oidc.outputs.token }}
    run: goose run --recipe my-recipe.yaml

  - name: Call MiMo through proxy
    env:
      MIMO_BASE_URL: https://oidc-api-broker.your-subdomain.workers.dev
      MIMO_API_KEY: ${{ steps.oidc.outputs.token }}
    run: |
      curl "$MIMO_BASE_URL/v1/messages" \
        -H "api-key: $MIMO_API_KEY" \
        -H "Content-Type: application/json" \
        -d '{"model":"mimo-v2.5-pro","max_tokens":1024,"messages":[{"role":"user","content":"Hello"}]}'
```

## Routing Logic

```
POST /v1/messages → read body.model
  ├─ model in MIMO_MODELS? → MIMO_UPSTREAM_URL
  └─ otherwise              → UPSTREAM_URL (Anthropic)
```

- Only POST requests are inspected for model-based routing
- Non-POST requests (GET, etc.) always go to the default upstream
- If body is not valid JSON, falls back to default upstream
- Built-in model list: `mimo-v2.5-pro`, `mimo-v2.5`, `mimo-v2-pro`, `mimo-v2-omni`, `mimo-v2.5-pro-claude`
- Override via `MIMO_MODELS` env var (comma-separated)

## Token Budget & Rate Limiting

Each OIDC token is tracked by its `jti` claim via a **Durable Object** (`TokenBucket`):

- **Budget** — `MAX_REQUESTS_PER_TOKEN` total requests per token (default 200). Returns `429` + `"Token budget exhausted"` when exhausted.
- **Rate limit** — `RATE_LIMIT_PER_SECOND` requests/sec per token (default 2). Returns `429` + `"Rate limit exceeded"` + `Retry-After: 1`.

Both enforced atomically — no race conditions per token. Budget/rate limits apply regardless of upstream provider.

## Token Age vs Expiry

Two gates, both must pass:

1. IdP's `exp` claim (always enforced).
2. Operator's `MAX_TOKEN_AGE_SECONDS` cap on `iat` (default 1200s).

`MAX_TOKEN_AGE_SECONDS` cannot extend a token past `exp`. For workflows >5 min, refresh the OIDC token.

## Development

```bash
npm run dev      # local dev via wrangler
npm test         # run tests via vitest
npm run deploy   # deploy to Cloudflare
```

## Project Structure

```
.
├── src/
│   └── index.js          # Worker entry + TokenBucket DO + model routing
├── test/
│   └── index.test.js     # Tests
├── docs/
│   └── README.md         # Original upstream documentation
├── wrangler.toml         # Cloudflare Worker config
├── vitest.config.js      # Test config
├── LICENSE               # Apache 2.0
└── package.json
```

## Attribution

This project is derived from the **OIDC API Broker** component of [AAIF Goose](https://github.com/aaif-goose/goose/tree/main/oidc-proxy), licensed under the [Apache License 2.0](LICENSE).

Per the Apache 2.0 license terms:

- The original copyright notice and license are retained in [LICENSE](LICENSE).
- Copyright 2024 Block, Inc.
- Modifications from the original source are documented in this repository's commit history.

When redistributing or creating derivative works, you must:

1. Include a copy of the [Apache 2.0 License](LICENSE)
2. Retain all copyright, patent, trademark, and attribution notices from the original source
3. State any changes made to modified files
4. Include the NOTICE file content (if applicable) in your distribution

For full license terms, see: <http://www.apache.org/licenses/LICENSE-2.0>

## License

[Apache License 2.0](LICENSE) — Copyright 2026 KietNT.
