# waf-packaging

Kiro Power (Agent Plugin format) that teaches the agent to **package any web app
behind a ModSecurity v3 + OWASP CRS WAF** running as a reverse-proxy container in
front of the app — technology-agnostic, but tuned for **FastAPI** backends.

The app's own image is left untouched; the WAF is a separate, reversible layer
added via `docker-compose`. The skill is phase-driven (recon → insert in
DetectionOnly → tune CRS → observe → block) so you never flip to blocking mode
blind.

## What it covers

- **Driving tuning from the app's own docs** — ingest the user's OpenAPI/Swagger,
  Postman collection, or API reference (or scan the routes) to derive exclusions and
  the CORS allowlist, instead of guessing.
- **Environment awareness** — classify dev / staging / prod and apply per-env values
  (engine mode, paranoia, audit level, CORS origins, TLS). Run the WAF in dev too, so
  security + navigation are tested the same way they'll run in prod.
- Inserting the WAF container in front of an existing app (compose), with the app
  no longer publishing a host port.
- CRS tuning as `-before` / `-after` plugins, including the **FastAPI exclusion
  set** (REST verbs, Swagger/OpenAPI, free-text search vs the SQLi family, body
  limits, health).
- **CORS** done right (env-driven allowlist, not `*`): same-origin front-ends need
  no CORS; add an origin only for a different-origin SPA or external consumer.
- **TLS placement** — works whether TLS is terminated at an external **ALB** (the
  PaaS/Coolify only routes HTTP — do not assume Coolify terminates TLS) or at the
  PaaS proxy; the backend hop stays plain HTTP (`PROXY_SSL=off`).
- Verification: smoke tests, a binary "does it actually block" test, audit-log
  triage, and **autonomous UI validation with AI + Playwright**.
- Hiding stack version info (FastAPI/Python/Angular/server) from unauthenticated
  users; minimal `/health`.
- Optional: managing the WAF from an **admin-only UI** safely (config in DB, not
  `.env`; no Docker socket; validate everything written to rule files).
- Works alongside weak perimeter controls like **nginx/npm basic auth** — the WAF
  filters payloads on top of whatever gates access.

## Based on

This skill packages and tunes open-source projects — credit to:

- **OWASP Core Rule Set (CRS)** — generic attack-detection rules.
  <https://github.com/coreruleset/coreruleset> (Apache-2.0)
- **coreruleset/modsecurity-crs-docker** — official ModSecurity v3 + CRS Docker
  images (nginx/Apache/OpenResty connectors), env-configurable.
  <https://github.com/coreruleset/modsecurity-crs-docker> · images:
  `owasp/modsecurity-crs` on Docker Hub.

It was distilled from a real integration where these two repos made the
implementation straightforward: a FastAPI + Angular app served by a unified
nginx+uvicorn image, placed behind the OWASP CRS nginx-alpine container in front of
a load balancer that terminates TLS.

> Content was rephrased for compliance with licensing restrictions. Keep upstream
> licenses when redistributing.

## Install

Kiro → Powers panel → **Add Custom Power** → **Import power from GitHub**, pointing
at this directory:

```
https://github.com/JosSantamaria/Kiro-Skills-Ports/tree/main/waf-packaging
```

## Package layout

```
waf-packaging/
├── plugin.json                 # Agent Plugin manifest
├── skills/
│   └── waf-packaging/
│       ├── SKILL.md            # the skill behavior (phase-driven workflow)
│       └── references/
│           ├── environments.md # dev/staging/prod, CORS, TLS/ALB vs Coolify
│           ├── fastapi-tuning.md
│           ├── crs-tuning.md
│           ├── verification.md
│           └── ui-management.md
├── examples/
│   └── docker-compose.waf.yml  # copy-paste starting point
├── LICENSE
└── README.md
```

## License

MIT (this port). Upstream CRS is Apache-2.0; the Docker images carry their own
licenses — respect them when distributing.
