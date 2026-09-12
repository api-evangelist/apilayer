---
name: apilayer-convert-currency
description: Convert an amount between two currencies, at today's rate or at a historical date,
  using an APILayer foreign-exchange API.
generated: '2026-09-12'
method: generated
source: openapi/apilayer-exchangeratesapi-openapi.yml, openapi/apilayer-exchangerate-openapi.yml
api: Exchange Rates API / ExchangeRate.host API
operations:
  - exchangeratesapiConvert
  - exchangeratehostConvert
  - exchangeratesapiSymbols
---

# Convert currency with APILayer

APILayer runs two interchangeable FX products. Use whichever the caller holds a key for — the
parameters are nearly identical and the responses are the same shape.

| Product | Base URL | Convert operation |
|---|---|---|
| Exchange Rates API | `https://api.exchangeratesapi.io/v1` | `exchangeratesapiConvert` → `GET /convert` |
| ExchangeRate.host | `https://api.exchangerate.host` | `exchangeratehostConvert` → `GET /convert` |

## Before you call

- The credential is the **`access_key` query parameter**. There is no header form. It is
  product-specific: a key for Exchange Rates API will not work on ExchangeRate.host.
- Keys and plans are per product. Check `plans/apilayer-plans-pricing.yml` — the free plan is
  **100 requests a month** and is non-commercial.
- If you need to know which currency codes are valid, call `exchangeratesapiSymbols`
  (`GET /symbols`) first. Do not guess a code.

## Steps

1. **Resolve the currencies.** `from` and `to` take ISO 4217 codes. Call `GET /symbols` once and
   cache it if the caller supplied currency *names* rather than codes.
2. **Call convert.** All three of `from`, `to` and `amount` are **required**:

   ```
   GET /convert?access_key=KEY&from=USD&to=EUR&amount=100
   ```

   Add `date=YYYY-MM-DD` to convert at a historical rate instead of the latest one.
3. **Check `success` before reading the result.** See "Reading the response" below.
4. **Read the converted value** from `result`; the rate used is under `info`.

## Reading the response — this API lies with status codes

**Do not branch on the HTTP status.** APILayer returned **HTTP 200** with an error body for an
invalid access key on every host probed. The contract documents 401/403/404/429; the deployed
surface does not always send them.

Branch on the body instead:

```json
{ "success": false, "error": { "code": 101, "type": "invalid_access_key", "info": "..." } }
```

- `success: true` → read `result`.
- `success: false` → look at **`error.type`**, never `error.code` alone. The codes are reused:
  `104` means `api_access_blocked` in one place and `usage_limit_reached` in another.

Types you must handle, from `errors/apilayer-error-codes.yml`:

| `error.type` | What to do |
|---|---|
| `missing_access_key`, `invalid_access_key` | Stop. Ask for a valid key for *this* product. |
| `https_access_restricted` | The plan does not include HTTPS. Do not silently downgrade to HTTP — the key would travel in cleartext. Stop and report. |
| `function_access_restricted` | The endpoint needs a higher tier. Stop. |
| `invalid_from_currency`, `invalid_to_currency`, `invalid_conversion_amount`, `invalid_date` | Fix the input and retry once. |
| `rate_limit_reached` | Burst ceiling. Back off and retry. |
| `usage_limit_reached`, `daily_usage_limit_reached` | Quota gone. **Do not retry in a loop** — on a paid plan, calls past the quota bill per request. |
| `internal_error`, `maintenance_mode` | Retry with backoff. |

## Cost and safety

- There are **no rate-limit response headers**. You cannot see remaining quota before you spend
  it; the first sign of exhaustion is the error body.
- Paid plans **meter overage per request** rather than cutting off, so an unbounded retry loop
  spends real money. Cap your attempts.
- Every operation is a `GET` and changes nothing on the provider side, so there is nothing to
  undo and no idempotency key to send.
