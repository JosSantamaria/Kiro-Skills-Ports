# CRS tuning — how the image loads rules and how to exclude safely

## How the OWASP image includes rules

The `owasp/modsecurity-crs` image loads everything from `setup.conf`, in order:

```
modsecurity.conf                      # engine defaults (SecRuleEngine, body access…)
modsecurity-override.conf             # env-driven overrides
owasp-crs/crs-setup.conf              # CRS config (paranoia, anomaly thresholds)
owasp-crs/plugins/*-config.conf
owasp-crs/plugins/*-before.conf       # ← your "before" plugin runs HERE
owasp-crs/rules/*.conf                # the actual CRS rules
owasp-crs/plugins/*-after.conf        # ← your "after" plugin runs HERE
```

Consequences:
- **Config that rules read** (e.g. `tx.allowed_methods`) must be in a `*-before.conf`.
- **Exclusions that reference rule IDs** (`ctl:ruleRemoveById`) must be in a
  `*-after.conf`, because the rules must exist first.
- A plain `/custom` mount is **not** auto-included — mount under `owasp-crs/plugins/`.

## Key environment variables

| Env | Meaning |
| --- | --- |
| `BACKEND` | Upstream the WAF proxies to, e.g. `http://app:80` |
| `PROXY_SSL` | `off` when TLS is terminated upstream (LB/ALB) and the backend is HTTP |
| `PORT` / `SSL_PORT` | Listen ports; image runs unprivileged so defaults are `8080`/`8443` |
| `MODSEC_RULE_ENGINE` | `Off` / `DetectionOnly` / `On` |
| `PARANOIA`, `BLOCKING_PARANOIA` | CRS paranoia level 1–4 (higher = stricter, more FPs) |
| `ANOMALY_INBOUND`, `ANOMALY_OUTBOUND` | Anomaly-scoring thresholds |
| `MODSEC_REQ_BODY_LIMIT` | Global request body limit (bytes) — use this, not a per-rule ctl |
| `MODSEC_AUDIT_ENGINE` | `On` / `RelevantOnly` / `Off` |
| `MODSEC_AUDIT_LOG_FORMAT` | `JSON` (parse-friendly) or `Native` |
| `MODSEC_AUDIT_LOG_TYPE` | `Serial` or `Concurrent` (prefer Concurrent, see below) |
| `MODSEC_AUDIT_STORAGE_DIR` | Dir for concurrent audit entries |
| `REAL_IP_FROM`, `REAL_IP_HEADER` | Trust the edge proxy for the real client IP |

## Writing exclusions correctly

- **By path + family:**
  ```apache
  SecRule REQUEST_URI "@rx ^/api/search" \
      "id:1005,phase:1,pass,nolog,ctl:ruleRemoveById=942000-942999"
  ```
- **By single rule ID** (after you identify a specific noisy rule in the audit log):
  ```apache
  SecRule REQUEST_URI "@beginsWith /api/import" \
      "id:1010,phase:1,pass,nolog,ctl:ruleRemoveById=920420"
  ```
- **Turn engine off for a path** (only for truly safe, non-user-data routes like
  docs/health):
  ```apache
  SecRule REQUEST_URI "@streq /health" "id:1004,phase:1,pass,nolog,ctl:ruleEngine=Off"
  ```

### ModSecurity v2 vs v3 — do not mix

- `ctl:requestBodyLimit=...` is **v2 only**. In v3 (libmodsecurity, what the nginx
  connector uses) it is an invalid action and will abort rule loading with
  `Expecting an action, got: ctl:requestBodyLimit`. Use `SecRequestBodyLimit`
  (global) via `MODSEC_REQ_BODY_LIMIT`.
- Pick a unique `id:` for every custom rule; collisions also abort loading.

## IP allow / deny (optional)

```apache
# Deny a range outright (phase 1, before scoring)
SecRule REMOTE_ADDR "@ipMatch 203.0.113.0/24" \
    "id:1100,phase:1,deny,status:403,log,msg:'Blocked IP range'"

# Allow-list bypass (skip the engine for trusted ranges)
SecRule REMOTE_ADDR "@ipMatch 10.0.0.0/8" \
    "id:1101,phase:1,pass,nolog,ctl:ruleEngine=Off"
```

Validate CIDR values before writing them into a rule if they come from user input.
