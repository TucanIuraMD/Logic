# 14 — Configuration

## Источники конфигурации

| Источник | Приоритет | Что содержит |
|----------|-----------|-------------|
| MySQL `cashin.settings` | highest | настройки терминала: MonitorURL, MonitorPort, PaymentUri, PointId, ClientId, CashCodeName, PrinterName и др. |
| `Setting.cs` defaults | default | значения по умолчанию, если в БД нет записи |
| `.config` файлы | startup | `Logic.exe.config`, `DataBase.dll.config`, `MainLogic.dll.config` (assembly bindings, version) |
| Hardcoded | runtime | `ConnectionStringHelper.cs` (DB connection string), `WebSocketControl.cs` (8585 ws://0.0.0.0), `COMPortWrapper.cs` (COM1, 19200, 866), `DeviceFolders.cs` (Windows paths) |

## Настройки MySQL (settings table)

| Name | Current value (test) | Production needed |
|------|---------------------|-------------------|
| MonitorURL | localhost | infra-assigned host |
| MonitorPort | 333 | 333 |
| PaymentUri | http://localhost:14111 | http://<payment-host>:14111 |
| PointId | 0 | assigned per terminal |
| ClientId | 011f336a-a6bb-11f1-9ecd-bc24110e87dd | unique per terminal |
| SerialPoint | 1 | assigned |
| ClientName | LogicTest | terminal name |
| CashCodeName | Cup | Cup (kept) |
| PrinterName | printer | printer (kept) |
| Design | WebSocket | WebSocket (kiosk UI) |
| ScreenSize | 1024x768 | kiosk resolution |
| StartStep | MainMenu | MainMenu |

Полный список: ~30 настроек (см. `Setting.cs`). Settings not in DB fall back to defaults.

## Hardcoded значения (требуют пересборки для изменения)

| Значение | Где | Влияние |
|----------|-----|---------|
| `server=localhost;userid=<DB_USER>;password=<DB_PASSWORD>;database=cashin;CharSet=utf8mb4;CheckParameters=false;` | `ConnectionStringHelper.cs` (SqlSettingsProvider) | DB connection (пароль вшит) |
| `<MONITORING_SERVER>` | Setting.cs — MonitorURL default | default мониторинга |
| `333` | Setting.cs — MonitorPort default | порт мониторинга |
| `http://<MONITORING_SERVER>:14111` | Setting.cs — PaymentUri default | default платежей |
| `<PROXY_SERVER>` | Setting.cs — ProxyName default | default proxy |
| `14444` | Setting.cs — ProxyPort default | порт proxy |
| `ws://0.0.0.0:8585` | WebSocketControl.cs | WebSocket UI port |
| `"\logs"` (backslash) | Logger.cs | log directory name |
| `"\Devices\Printers"` etc. | DeviceFolders.cs | external device folders |
| `COM1` / 19200 / 866 | COMPortWrapper.cs | serial defaults |
| `C:\dabaseDLL_<date>.txt` | DAL.cs:52 | debug log path |

## Device configuration (MySQL)

**Cup (CashCode → CCNet):** ComPort=/dev/ttyS0, BaudRate=9600, DataBits=8, Parity=None, StopBits=One, Encoding=866, CassetteCapacity=1000  
**printer (Printer → Citizen PPU 700):** ComPort=/dev/ttyS1, BaudRate=19200, DataBits=8, Parity=None, StopBits=One, Encoding=866, PaperWidth=80, PrinterCodeTable=7

## Production configuration model

Единственный источник: MySQL `cashin.settings` + `devices`/`properties4devices` + `keys`.  
Per-terminal provisioning: установить MonitorURL, PaymentUri, PointId, ClientId, заменить keys.

## .config files (assembly bindings)

- `Logic.exe.config`, `DataBase.dll.config`: `System.Buffers` binding redirect; `.NETFramework,Version=v4.6.1`
- `MainLogic.dll.config`: version 4.2.5.3 (не влияет, т.к. MainLogic v7.0.0.0)