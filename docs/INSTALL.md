# Install (outline)

> Detailed technical documentation: see **WIKI-LLM/docs/** at
> <https://github.com/TucanIuraMD/WIKI-LLM>. This file is a repository-scoped
> outline only. No binaries are committed in this repository.

## Target environment

- Ubuntu 24.04 LTS (x86_64)
- Mono 6.8 runtime packages
- external MySQL 8.x (provisioned separately)
- service user with `dialout` membership for serial devices

## Steps

1. **Database (external component)**

   Provision the MySQL schema/data outside this repository. The application
   connects to an already-prepared database (`docs/DATABASE.md`).

2. **Runtime files**

   Deploy the verified Linux build (Logic.exe + Linux-port assemblies + third
   party libraries) into the application directory (e.g. `/opt/logic/run`).
   See the artifact/hash notes in `WIKI-LLM/docs/LINUX_PORT.md`.

3. **Configuration**

   - Set settings in MySQL (`MonitorURL`, `PaymentUri`, terminal ids, device
     names).
   - Install production RSA keys into the `keys` table (never test keys).
   - Use `config.example.env` / `config.example.json` as templates for local
     overrides; never commit real values.

4. **Serial devices**

   - CashCode CCNet on `/dev/ttyS0`
   - Citizen PPU 700 on `/dev/ttyS1`
   - udev: `MODE="0660" GROUP="dialout"`; service user in `dialout`.

5. **Run**

   ```
   /usr/bin/mono /opt/logic/run/Logic.exe
   ```

   The application reaches the "Logis is STARTED" state, connects to monitoring
   and payment servers, and starts its periodic tasks (GetInfo).

## Post-install checks

- `sha256sum` of the deployed MainLogic.dll matches the verified hash
  (see `WIKI-LLM/docs/LINUX_PORT.md`).
- Startup log shows monitoring messages sent successfully.
- The Info task (GetInfo) succeeds against the configured payment server.
- E2E/dev regression documented in `WIKI-LLM/docs/TESTING.md`.
- Hardware acceptance on real CashCode/Citizen devices is a separate step.
