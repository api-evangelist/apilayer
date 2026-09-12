---
name: apilayer-historical-fx-series
description: Pull a historical exchange-rate series, a single past day's rates, or the change
  between two dates from an APILayer foreign-exchange API.
generated: '2026-09-12'
method: generated
source: openapi/apilayer-exchangeratesapi-openapi.yml, openapi/apilayer-exchangerate-openapi.yml
api: Exchange Rates API / ExchangeRate.host API
operations:
  - exchangeratesapiTimeseries
  - exchangeratesapiFluctuation
  - exchangeratesapiHistorical
  - exchangeratesapiLatest
  - exchangeratehostTimeframe
  - exchangeratehostChange
  - exchangeratehostHistorical
  - exchangeratehostLive
---

# Historical FX series with APILayer

## Pick the right operation

| You want | Exchange Rates API (`https://api.exchangeratesapi.io/v1`) | ExchangeRate.host (`https://api.exchangerate.host`) |
|---|---|---|
| Today's rates | `exchangeratesapiLatest` → `GET /latest` | `exchangeratehostLive` → `GET /live` |
| One past day | `exchangeratesapiHistorical` → `GET /{date}` | `exchangeratehostHistorical` → `GET /historical?date=` |
| A date range, day by day | `exchangeratesapiTimeseries` → `GET /timeseries` | `exchangeratehostTimeframe` → `GET /timeframe` |
| Change between two dates | `exchangeratesapiFluctuation` → `GET /fluctuation` | `exchangeratehostChange` → `GET /change` |

Note the shape difference: Exchange Rates API puts the date in the **path**
(`GET /2024-03-01?access_key=…`), ExchangeRate.host puts it in a **query parameter**
(`GET /historical?date=2024-03-01&access_key=…`).

## Parameters

- `access_key` — required on every call, always in the query string.
- `start_date` / `end_date` — required on `timeseries` and `fluctuation`; both `YYYY-MM-DD`.
  On ExchangeRate.host `change` they are optional and default to the current period.
- Base and target currencies are named differently between the two products:
  - Exchange Rates API: `base` and `symbols` (comma-separated).
  - ExchangeRate.host: `source` and `currencies` (comma-separated).

  Getting these crossed is the most common failure. Check which host you are on first.

## Steps

1. Choose the host from the key you hold, then choose the operation from the table above.
2. Build the range. Keep it bounded — a wide range is one request but a large response, and
   `timeframe`/`timeseries` are gated to paid tiers on some plans
   (`function_access_restricted`).
3. Call, then check `success` in the body **before** reading `rates` or `quotes`.
4. Iterate ranges rather than days. One `timeseries` call over 30 days costs one request; 30
   `historical` calls cost 30.

## Errors

Same envelope and the same discipline as `apilayer-convert-currency`: HTTP 200 can carry a
failure, so branch on `success` and then `error.type`. Full table in
`errors/apilayer-error-codes.yml`.

Range-specific types to expect: `invalid_date` (bad or future date), and
`function_access_restricted` when the plan does not include time-frame queries.

## Cost and safety

Read-only GETs — nothing to reverse. No rate-limit headers are returned, and paid plans bill
per request past the quota, so bound your loop before you start it.
