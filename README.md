# Logic Linux

Linux/Mono port of a self-service payment kiosk terminal application.

The terminal accepts banknotes (CashCode/CCNet), prints receipts
(Citizen PPU 700), talks to a monitoring server and a payment server, and keeps
its ledger in an external MySQL database.

> **Full technical documentation lives in the companion wiki:
> <https://github.com/TucanIuraMD/WIKI-LLM>** (see `WIKI-LLM/docs/`).
> This repository contains the program-related material: safe configuration
> examples and minimal, repository-scoped docs. It does **not** contain the
> database, secrets, or commercial source material.

## Repository contents

```
Logic/
├── README.md             ← this file
├── .gitignore
├── config.example.env    ← safe configuration template (placeholders only)
├── config.example.json   ← safe JSON configuration template (placeholders only)
└── docs/
    ├── INSTALL.md        ← install outline (Ubuntu + Mono + external MySQL)
    ├── CONFIGURATION.md  ← settings and configuration model
    └── DATABASE.md       ← external MySQL contract (no secrets)
```

## Requirements (outline)

- Ubuntu 24.04 LTS, Mono 6.8 runtime
- external MySQL 8.x database, provisioned separately (see `docs/DATABASE.md`)
- serial peripherals: CashCode CCNet (`/dev/ttyS0`) and Citizen PPU 700
  (`/dev/ttyS1`) with `dialout` permissions

## Quick start (outline)

1. Provision the external MySQL schema and data (outside this repo).
2. Install runtime files (the verified Linux build) into the app directory.
3. Configure settings in MySQL and copy `config.example.*` as local override.
4. Run the entry assembly under Mono with the kiosk display available.

See `docs/INSTALL.md` and `docs/CONFIGURATION.md` for details.

## Verification

The maintained test suite (DB, devices, network, cash-in, printer, E2E) is
documented in `WIKI-LLM/docs/TESTING.md`. Hardware acceptance on real devices is
a separate step and was not performed in the build environment.

## License / provenance

This repository is published as a **Linux-port derivative** of the terminal
application for maintenance purposes. Original commercial Windows sources,
database dumps, real credentials and production keys are intentionally **not**
included. No binaries are committed here.
