# UI management — tune the WAF from an admin panel, safely

Optional, advanced. Only build this if the user wants to change WAF settings from an
app UI instead of editing files and recreating the container. The goal: admins
adjust engine mode / paranoia / exclusions / IP lists from the UI, applied live and
persistently, **without new security holes**.

## Non-negotiable rules

1. **Never write to the server `.env` from a UI/endpoint.** `.env` is read once at
   boot and is lost on redeploy of ephemeral containers (Coolify, etc.). Writing
   secrets/config to it from a request is an attack surface. Store config in the
   app's database instead.
2. **Admin-only.** Gate every WAF endpoint with both an RBAC permission AND an
   explicit role check (`role == admin`). Return a generic 403 otherwise.
3. **Validate everything written to a rule file.** Config from the DB that ends up
   in a ModSecurity directive is untrusted — whitelist enums, range-check ints,
   regex-restrict path patterns, validate CIDRs. Prevent directive injection.
4. **Don't mount the Docker socket** into the app to reload the WAF — that is a
   privilege-escalation risk. Use an in-container file watcher instead (below).

## Architecture

```
Admin UI ──PUT /api/waf/config (RBAC + admin guard)──► Backend
                                                         │ persist to DB (waf_settings…)
                                                         │ render DB → CRS plugin .conf files
                                                         ▼ write to shared volume
                                               WAF container (inotify watcher) → nginx -s reload
```

- **DB tables**: `waf_settings` (engine_mode, paranoia_level, anomaly_threshold),
  `waf_rule_exclusions` (path_pattern, excluded_rule_ids), `waf_ip_rules`
  (ip_cidr, allow/deny). The DB is the source of truth → survives redeploys.
- **Render step**: a backend service turns the rows into `*-before.conf` /
  `*-after.conf` on a shared volume mounted by the WAF.
- **Reload**: a tiny `inotifywait` loop inside the WAF container watches the volume
  and runs `nginx -t && nginx -s reload` on change. If `nginx -t` fails, keep the
  previous config and report the error back to the UI — never leave the WAF down.

## Endpoints (shape)

| Method | Route | Auth | Purpose |
| --- | --- | --- | --- |
| GET | `/api/waf/config` | admin | current settings + exclusions + IP rules |
| PUT | `/api/waf/config` | admin | validate → persist → render → reload |
| GET | `/api/waf/events` | admin | recent audit-log entries (parsed JSON) |
| GET | `/api/waf/status` | admin | engine mode, CRS version, last reload |

Log every change to the app's audit trail (who, what, when).

## UI (admin-only)

- Engine mode: Off / DetectionOnly / On — require explicit confirmation for `On`.
- Paranoia (1–4) + anomaly threshold.
- Exclusions table: path + rule IDs (seeded with the stack's base exclusions).
- IP allow/deny with client-side CIDR validation.
- Event viewer: latest fired rules (rule, IP, path, action) from `/events`.
- "Apply & reload" button → `PUT /config`, show success or validation error.
- Hide the whole section from non-admins (route guard + conditional nav).

## Relationship to weak perimeter auth (e.g. npm basic auth)

If the app currently sits behind only nginx/npm **basic auth** exposed to the
internet: basic auth answers *who may connect*; the WAF answers *what they may
send*. Keep both. Put the WAF so that even an authenticated user's request body is
inspected. The admin WAF panel then lets you tighten rules without redeploying.
