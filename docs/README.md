# Logic Linux — Technical Documentation

Complete technical documentation for the Logic Linux project (self-service payment
kiosk terminal, ported from Windows to Linux/Mono).

> This documentation was created during the Linux port and captures architecture,
> migration history, component details, testing, deployment, and troubleshooting
> knowledge. All sensitive data (IPs, credentials, keys) has been replaced with
> placeholders.

## Quick navigation

### Core documents

| Document | Description |
|----------|-------------|
| [LLM_CONTEXT.md](LLM_CONTEXT.md) | **Main entry point** — project rules, status, baseline |
| [PROJECT_OVERVIEW.md](PROJECT_OVERVIEW.md) | What Logic Linux is, history, current state |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Component map, data flows, processes |

### Linux port & migration

| Document | Description |
|----------|-------------|
| [LINUX_PORT.md](LINUX_PORT.md) | Windows → Linux migration summary, what changed |
| [ORIGINAL_WINDOWS_VERSION.md](ORIGINAL_WINDOWS_VERSION.md) | Original Windows baseline |
| [SOURCE_TREE.md](SOURCE_TREE.md) | Source code structure |
| [BINARY_ARTIFACTS.md](BINARY_ARTIFACTS.md) | DLL inventory, verified hashes |
| [BUILD_REBUILD.md](BUILD_REBUILD.md) | Build toolchain, rebuild notes |
| [CHANGELOG.md](CHANGELOG.md) | Migration changelog |

### Database & data layer

| Document | Description |
|----------|-------------|
| [DATABASE.md](DATABASE.md) | MySQL schema, tables, stored procedures, DAL |
| [CONFIGURATION.md](CONFIGURATION.md) | Settings model, sources, keys |
| [RSA_KEYS.md](RSA_KEYS.md) | RSA key format, storage, usage |

### Devices & hardware

| Document | Description |
|----------|-------------|
| [DEVICES.md](DEVICES.md) | DeviceCenter, device factory, COM layer |
| [CASHCODE.md](CASHCODE.md) | CashCode CCNet bill acceptor |
| [PRINTER.md](PRINTER.md) | Citizen PPU 700 thermal printer |

### Network protocols

| Document | Description |
|----------|-------------|
| [PAYMENT.md](PAYMENT.md) | Payment protocol (HTTP POST, signed frames) |
| [MONITORING.md](MONITORING.md) | Monitoring protocol (TCP) |
| [INFO_XML_AUDIT.md](INFO_XML_AUDIT.md) | Info/InfoShort flow analysis |
| [INFO_PAYMENT_FAILURE_AUDIT.md](INFO_PAYMENT_FAILURE_AUDIT.md) | Payment failure investigation & fixes |

### Testing & deployment

| Document | Description |
|----------|-------------|
| [TESTING.md](TESTING.md) | Test executables, mock servers, E2E |
| [DEPLOYMENT.md](DEPLOYMENT.md) | Production deployment outline |
| [PRODUCTION_DEPLOYMENT_CHECKLIST.md](PRODUCTION_DEPLOYMENT_CHECKLIST.md) | Production key installation checklist |
| [HARDWARE_ACCEPTANCE_RUNBOOK.md](HARDWARE_ACCEPTANCE_RUNBOOK.md) | Hardware acceptance procedure |

### Diagnostics & operations

| Document | Description |
|----------|-------------|
| [TROUBLESHOOTING.md](TROUBLESHOOTING.md) | Common symptoms, causes, solutions |
| [SECURITY.md](SECURITY.md) | Security findings, credentials, hardcoded values |
| [KNOWN_LIMITATIONS.md](KNOWN_LIMITATIONS.md) | Production blockers, toolchain limits |

### Validation & completeness

| Document | Description |
|----------|-------------|
| [WIKI_VALIDATION.md](WIKI_VALIDATION.md) | Documentation completeness check |

## Reading order (for newcomers)

1. **LLM_CONTEXT.md** — understand the project, rules, baseline
2. **PROJECT_OVERVIEW.md** — what it is, history, status
3. **ARCHITECTURE.md** — component map
4. **LINUX_PORT.md** — what changed in the migration
5. Topic-specific documents as needed

## Document conventions

- **Placeholders** replace sensitive data:
  - `<DB_USER>`, `<DB_PASSWORD>` — database credentials
  - `<MONITORING_SERVER>`, `<PAYMENT_SERVER>`, `<PROXY_SERVER>` — network endpoints
  - `127.0.0.1`, `example.local` — generic addresses
- **Test-only** annotations mark materials not for production
- **Verified hashes** anchor critical binaries (MainLogic.dll)

## External resources

- Main program repository: **Logic.git** (this repository)
- AI-WIKI companion: [WIKI-LLM.git](https://github.com/TucanIuraMD/WIKI-LLM)
  (additional AI-focused documentation)
