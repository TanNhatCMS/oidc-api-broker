# OIDC API Broker

A Cloudflare Worker that authenticates GitHub Actions OIDC tokens and brokers requests to an upstream API with an injected API key. CI workflows call APIs without storing long-lived secrets in GitHub.

> **Forked from [aaif-goose/goose/oidc-proxy](https://github.com/aaif-goose/goose/tree/main/oidc-proxy)** — see [Attribution](#attribution) below.

## Architecture

```
GitHub Actions (OIDC token) → Worker (validate JWT, inject API key) → Upstream API
```

1. GitHub Actions mints an OIDC token with a configured audience
2. Workflow sends requests to this proxy, passing the OIDC token as the API key
3. Worker validates JWT against GitHub's JWKS, checks issuer / audience / age / repo
4. Valid requests forwarded to upstream with the real API key injected

## Setup

```bash
npm install
```

## Configuration

Edit `wrangler.toml`:

| Variable | Description |
|---|---|
| `OIDC_ISSUER` | `https://token.actions.githubusercontent.com` |
| `OIDC_AUDIENCE` | Audience your workflow requests (e.g. `goose-oidc-api-broker`) |
| `MAX_TOKEN_AGE_SECONDS` | Upper bound on `iat` age (default `1200` = 20 min). Applied *in addition to* IdP's `exp` claim. |
| `MAX_REQUESTS_PER_TOKEN` | Max requests per OIDC token (default `200`) |
| `RATE_LIMIT_PER_SECOND` | Max requests/sec per token (default `2`) |
| `ALLOWED_REPOS` | *(optional)* Comma-separated `owner/repo` list |
| `ALLOWED_REFS` | *(optional)* Comma-separated allowed refs |
| `UPSTREAM_URL` | Upstream API base URL |
| `UPSTREAM_AUTH_HEADER` | Header name for API key (e.g. `x-api-key`, `Authorization`) |
| `UPSTREAM_AUTH_PREFIX` | *(optional)* Prefix before key (e.g. `Bearer `) — omit for raw value |
| `CORS_ORIGIN` | *(optional)* Allowed CORS origin |
| `CORS_EXTRA_HEADERS` | *(optional)* Additional CORS allowed headers |

Set the upstream API key as a secret:

```bash
npx wrangler secret put UPSTREAM_API_KEY
```

### Examples

**Anthropic**

```toml
UPSTREAM_URL = "https://api.anthropic.com"
UPSTREAM_AUTH_HEADER = "x-api-key"
CORS_EXTRA_HEADERS = "anthropic-version"
```

**OpenAI-compatible**

```toml
UPSTREAM_URL = "https://api.openai.com"
UPSTREAM_AUTH_HEADER = "Authorization"
UPSTREAM_AUTH_PREFIX = "Bearer "
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

  - name: Call API through proxy
    env:
      ANTHROPIC_BASE_URL: https://oidc-api-broker.your-subdomain.workers.dev
      ANTHROPIC_API_KEY: ${{ steps.oidc.outputs.token }}
    run: goose run --recipe my-recipe.yaml
```

## Token Budget & Rate Limiting

Each OIDC token is tracked by its `jti` claim via a **Durable Object** (`TokenBucket`):

- **Budget** — `MAX_REQUESTS_PER_TOKEN` total requests per token (default 200). Returns `429` + `"Token budget exhausted"` when exhausted.
- **Rate limit** — `RATE_LIMIT_PER_SECOND` requests/sec per token (default 2). Returns `429` + `"Rate limit exceeded"` + `Retry-After: 1`.

Both enforced atomically — no race conditions per token.

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
│   └── index.js          # Worker entry + TokenBucket Durable Object
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

[Apache License 2.0](LICENSE) — Copyright 2024 Block, Inc.
