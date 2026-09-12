---
name: apilayer-geolocate-ip
description: Look up location, connection, timezone, currency and threat data for an IP address
  — or for the caller's own IP — using the APILayer ipapi API.
generated: '2026-09-12'
method: generated
source: openapi/apilayer-ipapi-openapi.yml
api: ipapi
operations:
  - ipapiIPLookup
  - ipapiCheckRequesterIPLookup
---

# Geolocate an IP with APILayer ipapi

Base URL: `https://api.ipapi.com/api`

| Operation | Call | Use when |
|---|---|---|
| `ipapiIPLookup` | `GET /{IPAddress}` | You have an IP (IPv4 or IPv6) to look up. |
| `ipapiCheckRequesterIPLookup` | `GET /check` | You want the IP of the machine making the call. |

## Steps

1. **Call the lookup.** `access_key` is required and goes in the query string:

   ```
   GET /134.201.250.155?access_key=KEY
   ```

2. **Ask only for what you need.** `fields` takes a comma-separated list and trims the
   response. Two modules are opt-in and tier-gated:
   - `security=1` — threat/proxy/VPN assessment. Business Pro tier and above.
   - `hostname=1` — reverse hostname lookup.

   `language` takes a two-letter code; `output` takes `json` or `xml` (default `json`).
3. **Check `success` in the body before reading any field.**
4. For several addresses, use the bulk form rather than a loop — but keep the batch small:
   too many addresses in one request returns **HTTP 422** with `too_many_ips`.

## Errors

The envelope is `{"success": false, "error": {"code": …, "type": …, "info": …}}` and, as with
every APILayer product, **an authentication failure can arrive as HTTP 200** — observed on
`https://api.ipapi.com/api/check?access_key=x` on 2026-09-12. Branch on `success`, then on
`error.type`.

| `error.type` | Meaning |
|---|---|
| `invalid_access_key` / `missing_access_key` | Bad or absent key. Stop. |
| `function_access_restricted` | The module you asked for (usually `security`) is not on this plan. Drop the parameter or stop. |
| `too_many_ips` | Shrink the bulk batch. |
| `404_not_found` | The path parameter is not a valid address. |
| `usage_limit_reached` / `daily_usage_limit_reached` | Quota gone. Do not loop. |
| `rate_limit_reached` | Burst ceiling. Back off. |

## Privacy and cost

- An IP address is personal data in several jurisdictions. Look up only addresses you have a
  reason to resolve, and do not retain the enriched record longer than the task needs.
- The free plan is 100 lookups a month, non-commercial, and its feature matrix excludes the
  security module. Paid plans bill per lookup past the quota with **no rate-limit headers** to
  warn you first.
- Read-only: nothing to undo.
