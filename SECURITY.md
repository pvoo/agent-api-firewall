# Security

## Reporting Vulnerabilities

Open a private security advisory on GitHub or email the author directly. Do **not** open public issues for security vulnerabilities.

## Design Assumptions

- **Wallarm API Firewall** is the enforcement engine — it blocks any request not matching the OpenAPI spec.
- **Caddy** handles routing and auth injection. It runs on the trusted host alongside the keys.
- By default, the proxy binds to **localhost only** (`127.0.0.1`). Do not expose to untrusted networks without additional auth.
- APIs without an OpenAPI `spec:` get **full pass-through** — only auth is injected, no endpoint restrictions.
- Query-param auth (`auth_query`) places secrets in URLs, which may appear in logs.

## Hardening Checklist

- [ ] Bind to localhost only (default)
- [ ] Pin container image versions (default)
- [ ] Use OpenAPI specs for all sensitive APIs
- [ ] Review Wallarm logs for unexpected requests
- [ ] Restrict Docker network access
- [ ] Consider adding a shared-secret header for proxy auth
