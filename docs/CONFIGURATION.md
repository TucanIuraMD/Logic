# Configuration (outline)

> Full configuration reference: see **WIKI-LLM/docs/** (CONFIGURATION,
> DATABASE, DEVICES, DEPLOYMENT). This page is repository-scoped and contains
> no secrets.

## Configuration sources

| Source | Priority | Contents |
|--------|----------|----------|
| MySQL `cashin.settings` | highest | terminal settings: monitoring/payment endpoints, terminal ids, device names, flags |
| Code defaults (`Setting.cs`) | fallback | default values when not present in DB |
| `.config` files | startup | assembly binding redirects / framework |
| compile-time values | runtime | DB connection string, WebSocket UI port, serial defaults |

## Settings model

Settings are key/value rows in the external MySQL database (`_name`, `_value`)
grouped in sections, exposed to the application through `SqlSettingsProvider`
(`IBP.Setting`). Key groups:

- **Network**: `MonitorURL`, `MonitorPort`, `PaymentUri`, `UseProxy`,
  `ProxyName`, `ProxyPort`, `HttpTimeOut`.
- **Terminal identity**: `PointId`, `ClientId`, `SerialPoint`,
  `SerialClient`, `ClientName`.
- **Devices**: `CashCodeName`, `PrinterName`, device parameters in
  `properties4devices` (COM port, baud rate, parity, encoding).
- **UI**: `Design` (WebSocket), `ScreenSize`, `StartStep`.
- **Behaviour flags**: `LogRequest`, blocks, cheque options, etc.

Values not present in the DB fall back to code defaults.

## Security rules

- Never store real credentials in committed files. Templates:
  `config.example.env`, `config.example.json` (placeholders only).
- The DB connection string is compile-time; align MySQL account with it or
  rebuild (see `WIKI-LLM/docs/DATABASE.md`).
- `LogRequest=false` in production (it would log full payment XML).
- Use provider-issued RSA keys in production (see `WIKI-LLM/docs/DATABASE.md`).

## Local overrides

Copy `config.example.env` to `.env` (or `config.example.json` to
`config.local.json`) and fill values. Such files are git-ignored.
