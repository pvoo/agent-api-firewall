# Agent API Firewall

Policy-driven API gateway for AI agents, powered by [Wallarm API Firewall](https://github.com/wallarm/api-firewall).

## Why this project

This project keeps upstream API keys on the host and gives agents controlled access through:

- Agent-specific routes: `/<agent>/<api>/*`
- Agent tokens (`X-Agent-Token` by default)
- OpenAPI policy profiles (allowlist, default deny)

Result: one reusable firewall that supports different agent permissions without hardcoding provider logic.

## Architecture

```text
Agent (no upstream keys)
  -> Agent API Firewall (Caddy + Wallarm)
    -> External APIs
```

Request flow:

1. Route lookup by `/<agent>/<api>/*`
2. Agent token validation
3. Policy enforcement (if `spec` configured)
4. Upstream auth/header injection
5. Forward to provider

## Quick start

```bash
git clone https://github.com/pvoo/agent-api-firewall.git
cd agent-api-firewall
cp .env.example .env
```

Populate `.env`:

- Upstream credentials (`INTERCOM_TOKEN`, `OPENAI_API_KEY`, ...)
- Agent tokens (`AGENT_SUPPORT_TOKEN`, ...)

Render and run:

```bash
./render
docker compose up -d
```

Run live tests:

```bash
./test
```

## Configuration model

`config.yaml` defines five things:

- `auth.header`: request header used for agent tokens
- `wallarm`: firewall image and runtime tuning
- `agents`: token + API policy access per agent
- `apis`: upstream credentials/headers/query auth
- `apis.<api>.policies`: named policy profiles (`spec` optional)

Minimal example:

```yaml
listen: ":8282"
auth:
  header: "X-Agent-Token"
wallarm:
  image: wallarm/api-firewall:v0.9.5
  server_read_buffer_size: 65536

agents:
  support:
    token: "${AGENT_SUPPORT_TOKEN}"
    access:
      - api: intercom
        policy: support

apis:
  intercom:
    upstream: https://api.intercom.io
    auth: "Bearer ${INTERCOM_TOKEN}"
    headers:
      Intercom-Version: "2.14"
    policies:
      support:
        spec: specs/intercom-support.yaml
```

## Routes and usage

If `support` has access to `intercom`, call:

```bash
curl "http://localhost:8282/support/intercom/me" \
  -H "X-Agent-Token: $AGENT_SUPPORT_TOKEN"
```

No upstream key is sent by the agent. Caddy injects it from host env.

## Testing

`./test` validates:

- Unknown routes return `404`
- Missing/wrong token returns `401`
- Correct token reaches authorized route
- Spec-backed policies block unknown endpoints (`403`)
- Cross-agent token isolation

`./test` uses a probe header (`X-Agent-Firewall-Probe: 1`) for deterministic auth checks without relying on upstream API behavior.

## Included policy packs

- `specs/intercom-support.yaml`:
  - Support-safe Intercom subset
  - Team-scoped search constraints
  - No bulk export/download/contact mutation
- `specs/openai-safe.yaml`:
  - Models, chat completions, embeddings, responses
  - Sensitive endpoints blocked by omission

## Project layout

```text
.
├── config.yaml
├── .env.example
├── render
├── test
├── specs/
│   ├── intercom-support.yaml
│   └── openai-safe.yaml
├── Caddyfile              # generated
└── docker-compose.yml     # generated
```

## Security notes

- Default bind is localhost-only (`127.0.0.1`)
- Keep `.env` private and out of version control
- Prefer spec-backed policies for all sensitive APIs
- See `SECURITY.md` for hardening guidance

## License

MIT
