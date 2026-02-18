# 🛡️ Agent API Firewall

OpenAPI-based request firewall for AI agents, powered by [Wallarm API Firewall](https://github.com/wallarm/api-firewall).

```
Agent (no keys) ──▶ Agent API Firewall ──▶ External APIs
                    │                       
                    ├ Wallarm validates against OpenAPI spec (unlisted = 403)
                    ├ Caddy injects auth + routes by path prefix
                    └ Keys never leave the host
```

## What This Does

**[Wallarm API Firewall](https://github.com/wallarm/api-firewall)** is the enforcement engine — it validates every request against an OpenAPI spec and blocks anything not explicitly allowed. This repo wraps it with:

- **Auth injection** — credentials live in `.env`, injected per-API by Caddy
- **Path routing** — `/openai/*` → OpenAI, `/intercom/*` → Intercom, etc.
- **Config generator** — one `config.yaml` produces the full Docker Compose stack

You write an OpenAPI spec that acts as an **allowlist**. Wallarm blocks the rest.

## Quick Start

```bash
git clone https://github.com/pvoo/agent-api-firewall.git
cd agent-api-firewall
```

1. **Add your API keys**
   ```bash
   cp .env.example .env
   vi .env                        # add INTERCOM_TOKEN, OPENAI_API_KEY, etc.
   ```

2. **Review `config.yaml`** — two sample APIs included (Intercom, OpenAI)

3. **Generate + start**
   ```bash
   ./render                       # config.yaml → Caddyfile + docker-compose.yml
   docker compose up -d
   ```

4. **Verify**
   ```bash
   ./test                         # runs allow/block checks against live proxy
   ```

5. **Use from your agent**
   ```bash
   # No API key needed — the proxy injects it
   curl http://localhost:8282/openai/v1/models
   ```

## How It Works

```
config.yaml          ─── ./render ───▶  Caddyfile
  + specs/*.yaml                        docker-compose.yml
  + .env                                  │
                                          ▼
                              ┌─────────────────────┐
                              │  Caddy (router)      │ :8282
                              │  ├ /openai/*         │
                              │  └ /intercom/*       │
                              └──────┬──────┬────────┘
                                     │      │
                              ┌──────▼──┐ ┌─▼────────┐
                              │ Wallarm  │ │ Wallarm   │
                              │ (OpenAI  │ │(Intercom  │
                              │  spec)   │ │ spec)     │
                              └──────┬───┘ └─┬────────┘
                                     │       │
                              ┌──────▼───────▼────────┐
                              │   External APIs        │
                              └────────────────────────┘
```

- **Caddy** handles routing (path prefix → backend), auth header injection, and TLS
- **Wallarm API Firewall** (one container per spec) validates requests against the OpenAPI spec — anything not listed returns 403
- APIs without a `spec:` get pass-through (auth injection only, no validation)

## Configuration

### config.yaml

```yaml
listen: ":8282"

apis:
  intercom:
    upstream: https://api.eu.intercom.io
    auth: "Bearer ${INTERCOM_TOKEN}"
    headers:
      Intercom-Version: "2.11"
    spec: specs/intercom.yaml          # ← Wallarm validates against this

  openai:
    upstream: https://api.openai.com
    auth: "Bearer ${OPENAI_API_KEY}"
    spec: specs/openai.yaml
```

### Auth Methods

```yaml
auth: "Bearer ${TOKEN}"          # Authorization header (Bearer)
auth: "${API_KEY}"                # Authorization header (plain)

auth_query:                       # Query parameter
  api_key: "${KEY}"               # → ?api_key=<value>

headers:                          # Extra headers
  X-Custom: "value"
```

## Writing OpenAPI Specs (Security Policy)

The OpenAPI spec **is** your security policy. Only paths and methods listed in the spec are allowed — everything else is blocked by Wallarm.

### Example: Team-Scoped Access

The included Intercom spec restricts conversation search to specific teams:

```yaml
# specs/intercom.yaml (excerpt)
paths:
  /conversations/search:
    post:
      requestBody:
        content:
          application/json:
            schema:
              properties:
                query:
                  properties:
                    field:
                      type: string
                    operator:
                      type: string
                    value:
                      type: string
              # Wallarm validates the body structure
```

Combined with **not listing** `/conversations` (list all), agents can only search — never browse all conversations. Add `enum` constraints on fields like `team_assignee_id` to limit which teams are visible.

### Tips

- Start restrictive, add endpoints as needed
- Use `additionalProperties: false` on request bodies for strict enforcement
- Comment blocked endpoints at the bottom of the spec for documentation

## Adding a New API

1. Add entry to `config.yaml` (upstream + auth)
2. Add credentials to `.env`
3. Write an OpenAPI spec in `specs/` (or omit `spec:` for pass-through)
4. `./render && docker compose up -d`
5. Add test cases to `./test`

## Included Specs

| API | Allowed | Blocked |
|-----|---------|---------|
| **Intercom** | Search/read/reply conversations, contacts, articles, teams | Delete, export, create contacts, send messages, list all conversations |
| **OpenAI** | Chat completions, embeddings, audio, models | Fine-tuning, files, images, assistants |

## Security

- Binds to **localhost only** by default (`VAULT_BIND=127.0.0.1`)
- See [SECURITY.md](SECURITY.md) for hardening checklist

## Requirements

- Docker (Compose v2)
- [yq v4+](https://github.com/mikefarah/yq)

## Contributing

- `./render` and check the generated files
- `./test` against a running proxy
- Don't commit `.env`, `Caddyfile`, or `docker-compose.yml` (generated)

## Author

Paul van Oorschot — [@pvoo](https://github.com/pvoo)

## License

MIT
