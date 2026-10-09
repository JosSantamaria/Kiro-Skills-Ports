# FastAPI tuning — CRS exclusions for a FastAPI backend

FastAPI apps trip a predictable set of CRS false positives. Apply these
path-scoped exclusions so legitimate traffic isn't flagged. Each is scoped to the
exact route prefix — never disable a family globally.

## The FastAPI exclusion set

| Concern | What breaks | Fix |
| --- | --- | --- |
| **REST verbs** | CRS 911100 allows only GET/HEAD/POST/OPTIONS; FastAPI uses PUT/PATCH/DELETE | In a `-before` plugin, set `tx.allowed_methods` to include them |
| **Swagger / OpenAPI** | `/docs`, `/redoc`, `/openapi.json` emit payloads flagged as anomalies | Turn the engine off for those exact paths (they should be auth-gated anyway) |
| **Search endpoints** | Free-text `?q=`/`?term=` with quotes, `*`, `%`, SQL-ish words → SQLi family (942xxx) | Drop `942000-942999` on those specific search paths only |
| **Large bodies** | Upload / bulk-sync endpoints exceed default body limit → `REQBODY_ERROR` | Global `SecRequestBodyLimit` via env `MODSEC_REQ_BODY_LIMIT` (NOT a per-rule ctl) |
| **Health** | `/health` doesn't need inspection | Engine off for `/health` |

## `-before` plugin: allowed methods

```apache
# app-config-before.conf  → mounted as owasp-crs/plugins/app-config-before.conf
SecAction \
    "id:1000,phase:1,nolog,pass,t:none,\
    setvar:'tx.allowed_methods=GET HEAD POST OPTIONS PUT PATCH DELETE'"
```

## `-after` plugin: path-scoped exclusions

```apache
# app-exclusions.conf  → mounted as owasp-crs/plugins/app-exclusions-after.conf

# Swagger / OpenAPI (keep these behind app auth regardless)
SecRule REQUEST_URI "@beginsWith /docs"         "id:1001,phase:1,pass,nolog,ctl:ruleEngine=Off"
SecRule REQUEST_URI "@beginsWith /redoc"        "id:1002,phase:1,pass,nolog,ctl:ruleEngine=Off"
SecRule REQUEST_URI "@beginsWith /openapi.json" "id:1003,phase:1,pass,nolog,ctl:ruleEngine=Off"

# Health check
SecRule REQUEST_URI "@streq /health"            "id:1004,phase:1,pass,nolog,ctl:ruleEngine=Off"

# Search endpoints: drop SQLi family ONLY here (adjust the regex to your routes)
SecRule REQUEST_URI "@rx ^/(api/)?(search|.*/search)" \
    "id:1005,phase:1,pass,nolog,ctl:ruleRemoveById=942000-942999"
```

Adjust the prefixes to the app's real routes (your API may or may not use an
`/api` prefix).

## Deriving exclusions from the API docs (preferred)

Ask the user for the API documentation and derive the exclusions from it rather
than guessing. In order of usefulness:

1. **OpenAPI / Swagger** — the ground truth. If the app exposes `openapi.json`,
   pull the exact paths, methods and which take free-text params:
   ```bash
   # list method + path from an OpenAPI doc
   curl -s http://APP/openapi.json | \
     python3 -c 'import sys,json; d=json.load(sys.stdin);
   [print(m.upper(), p) for p,ms in d["paths"].items() for m in ms]'
   ```
   Map that to exclusions: search/query endpoints → drop 942xxx on those paths;
   upload endpoints → raise body limit; the set of methods seen → `tx.allowed_methods`.
2. **Postman/Insomnia collection or an API reference** the user provides — same
   mapping, read the request definitions.
3. **Source routes** if no docs exist:
   ```bash
   grep -rn -E '@(app|router)\.(get|post|put|patch|delete|websocket)\(' --include="*.py" .
   ```

Write the exclusion regex to match the real prefixes you found. Keep each exclusion
commented with *which* endpoint and *why*, so it's auditable later.

## Important: exclusions are not a license to be unsafe

Dropping the SQLi family on a search path assumes the app **does not build SQL from
that input unsafely**. For FastAPI + an ORM / parameterized queries that's fine. If
the search path interpolates user input into raw SQL or a PostgREST `.or_()` filter,
fix that in the app first — the WAF exclusion would otherwise remove your only net.
(This is exactly the kind of filter-injection bug worth auditing before you exclude.)

## Hide stack version from unauthenticated users (defense in depth)

Independent of the WAF, FastAPI apps often leak the stack version to anyone:

- `/openapi.json` and Swagger expose the framework; keep them auth-gated.
- Custom `/version` or `/health` endpoints that return app/commit/build info should
  **require authentication** (or at least not echo versions to the public). Keep the
  public health check as minimal as possible — a bare `200` with plain `ok` (not
  even a JSON structure) is enough for an ALB/Coolify probe and leaks nothing. In
  FastAPI: `@app.get("/health", response_class=PlainTextResponse)` returning `"ok"`.
- The reverse proxy should send `Server: nginx` without a version (`server_tokens
  off` — the OWASP image already does this) and must not add `X-Powered-By`.
- Angular/SPA front-ends: the static `index.html` should not carry a build/version
  comment; `ng-version` only appears in the rendered DOM at runtime (low risk).
