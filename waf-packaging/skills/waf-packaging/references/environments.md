# Environments — dev / staging / prod, CORS, and where TLS terminates

The same compose + WAF should run in **every** environment; only the env values
change. Running the WAF in dev (not just prod) is the point: you catch false
positives and CORS problems against real navigation before they reach production.

## Ask the user which environment this is

Classify as `development` / `staging` / `production` / other, and record it. Drive
an explicit `ENVIRONMENT` variable in the app and the compose so behavior is
deterministic (don't infer it from hostnames).

## Per-environment values

| Setting | dev / local | staging | production |
| --- | --- | --- | --- |
| `ENVIRONMENT` | `development` | `staging` | `production` |
| `MODSEC_RULE_ENGINE` | `DetectionOnly` | `DetectionOnly` → `On` | `On` (after tuning) |
| `PARANOIA` / `BLOCKING_PARANOIA` | `1` | `1`–`2` | `1`–`2`, raise carefully |
| `ANOMALY_INBOUND` | `5` | `5` | `5` (lower = stricter) |
| `MODSEC_AUDIT_ENGINE` | `On` (observe all) | `On` | `RelevantOnly` (log volume) |
| `CORS_ORIGINS` (app) | `localhost` ports | staging domain | prod domain only |
| TLS | none (plain HTTP) | edge/ALB or PaaS | edge/ALB or PaaS |
| App debug / docs | may be open | gated | gated / disabled |

Promote `DetectionOnly → On` only after Phase 3 (observe) is clean in that
environment. Never ship `On` to prod without having observed real traffic first —
ideally in staging or in dev with representative navigation.

## App-side environment awareness (not just the WAF)

Have the application itself respect `ENVIRONMENT`, so hardening is consistent with
the WAF layer. For a FastAPI backend, a boot-time guard pays off:

```python
ENVIRONMENT = os.getenv("ENVIRONMENT", "development").strip().lower()
IS_PRODUCTION = ENVIRONMENT == "production"

# Never allow an auth-bypass dev mode in prod
if DEV_MODE and IS_PRODUCTION:
    raise RuntimeError("DEV_MODE must be off in production")

# Require real secrets in prod (no public repo defaults)
JWT_SECRET = os.getenv("JWT_SECRET", "" if IS_PRODUCTION else "dev-only-secret")
if IS_PRODUCTION and not JWT_SECRET:
    raise RuntimeError("JWT_SECRET is required in production")
```

Also hide stack version from the public in every non-dev environment: minimal
`/health` (bare `200 ok`), auth-gate any `/version`, keep Swagger/OpenAPI behind
auth. See `fastapi-tuning.md`.

## CORS allowlist — derive it from the API consumers

CORS is an **app-level** control (set it in the backend), independent of the WAF.
Get the origins right per environment instead of using `*`.

**Key insight:** if the front-end calls the API with **relative paths** (e.g. a
single origin serving both the SPA and `/api`, which is the common unified-image /
behind-the-WAF setup), those requests are **same-origin** and the browser never
does CORS at all. In that case `*` is both unnecessary and unsafe — a tight
allowlist changes nothing for normal use and closes the hole.

CORS only matters for:
- a front-end served from a **different** origin (e.g. `ng serve` on `:4200` during
  front-end dev, or a separately hosted SPA),
- third-party/external API consumers.

Recommended FastAPI config (env-driven allowlist, not `*`):

```python
_default = "http://localhost:8080,http://localhost:4200"
CORS_ORIGINS = [o.strip() for o in os.getenv("CORS_ORIGINS", _default).split(",") if o.strip()]

app.add_middleware(
    CORSMiddleware,
    allow_origins=CORS_ORIGINS,                       # allowlist, never ["*"]
    allow_credentials=True,
    allow_methods=["GET","POST","PUT","PATCH","DELETE","OPTIONS"],
    allow_headers=["Authorization","Content-Type"],
)
```

Per environment set `CORS_ORIGINS` to: dev → `localhost:8080,localhost:4200`;
staging → the staging domain; prod → the prod domain only. Verify by preflighting:
an allowed `Origin` is reflected in `Access-Control-Allow-Origin`; an unknown
`Origin` is **not** reflected (so the browser blocks it). See `verification.md`.

Don't keep adding origins for internal APIs/services: server-to-server calls and
same-origin front-ends don't need CORS. Add an origin only when a real browser on a
*different* origin must call the API.

## Where TLS terminates — the `PROXY_SSL` decision

The WAF speaks plain HTTP to the backend in all these cases; what changes is the
edge and how the route reaches the WAF:

- **External ALB / load balancer terminates TLS** (edge → HTTP to the stack): point
  the ALB target group / listener at the WAF. `PROXY_SSL=off`. **In this setup the
  PaaS (e.g. Coolify) is NOT doing TLS — it just routes HTTP.** Confirm this with
  the user; do not assume the PaaS terminates TLS.
- **PaaS proxy terminates TLS** (Coolify/Traefik/Caddy): point the PaaS domain/route
  at the `waf` service instead of the app. Backend hop stays HTTP (`PROXY_SSL=off`).
- **Local dev / no TLS**: everything plain HTTP, `PROXY_SSL=off`.

Set `REAL_IP_FROM` / `REAL_IP_HEADER` (e.g. `X-Forwarded-For`) so the WAF logs the
real client IP coming from the edge instead of the proxy's IP.

## PaaS routing checklist (Coolify example, ALB-terminated TLS)

1. External ALB terminates TLS and forwards HTTP to the host/PaaS.
2. In Coolify, expose/route the **`waf`** service (not `app`) on the app's domain.
3. `app` has no published host port — only reachable via the compose network.
4. Env in Coolify: `ENVIRONMENT=production`, `DEV_MODE=0`, `CORS_ORIGINS=<prod domain>`,
   WAF `MODSEC_RULE_ENGINE` starting at `DetectionOnly` until tuned.
5. Keep rollback ready: `MODSEC_RULE_ENGINE=Off` or route back to `app` directly.
