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

## 7. Autonomous UI validation with Playwright (AI-driven)

`curl` smoke tests prove the HTTP paths, but they don't prove the **real UI still
works** through the WAF — a too-aggressive CRS ruleset can break form posts, file
uploads, websockets or XHR calls that only a browser exercises. Use a browser
automation agent (Playwright) so an AI can navigate the app autonomously and
validate behavior end-to-end, before and after flipping to blocking mode.

Why it fits WAF work specifically:
- The WAF inspects **request bodies and headers** that a browser sends but a simple
  `curl` may not reproduce (CSRF tokens, multipart uploads, JSON XHR, cookies).
- Running the same browser flow in `DetectionOnly` vs `On` surfaces exactly which
  user action a rule would block — turning a vague "something broke" into a concrete
  path + payload you can write a scoped exclusion for (`crs-tuning.md`).
- It's repeatable: keep the flow as a script/checklist and re-run it on every CRS
  change as a regression gate.

### Decision point — ask before automating (volume-based)

This step is **optional**. Offer it, don't impose it. Decide with the user based on
how much changed:

- **Small change** (one or two scoped exclusions, a single route touched): the
  `curl` smoke test + a quick manual click-through is usually enough. Ask: *"the
  change is small — want a quick manual check, or should I run the automated
  Playwright validation anyway?"*
- **Large change / high volume** (many exclusions added, paranoia level raised,
  switching `DetectionOnly → On`, or many routes affected): **recommend the AI-driven
  Playwright validation**. Ask: *"this touched a lot — I recommend running the
  automated browser validation so we catch any flow the WAF breaks before prod. Run
  it?"* A good heuristic to trigger the recommendation:
  - more than ~3 exclusions changed, OR
  - any paranoia/anomaly-threshold change, OR
  - promoting the engine to `On`, OR
  - the app has forms/uploads/websockets you can't fully cover with `curl`.

Always let the user decline; record that validation was skipped so it can be run
later.

### Which tool to use

Pick based on the IDE/environment, in this order of preference:

1. **If the IDE is Kiro: prefer the `kiro-webwright` power** (the Kiro port at
   `github.com/JosSantamaria/Kiro-Skills-Ports`). It's the strongest fit — a
   terminal-native web agent that plans, discovers selectors, writes a **reusable
   `final_script.py`**, and self-verifies with screenshot evidence. Ideal for a
   regression validator you re-run on every CRS change and for the
   `DetectionOnly` vs `On` diff. Install it from that repo if not present.
2. **An MCP Playwright server** (bundled in some powers; may be disabled by default
   — enable it in that power's `mcp.json`) — good for *ad-hoc exploration*:
   low-level navigate/click/snapshot driven turn by turn, great for discovering a
   selector or inspecting one failure, but it doesn't leave a reusable script.
3. **A standalone Playwright script** (Python or JS, below) when no agent/MCP is
   wired up — `pip install playwright && playwright install chromium`.

Rule of thumb: on Kiro, use `kiro-webwright` for the keep-it validator and the MCP
Playwright server for quick pokes; they're complementary. If only one is available,
use it.

### Complement with `curl` (fast, no browser)

Before (or alongside) a browser run, `curl` covers a lot cheaply and is great for
scripted gates. Techniques that work well against a WAF:

```bash
BASE=http://localhost:8080
# 1. Get a token (same-origin API login) and walk the API with it
TOKEN=$(curl -s -X POST $BASE/api/auth/login -H 'Content-Type: application/json' \
         -d '{"username":"admin","password":"admin123"}' | python3 -c 'import sys,json;print(json.load(sys.stdin).get("token",""))')
for ep in /api/auth/me /api/accounts/ "/api/search?q=prod-01"; do
  printf "%-32s %s\n" "$ep" "$(curl -s -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOKEN" "$BASE$ep")"
done

# 2. Confirm security headers survive the WAF hop
curl -s -D - -o /dev/null $BASE/ | grep -iE 'x-frame|x-content-type|content-security|server'

# 3. Prove the WAF inspects without blocking (DetectionOnly): diff the audit-log
#    entry count around an attack payload. A delta > 0 means it evaluated & logged.
before=$(docker compose exec -T waf sh -c 'find /var/log/modsecurity/audit -type f | wc -l')
curl -s -o /dev/null "$BASE/api/accounts/x?f=1%27%20UNION%20SELECT%20NULL--"
after=$(docker compose exec -T waf sh -c 'find /var/log/modsecurity/audit -type f | wc -l')
echo "audit delta: $((after - before))   # >0 ⇒ WAF saw it"
```

Note a non-403 after an attack in DetectionOnly is expected — the status you see is
the backend's (e.g. `401`/`405`), not a WAF block; the audit-log delta is what proves
the WAF engaged. `curl` can't drive JS/forms/uploads, so pair it with the browser
step for anything the UI does beyond plain requests.

### How to use it

Drive the chosen tool with a checklist like:

1. Open the app through the WAF entrypoint (e.g. `http://localhost:8080`).
2. Log in (or pass the perimeter basic-auth / app auth), confirm the dashboard loads.
3. Exercise the risky flows the WAF is most likely to flag:
   - a search box with special characters,
   - a create/update form (PUT/PATCH),
   - a file upload,
   - any endpoint with a large JSON body.
4. Capture a screenshot + the network results at each step as evidence.
5. Diff the run between `DetectionOnly` and `On`: any step that changes from success
   to a `403`/error under `On` is a false positive to exclude.

Standalone Python Playwright example (tested; good for SPAs — uses `page.request`
for API probes and captures any 403 the WAF would raise):

```python
# .venv/bin/python nav_check.py   (pip install playwright && playwright install chromium)
from playwright.sync_api import sync_playwright
BASE, USER, PASS = "http://localhost:8080", "admin", "admin123"

blocked = []
with sync_playwright() as p:
    b = p.chromium.launch(headless=True)
    page = b.new_context(viewport={"width": 1280, "height": 1800}).new_page()
    page.on("response", lambda r: blocked.append((r.status, r.url))
            if r.status == 403 or r.status >= 500 else None)
    page.set_default_timeout(15000)

    page.goto(BASE, wait_until="domcontentloaded")
    page.screenshot(path="01_home.png")

    # same-origin API login → token
    r = page.request.post(f"{BASE}/api/auth/login",
                          data={"username": USER, "password": PASS},
                          headers={"Content-Type": "application/json"})
    token = (r.json() or {}).get("token", "") if r.ok else ""
    auth = {"Authorization": f"Bearer {token}"} if token else {}

    # exercise the risky flows (adjust to the app's real routes)
    for path in ["/api/search?q=web01", "/api/search?q=o'brien & co *",
                 "/api/dashboard/summary", "/api/accounts/"]:
        print(path, "->", page.request.get(f"{BASE}{path}", headers=auth).status)
    b.close()

print("403/5xx captured:", blocked)   # empty in DetectionOnly on legit traffic
```

After the run, cross-check the WAF audit log (step 5) to see *which rules fired* even
when nothing was blocked — that's your false-positive triage list for when you flip
to `On`. Rules firing on `/foo`-style throwaway attack paths are your own probes, not
app traffic; ignore those.

Minimal standalone Playwright example (JS, if you prefer `@playwright/test`):

```javascript
// npx playwright test  — smoke-validate the app behind the WAF
const { test, expect } = require('@playwright/test');
const BASE = process.env.BASE || 'http://localhost:8080';

test('app works through the WAF', async ({ page }) => {
  const blocked = [];
  page.on('response', r => { if (r.status() === 403) blocked.push(r.url()); });

  await page.goto(BASE);
  // login flow (adjust selectors to the app)
  await page.fill('[name=username]', process.env.USER || 'admin');
  await page.fill('[name=password]', process.env.PASS || 'admin123');
  await page.click('button[type=submit]');
  await expect(page).toHaveURL(/dashboard|home|\/$/);

  // exercise a risky flow (search with special chars)
  await page.goto(`${BASE}/search?q=${encodeURIComponent("o'brien & co *")}`);
  await page.screenshot({ path: 'waf-search.png', fullPage: true });

  // any 403s captured are WAF blocks to triage (should be none for legit flows)
  expect(blocked, `WAF blocked legit requests: ${blocked.join(', ')}`).toHaveLength(0);
});
```

Treat the list of `403` URLs as the triage queue: for each legitimate one, add a
path-scoped exclusion, reload, and re-run the Playwright flow until it's clean. Only
then promote `MODSEC_RULE_ENGINE` to `On` in the real deployment.

> Note on health checks: keep the public `/health` endpoint minimal (a bare `200`
> with `ok`, no version/build/JSON structure) so it can't be used to fingerprint the
> stack. Playwright/AI validation covers "does the app actually work"; the health
> endpoint only needs to answer "is it up".
