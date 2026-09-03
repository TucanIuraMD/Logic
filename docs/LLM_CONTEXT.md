# Logic Linux — LLM Context (Main Entry)

## 1. Краткое описание проекта

**Logic Linux** — порт Windows-терминала самообслуживания (киоск приёма платежей) на Linux (Ubuntu 24.04 LTS, Mono 6.8).  
Основное приложение: `Logic.exe` — C# .NET Framework 4.6.1, сборка AnyCPU/PE32, запускается под Mono.

Исходный проект: `LogicLinux` — репозиторий Windows-разработчика (hilov), содержащий исходники, адаптированные для Linux.  
Текущая рабочая версия: `/root/Logic/MonoDebugTemp/` — проверенные binary artifacts.  
Исходники портирования: `/root/Logic/_audit/app/logiclinux-master/`.

## 2. Архитектурная цепочка

```
Logic.exe (entry point, winforms, WebSocket UI)
  └── MainLogic.dll (v7.0.0.0, бизнес-логика, сценарии, emonitor, device init)
        ├── CommonLib.dll (модели: Payment, Card, ServiceTerminal, ExternalProcess, Logger)
        ├── DataBase.dll (MySQL DAL: SavePayment, AddMoney, GetBalance, settings, keys, devices)
        │     └── MySql.Data.dll
        ├── DeviceCenter.dll (фабрика устройств: BaseFactory, PrinterDevice, BillAcceptor)
        │     └── Devices.dll (драйверы: COMPortWrapper, CashCode, Printer)
        │           ├── CashCodeWrapper.dll (CCNet протокол)
        │           └── Printer.dll (модели принтеров: Citizen, PrinterEMPTY)
        ├── LoggerLib.dll (логирование в файл + stdout)
        ├── SqlSettingsProvider.dll (читает настройки из MySQL `cashin.settings`)
        ├── SinkLib.dll (HTTP-клиент для платежного протокола: Formatter, Frame, Class607)
        ├── Protocol.dll (TCP-клиент мониторинга: ClientFormatter, Package, мониторинговые сообщения)
        ├── SettingParser.dll (парсинг XML-сценариев)
        ├── LogicForm.dll (WinForms/WebSocket UI, FormManagerSocket)
        ├── AdminForm.dll (административная форма)
        ├── Fleck.dll (WebSocket-сервер для kiosk UI, ws://0.0.0.0:8585)
        ├── BouncyCastle.Crypto.dll, Newtonsoft.Json.dll, Google.Protobuf.dll (сторонние)
        ├── CardClient.dll, CardReader.dll, SmartCardReader.dll, PinPad.dll и др. (опциональные периферийные драйверы)
        └── ExitSystem.dll (shutdown/restart)
```

## 3. Расположение файлов

| Назначение | Путь |
|-----------|------|
| Рабочая директория (проверенные DLL) | `/root/Logic/MonoDebugTemp/` |
| Исходники портирования (C#) | `/root/Logic/_audit/app/logiclinux-master/` |
| Оригинальные DLL из Windows-сборки | `/root/Logic/_audit/app/logiclinux-master/Source/` |
| Mock-серверы (TEST ONLY) | `/root/Logic/mock_servers/` |
| SQL-схема DB | `/root/Logic/db/script.sql` |
| Дамп DB | `/root/Logic/db/cash-in.bak` |
| Тестовые скрипты | `/root/Logic/start_logic_linux_test.sh` |
| Protocol specs (audit) | `/root/Logic/_audit/network/` |
| Deployment audit | `/root/Logic/_audit/deploy/` |
| Backups | `/root/Logic/backup_20260902_*` |
| Данная Wiki | `/root/Logic/LLM_WIKI/` |

## 4. Проверенные Linux binary artifacts

| DLL | Версия | Размер | SHA256 | Статус |
|-----|--------|--------|--------|--------|
| Logic.exe | - | - | - | проверен |
| **MainLogic.dll** | **7.0.0.0** | **319488** | `6fd34e555367c00e20f87a0ce2c046e1215db07b02ab70c663fff96db6c724c9` | **Linux-port rebuild, НЕ заменять** |
| CommonLib.dll | - | 124928 | - | пересобран при портировании |
| DataBase.dll | - | 168448 | - | пересобран (патч SQL→MySQL) |
| Devices.dll | - | 266240 | - | пересобран |
| LoggerLib.dll | - | 16896 | - | пересобран |
| DeviceCenter.dll | - | 41984 | - | оригинал |
| CashCodeWrapper.dll | - | 57856 | - | оригинал |
| Printer.dll | - | 66560 | - | оригинал |
| Protocol.dll | - | 181248 | - | оригинал (Windows) |
| SinkLib.dll | - | 27136 | - | оригинал (Windows) |

**ВАЖНО:** `Source/MainLogic.dll` (v4.2.5.4, 307712 B, md5 `87690bf9dbd0cd420f759ba723f8830a`) — это **старая pre-port Windows-сборка**, НЕ ИСПОЛЬЗУЕТСЯ. Текущий working `MainLogic.dll` (v7.0.0.0) соответствует исходникам и содержит Linux-port маркеры.

## 5. Точный статус тестов

| Тест | Результат |
|------|-----------|
| DB integration | **10/10 PASS** |
| Device layer | **13/13 PASS** |
| Cash-in E2E (DB) | **16/16 PASS** |
| Printer E2E (stub) | **13/13 PASS** |
| Logic startup | **PASS** ("Logis is STARTED") |
| Monitoring mock (TCP :333) | **PASS** (ACKs получены) |
| Payment mock (HTTP :14111) | **PASS** (реальный signed request от Logic) |
| E2E (start_logic_linux_test.sh) | **13/13 PASS** |
| RSA (test keys) | **PASS** (XmlSyntaxException устранена) |
| **Info flow (полный, 2026-09-03)** | **PASS** — GetInfo task Success: info.xml → parseInfoXml → settings/operators/services/limitation DB update (см. INFO_PAYMENT_FAILURE_AUDIT.md §16) |

## 6. Текущие production blockers

1. **Нет production RSA ключей** — в БД `cashin.keys` сейчас TEST-ключи (сгенерированы локально, 2048 бит). Перед подключением к production payment server необходима замена на ключи, выданные провайдером.
2. **Нет production адресов мониторинга/платежей** — в `cashin.settings` сейчас `MonitorURL=<MONITORING_SERVER>`, `PaymentUri=http://<MONITORING_SERVER>:14111`. Необходимо заменить на инфраструктурные адреса.
3. **Hardcoded DB connection string** — пароль БД вшит в `SqlSettingsProvider.dll` (`ConnectionStringHelper.cs`). Для production требуется либо совпадение MySQL user password, либо пересборка DLL.
4. **Отсутствует физическое оборудование** — CashCode/CCNet и Citizen PPU 700 не тестировались на реальных serial-портах.
5. **C# 8 → rebuild невозможен** — текущий Mono mcs 6.8 не поддерживает C# 8 (switch expressions, using declarations). Для пересборки MainLogic.dll требуется Roslyn / .NET SDK.
6. **MySql.Data 8.0.26/Mono: параметры SP по позиции** — MySql.Data на Mono передаёт параметры хранимых процедур по позиции, а не по имени; порядок параметров всех SP должен совпадать с порядком вызова в DAL (см. `add_limitation_fix.sql`).
7. **MySql.Data 8.0.26/Mono: ODKU → KeyNotFoundException** — см. INFO_PAYMENT_FAILURE_AUDIT.md §15/§16; фикс — IF EXISTS/UPDATE/INSERT + `active=0`.

## 7. Security findings (кратко)

- Пароль БД `<DB_PASSWORD>` hardcoded в `ConnectionStringHelper.cs` (скомпилирован в DLL)
- TEST RSA private key в `mock_servers/priv.hex` — никогда не поставлять на production
- `DAL.cs` выводит debug-строку `SELECT cashin.savePayment(...)` с account/total в stdout (journald) — medium leak
- `LogRequest=true` (если включён) пишет полный XML платежа включая account — держать false
- MySQL user `terminal` имеет `ALL ON cashin.*` + `cashin_test.*` — в production отозвать `cashin_test`
- `BanknoteStacked` test hook (MainLogic.cs:2984) — потенциальный backdoor ввода купюр через сценарий
- `SerialPort` debug-выводы в `Class402.cs` (printer COM traffic) — шум, в production отключить

## 8. Hardware requirements

### CashCode / CCNet bill acceptor
- RS-232 serial (ttyS0) или USB→Serial (ttyUSB0)
- 9600 baud, 8 data bits, None parity, 1 stop bit, encoding 866
- DB device: `Cup` → `CCNet` (description), `ComPort=/dev/ttyS0`
- Driver: `Class394` in `CashCodeWrapper.dll`
- Permissions: user `logic` in group `dialout`, udev `MODE="0660" GROUP="dialout"`

### Citizen PPU 700 printer
- RS-232 serial (ttyS1) или USB→Serial (ttyUSB1)
- 19200 baud, 8 data bits, None parity, 1 stop bit, encoding 866, paper width 80
- DB device: `printer` → `Citizen PPU 700` (description), `ComPort=/dev/ttyS1`
- Driver: `Class402` in `Devices.dll`
- Protocol: query loop `0x10 0x04 0x01..0x06` + ESC/POS-like print

## 9. Правила для будущего AI-инженера

См. раздел "RULES FOR FUTURE AI ENGINEERS" ниже.

## 10. Список файлов Wiki

| Файл | Описание |
|------|----------|
| `00_PROJECT_OVERVIEW.md` | Общее описание проекта, цели, статус |
| `01_ARCHITECTURE.md` | Архитектура: компоненты, зависимости, потоки данных |
| `02_SOURCE_TREE.md` | Структура исходников, проекты, сборка |
| `03_BINARY_ARTIFACTS.md` | Все DLL: размеры, хеши, версии, статус |
| `04_ORIGINAL_WINDOWS_VERSION.md` | Оригинальная Windows-сборка (Terminal.zip) |
| `05_LINUX_PORT.md` | Процесс портирования, изменения, tested artifacts |
| `06_DATABASE_MYSQL.md` | MySQL: схема, таблицы, процедуры, connection string |
| `07_DEVICE_ARCHITECTURE.md` | Device factory, architecture device stubs |
| `08_CASHCODE_CCNET.md` | CashCode CCNet: интерфейс, настройки, драйвер |
| `09_PRINTER_CITIZEN_PPU700.md` | Citizen PPU 700: интерфейс, настройки, драйвер |
| `10_NETWORK_MONITOR_PROTOCOL.md` | Monitoring protocol TCP :333 |
| `11_PAYMENT_PROTOCOL.md` | Payment protocol HTTP :14111 |
| `12_RSA_AND_KEYS.md` | RSA keys: формат, где хранятся, test vs production |
| `13_TESTING_AND_E2E.md` | Тестовая инфраструктура, E2E сценарии |
| `14_CONFIGURATION.md` | Настройки: DB settings, defaults, hardcoded values |
| `15_PRODUCTION_DEPLOYMENT.md` | Production deployment: install, systemd, README |
| `16_SECURITY_FINDINGS.md` | Security audit findings |
| `17_KNOWN_LIMITATIONS.md` | Известные ограничения и blockers |
| `18_TROUBLESHOOTING.md` | Типовые проблемы и решения |
| `19_BUILD_AND_REBUILD.md` | Сборка проектов, toolchain limitations |
| `20_CHANGELOG.md` | Changelog: Linux port baseline |
| `LLM_CONTEXT.md` | **Главный входной файл (данный документ)** |

---

## RULES FOR FUTURE AI ENGINEERS

1. **НЕ заменять проверенный MainLogic.dll.**  
   Текущий `MonoDebugTemp/MainLogic.dll` (v7.0.0.0, SHA256 `6fd34e555367c00e20f87a0ce2c046e1215db07b02ab70c663fff96db6c724c9`) — проверенный Linux-port rebuild. Любая замена должна быть обоснована и протестирована через полный E2E.

2. **НЕ использовать старый `Source/MainLogic.dll`.**  
   `/root/Logic/_audit/app/logiclinux-master/Source/MainLogic.dll` (v4.2.5.4, 307712 B) — это pre-port Windows-сборка, НЕ содержит Linux-port маркеров. НЕ использовать.

3. **НЕ переделывать завершённую MySQL migration без конкретной ошибки.**  
   MySQL слой (DAL, savePayment, AddMoney, get_balance, настройки) проверен тестами 10/10. Любые изменения — только по конкретной задокументированной ошибке.

4. **НЕ создавать новую mock-device framework; использовать существующие EMPTY stubs.**  
   `CashCodeEmpty` (DescriptionMpl "EMPTY") и `PrinterEMPTY` (DescriptionMpl "Заглушка") уже встроены в DLL. Они не выбраны production DB-строками (Cup→CCNet, printer→Citizen), но доступны через фабрику при изменении description.

5. **НЕ считать mock network/payment успешным production acceptance.**  
   Mock-серверы (`monitor_mock.py`, `payment_mock.py`) — только для тестов. Production acceptance требует реальных серверов и ключей.

6. **TEST RSA keys никогда не использовать в production.**  
   `cashin.keys` содержит test-ключи. `mock_servers/priv.hex` — test private key. Замена на production keys обязательна.

7. **Перед изменением baseline — создать backup.**  
   `cp -a /root/Logic/MonoDebugTemp /root/Logic/backup_<timestamp>_<description>/`

8. **После существенных изменений запускать `start_logic_linux_test.sh`.**  
   Скрипт проверяет DB, device, network, cash-in, printer, startup, E2E. Все 13 checks должны быть PASS.

9. **Production hardware acceptance проводить отдельно от mock tests.**  
   Mock-тесты проверяют software-слой. Hardware acceptance (реальный CashCode, реальный Citizen PPU 700) — отдельный этап на реальном терминале.

10. **Явно различать типы изменений:**
    - **Source change** — изменение `.cs` файлов, требует пересборки и полного тестирования
    - **Binary change** — замена DLL, требует проверки целостности через hash
    - **DB change** — изменение схемы/данных MySQL, может затронуть DAL и тесты
    - **Configuration change** — изменение `settings` таблицы, device properties, keys
    - Каждый тип изменения имеет свою процедуру валидации.