# Verification — smoke tests, proving the WAF works, reading the audit log

Run these after Phase 1 (insert) and again after Phase 4 (block).

## 1. Compose validity + bring up

```bash
docker compose config >/dev/null && echo "compose OK"
docker compose up --build -d
docker compose ps        # both app and waf should become healthy
```

## 2. Rules loaded without errors

```bash
docker compose logs waf | grep -iE "rules loaded|emerg|error"
# Expect:  ModSecurity-nginx ... (rules loaded inline/local/remote: 0/NNN/0)
# NNN = CRS rules + your custom rules. No [emerg]/Rules error lines.
```

A `Rules error ... Expecting an action` almost always means a v2-only `ctl` action
(e.g. `ctl:requestBodyLimit`) or a bad rule id. Fix the plugin file and recreate.

## 3. Legitimate traffic passes through the WAF

```bash
BASE=http://localhost:8080
curl -s -o /dev/null -w "health=%{http_code}\n" $BASE/health
curl -s -o /dev/null -w "home=%{http_code}\n"   $BASE/
# App-specific legit calls (adjust): login, a normal search, etc.
curl -s -o /dev/null -w "login=%{http_code}\n" -X POST $BASE/api/auth/login \
     -H "Content-Type: application/json" -d '{"username":"u","password":"p"}'
```

All should return their normal codes (200 / 401 for bad creds / etc.) — proving the
proxy path `waf → app` works.

## 4. Prove the engine actually evaluates (binary test)

In DetectionOnly, attacks are **not** blocked (they reach the backend), so a non-200
is the app's doing, not the WAF. To prove ModSecurity evaluates, run a throwaway
instance in blocking mode:

```bash
docker compose run --rm -e MODSEC_RULE_ENGINE=On -e BACKEND=http://app:80 \
    -e PORT=8080 -p 8099:8080 -d --name waf-test waf
sleep 12
curl -s -o /dev/null -w "home=%{http_code}\n" http://localhost:8099/
curl -s -o /dev/null -w "sqli=%{http_code}\n" \
    "http://localhost:8099/foo?id=1%27%20OR%201=1%20UNION%20SELECT%20NULL--%20-"
curl -s -o /dev/null -w "xss=%{http_code}\n" \
    "http://localhost:8099/foo?x=%3Cscript%3Ealert(1)%3C%2fscript%3E"
docker rm -f waf-test
# Expect: home=200, sqli=403, xss=403  → engine confirmed working
```

## 5. Read the audit log (false-positive triage)

With `MODSEC_AUDIT_ENGINE=On`, `MODSEC_AUDIT_LOG_TYPE=Concurrent`,
`MODSEC_AUDIT_LOG_FORMAT=JSON`:

```bash
# how many entries
docker compose exec -T waf sh -c 'find /var/log/modsecurity/audit -type f | wc -l'
# inspect one entry (uri + which rules fired)
docker compose exec -T waf sh -c \
  'f=$(find /var/log/modsecurity/audit -type f | head -1); head -c 800 "$f"'
```

Each JSON entry has `transaction.request.uri` and the messages/rule IDs that
matched. For every rule that fired on a **legitimate** request, add a path-scoped
exclusion (see `crs-tuning.md`) — then re-observe.

If the audit log is empty but you expected hits, check the stderr stream too
(ModSecurity-nginx writes rule messages to nginx `error_log` → container stderr):

```bash
docker logs <waf-container> 2>&1 | grep -iE "ModSecurity|Matched|id \"9"
```

## 6. Rollback check

Confirm you can disable the WAF without downtime:

```bash
# soft: stop blocking
docker compose run --rm -e MODSEC_RULE_ENGINE=Off ... # or set env and recreate
# hard: remove the waf service and re-expose the app port in compose
```
