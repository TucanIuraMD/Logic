# Database (external component)

> The database is a **separate component** and is not part of this repository.
> This page is a safe interface description; detailed schema notes live in
> **WIKI-LLM/docs/DATABASE.md**.

## Contract

The application requires an external MySQL database that has already been
provisioned. It connects with a compile-time connection string:

```
server=<DB_HOST>;userid=<DB_USER>;password=<DB_PASSWORD>;database=<DB_NAME>;
CharSet=utf8mb4;CheckParameters=false;
```

Connection parameters required:

| Parameter | Notes |
|-----------|-------|
| host / port | typically `127.0.0.1` / 3306 |
| user | dedicated application user |
| password | provided via secure configuration, not committed |
| database | schema name (e.g. `cashin`) |
| charset | `utf8mb4` |

## What the application expects

- **settings** table + section model — read by `SqlSettingsProvider`.
- **keys** table — RSA `Public`/`Secret` and channel password material.
- **devices / properties4devices** — peripherals and COM parameters.
- **payments, money, money_history, balance** — ledger.
- **messages** family — monitoring queue.
- reference tables: services, operators, requisites, limitations, states,
  languages, tasks, etc.
- stored procedures/functions used by the DAL (savePayment, addMoney,
  get_balance, get_settings, settings store, limitation/requisite helpers…).

## Dependencies

- All monetary amounts are decimal.
- Payment statuses are integers; `payments.sent` flags outgoing payments.
- RSA keys stored as UTF-8 bytes of .NET RSA XML.

## Notes for integrators

- Schema/data migrations are applied by the operator before first run.
- Test key material must never be used in production.
- The test-only database `cashin_test` is not needed by production.
