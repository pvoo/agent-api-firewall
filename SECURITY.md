# Security

## Reporting Vulnerabilities

Open a private security advisory on GitHub or email the author directly. Do not open public issues for security vulnerabilities.

## Trust Boundaries

- Agents do not hold upstream API keys.
- Caddy runs on a trusted host and injects upstream credentials.
- Wallarm validates requests against OpenAPI specs and blocks unlisted operations.
- Per-agent access is enforced by route + token + policy mapping.

## Design Assumptions

- Proxy binds to localhost by default (`VAULT_BIND=127.0.0.1`).
- Each agent has a unique strong token.
- Every sensitive API should use a strict OpenAPI spec (`spec`) policy.
- Policies without a spec are pass-through and should be treated as high risk.

## Hardening Checklist

- [ ] Bind to localhost only (default)
- [ ] Use unique high-entropy token per agent
- [ ] Store `.env` with minimal host access permissions
- [ ] Use OpenAPI specs for sensitive APIs
- [ ] Use `additionalProperties: false` where practical in request schemas
- [ ] Review logs for denied requests and drift
- [ ] Restrict Docker network exposure
- [ ] Pin image versions (default in generated compose)
