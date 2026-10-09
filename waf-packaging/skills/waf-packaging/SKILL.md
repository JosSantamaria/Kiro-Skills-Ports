---
name: waf-packaging
description: >
  Package any web application behind a ModSecurity v3 + OWASP CRS Web Application
  Firewall, running as a reverse-proxy container in front of the app. Use when the
  user wants to add a WAF to an existing app, "put a firewall in front of my API",
  harden a backend exposed to the internet, protect against SQLi/XSS/scanners, or
  ship a WAF alongside the server image. Technology-agnostic (any backend served
  over HTTP) but tuned for FastAPI backends. Deterministic, phase-driven, and
  reversible: never flips straight to blocking mode without an observation phase.
---

# WAF Packaging — ModSecurity + OWASP CRS in front of any app

Add a Web Application Firewall to an existing web app by placing a **ModSecurity v3
+ OWASP Core Rule Set (CRS)** container in front of it as a reverse proxy. The app
image is left unchanged; the WAF is a separate, reversible layer.

This skill is **phase-driven by design**. Do not jump to blocking mode. Do not
modify the app's own image to shoehorn ModSecurity in. Do not disable whole rule
families globally to "make it work" — scope exclusions to the exact paths.

## Core principles (read first, non-negotiable)

1. **WAF lives in front, not inside.** A WAF is a perimeter control. Put it in a
   separate container that proxies to the app. Never embed WAF logic inside the
   application process (e.g. as framework middleware) — that defeats the purpose.
2. **Start in DetectionOnly.** Never deploy straight to blocking. First observe
   false positives against real traffic, tune, then switch to blocking.
3. **Scope exclusions to paths.** When the CRS false-positives on legitimate
   traffic, disable the *specific* rule IDs on the *specific* path — never turn
   off a whole attack family globally.
4. **Keep it reversible.** The WAF must be removable (`SecRuleEngine Off` or
   detach the container) without touching the app's availability.
5. **Don't trust the app to be the last line.** Fixing an app-level vuln and
   adding the WAF are complementary; the WAF buys time, it does not replace fixes.

## When this applies

- An app already serves HTTP (any stack: FastAPI, Express, Django, Rails, PHP,
  a static SPA + API, etc.). **Primary target: FastAPI backends.**
- The app is (or will be) exposed to the internet, possibly only behind weak
  controls like **nginx/npm basic auth** — a WAF adds real attack filtering on top.
- Deploy target is Docker / docker-compose, or a PaaS that runs compose/images
  (Coolify, Dokku, Render, etc.) behind a load balancer that terminates TLS.

## Upstream projects this builds on (attribution)

This skill packages and tunes well-known open-source projects. Credit and use:

- **OWASP Core Rule Set (CRS)** — the generic attack-detection rules.
  https://github.com/coreruleset/coreruleset  (Apache-2.0)
- **coreruleset/modsecurity-crs-docker** — the official Docker images that bundle
  ModSecurity v3 + the CRS + the web-server connector (nginx / Apache / OpenResty),
  configurable entirely via environment variables.
  https://github.com/coreruleset/modsecurity-crs-docker
  Images: `owasp/modsecurity-crs` on Docker Hub.

Content was rephrased for compliance with licensing restrictions; always keep the
upstream licenses when distributing.

## Workflow overview

```
Phase 0  Recon        → map the app: how it's served, what port, how traffic enters, TLS where
Phase 1  Insert WAF   → add the WAF container in front (DetectionOnly), app stops publishing its port
Phase 2  Tune CRS     → app-specific exclusions so legit traffic isn't flagged
Phase 3  Observe      → read the audit log, triage false positives, adjust
Phase 4  Block        → switch SecRuleEngine to On, verify legit traffic still passes
```

---

## Phase 0 — Recon

Record this before changing anything:

1. **How is the app served today?** Single container (nginx+app) or app alone?
   What internal port (e.g. `80`, `8000`, `3000`)?
2. **How does traffic enter?** Direct host port, a reverse proxy, or a PaaS proxy
   (Coolify/Traefik)? Where is **TLS terminated** (load balancer / ALB / Caddy)?
   The WAF should speak plain HTTP to the backend; TLS stays at the edge.
3. **What legitimate request patterns look like "attacks"?** This is the key input
   for Phase 2. For FastAPI specifically, note:
   - Search endpoints that take free-text query params (quotes, `*`, `%`, SQL-ish
     terms) → will false-positive on the SQLi family (942xxx).
   - `/docs`, `/redoc`, `/openapi.json` (Swagger) → noisy payloads.
   - REST verbs: `PUT`/`PATCH`/`DELETE` (CRS 911100 only allows GET/HEAD/POST/OPTIONS by default).
   - Large request bodies on upload/sync endpoints.
4. **Existing weak controls?** e.g. npm/nginx **basic auth** in front. The WAF goes
   *outside or alongside* that: basic auth gates access, the WAF filters payloads.

See `references/fastapi-tuning.md` for the FastAPI-specific checklist.

## Phase 1 — Insert the WAF container

Use the official image. **Pin a tag** (don't use a floating `latest`-style tag).
LTS tags look like `4.x-nginx-alpine-lts`; verify current tags on Docker Hub
(`owasp/modsecurity-crs`) — generic tags like `4-nginx-alpine` may not resolve.

Minimal `docker-compose.yml` change (nginx variant shown):

```yaml
services:
  app:
    # ...existing app service...
    # Stop publishing a host port — only the WAF should be reachable.
    expose:
      - "80"            # the app's internal port
    # (remove the "ports:" mapping that exposed it to the host)

  waf:
    image: owasp/modsecurity-crs:4.25-nginx-alpine-lts   # pin a real tag
    ports:
      - "8080:8080"     # external entrypoint (plain HTTP; TLS is at the edge/LB)
    environment:
      - BACKEND=http://app:80          # where the WAF proxies to (compose service name)
      - PROXY_SSL=off                  # TLS terminated upstream (ALB/LB); backend is HTTP
      - PORT=8080                      # image runs as unprivileged user → not :80
      - SERVER_NAME=localhost
      - MODSEC_RULE_ENGINE=DetectionOnly   # START HERE: log, do not block
      - PARANOIA=1
      - BLOCKING_PARANOIA=1
      - ANOMALY_INBOUND=5
      - ANOMALY_OUTBOUND=4
      - MODSEC_REQ_BODY_LIMIT=52428800     # raise if the app has large uploads
      - MODSEC_AUDIT_ENGINE=On             # Phase 3: capture everything that fires
      - MODSEC_AUDIT_LOG_FORMAT=JSON
      - MODSEC_AUDIT_LOG_TYPE=Concurrent   # see gotcha #4 (unprivileged user + logs)
      - MODSEC_AUDIT_STORAGE_DIR=/var/log/modsecurity/audit
      - REAL_IP_FROM=0.0.0.0/0             # trust the edge LB for client IP
      - REAL_IP_HEADER=X-Forwarded-For
    volumes:
      # App-specific tuning mounted as CRS plugins (Phase 2):
      - ./waf/conf/app-config-before.conf:/etc/modsecurity.d/owasp-crs/plugins/app-config-before.conf:ro
      - ./waf/conf/app-exclusions.conf:/etc/modsecurity.d/owasp-crs/plugins/app-exclusions-after.conf:ro
      - waf_logs:/var/log/modsecurity
    depends_on: [app]
    restart: unless-stopped

volumes:
  waf_logs:
```

Resulting flow:

```
Local (HTTP):  client → waf:8080 → app:<port>
Prod:          LB/ALB (TLS) → waf → app  (PROXY_SSL=off: backend stays HTTP)
```

Bring it up and smoke-test (`references/verification.md`): health, home, login and a
representative legit request should all return their normal codes **through the WAF**.

## Phase 2 — Tune CRS for the app

Mount two CRS plugin files (they are auto-included by the image's `setup.conf`,
which pulls `owasp-crs/plugins/*-before.conf` and `*-after.conf`):

- `*-before.conf` → config that must run **before** the rules (e.g. allowed methods).
- `*-after.conf` → exclusions by path (`ctl:ruleRemoveById=...`) that must run
  **after** the rules exist.

See `references/crs-tuning.md` for ready-to-copy snippets, and
`references/fastapi-tuning.md` for the FastAPI exclusion set (docs, search SQLi,
REST verbs, body limit).

**Gotcha:** `ctl:requestBodyLimit` is a ModSecurity **v2** action and does NOT exist
in v3 — setting it in a rule aborts rule loading. Use the global `SecRequestBodyLimit`
(env `MODSEC_REQ_BODY_LIMIT`) instead.

## Phase 3 — Observe

With `MODSEC_RULE_ENGINE=DetectionOnly` and `MODSEC_AUDIT_ENGINE=On`, drive real
traffic through the app, then read the JSON audit log and triage every rule that
fired against legitimate requests. Add path-scoped exclusions for the false
positives. Do not silence a whole family globally.

## Phase 4 — Switch to blocking

Only after Phase 3 is clean: set `MODSEC_RULE_ENGINE=On`, recreate the WAF, and
re-run the smoke test. A representative SQLi/XSS payload should now return `403`;
all legitimate traffic must still pass. Keep the rollback ready
(`MODSEC_RULE_ENGINE=Off` or remove the service and re-expose the app port).

**Validate the real UI, not just HTTP paths.** `curl` won't catch a rule that breaks
a form post, upload, or XHR that only a browser sends. Drive the app with an AI +
Playwright flow (an MCP Playwright server or a standalone script) in `DetectionOnly`
vs `On` and diff the runs: any user action that turns into a `403` under `On` is a
false positive to exclude. See `references/verification.md` §7.

## Common gotchas (hard-won)

1. **Tag resolution** — `4-nginx-alpine` may not exist; use a real pinned tag like
   `4.25-nginx-alpine-lts`. List tags before composing.
2. **Unprivileged user / port** — the image runs as a non-root user and listens on
   `8080`, not `80`. Set `PORT` accordingly and publish `8080`.
3. **Custom rules aren't auto-loaded from `/custom`** — mount them under
   `owasp-crs/plugins/` as `*-before.conf` / `*-after.conf`, which `setup.conf`
   includes.
4. **Audit log not written** — the non-root process can't create a serial log file
   in a root-owned `/var/log/modsecurity`. Use `MODSEC_AUDIT_LOG_TYPE=Concurrent`
   with `MODSEC_AUDIT_STORAGE_DIR=/var/log/modsecurity/audit` (the image creates the
   `audit/` subdir owned by the web-server user). ModSecurity-nginx also writes rule
   messages to the container's stderr (nginx `error_log`).
5. **DetectionOnly ≠ blocked** — in DetectionOnly, attacks reach the backend. A
   non-200 you see is the app's own auth/validation, not the WAF. To prove the WAF
   evaluates, temporarily set `On` and confirm a SQLi returns `403`.
6. **PaaS proxies (Coolify/Traefik)** — point the domain/route at the `waf` service,
   not the app. Keep `PROXY_SSL=off` because the edge terminates TLS before the WAF.
7. **Basic auth in front (npm/nginx)** — the WAF composes with it, it doesn't replace
   it. Basic auth controls *who* connects; the WAF filters *what* they send. Put the
   WAF so that even authenticated users' payloads are inspected.

## Making it manageable from a UI (optional, advanced)

If the user wants to tune the WAF from an admin UI (not shell), do NOT write to the
server `.env`. Store config (engine mode, paranoia, anomaly threshold, path
exclusions, IP allow/deny) in the app's database, render it to CRS plugin files on a
shared volume, and reload the WAF (prefer an in-container `inotify` watcher over
mounting the Docker socket, which would be a privilege-escalation risk). See
`references/ui-management.md`.

## Reference files

| File | When to read |
| --- | --- |
| `references/fastapi-tuning.md` | Always for FastAPI backends — the exclusion set and why |
| `references/crs-tuning.md` | Writing the before/after plugin files; rule-ID exclusions |
| `references/verification.md` | Smoke tests, proving the WAF blocks, reading the audit log, **autonomous UI validation with AI + Playwright** |
| `references/ui-management.md` | Building an admin UI to manage the WAF safely |
