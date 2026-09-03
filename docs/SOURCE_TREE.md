# 02 — Source Tree

## Расположение

| Путь | Содержимое |
|------|-----------|
| `/root/Logic/_audit/app/logiclinux-master/` | исходники портирования (C# projects) |
| `/root/Logic/_audit/app/logiclinux-master/Source/` | оригинальные pre-port DLL из Windows-сборки |
| `/root/Logic/MonoDebugTemp/` | рабочие binary artifacts (сборочный каталог) |
| `/root/Logic/_audit/` | тестовые исходники, SQL, audit-отчёты |
| `/root/Logic/db/` | SQL-схема (`script.sql`), дамп (`cash-in.bak`) |

## Проекты (solution LogicLinux.sln)

| Проект | каталог | назначение |
|--------|---------|-----------|
| CommonLib | `CommonLib/` | модели IBP.*: Payment, Card, ServiceTerminal, ExternalProcess, Logger |
| DataBase | `DataBase/` | MySQL DAL (`IBP.DAL`), PSCB.CashIn |
| DeviceCenter | `DeviceCenter/` | фабрика устройств, `BaseFactory`, `PrinterDevice`, `BillAcceptor` |
| Device | `Device/` | драйверы: `IBP.Devices.Printer`, `IBP.Devices.COMPort`, card readers, pinpads |
| Devices | `Devices/` | альтернативная папка драйверов (частично дублирует Device/) |
| CashCodeWrapper | `CashCodeWrapper/` | CCNet протокол (Class394 и др.) |
| Printer | `Printer/` | модели принтеров (Citizen PPU 700, PrinterEMPTY) |
| LogicForm | `LogicForm/` | UI: FormManagerSocket, WebSocket |
| MainLogic | `MainLogic/` | бизнес-логика, сценарии |
| Logic | `Logic/` | entry point (Logic.exe), logictest/ |
| LoggerLib | `LoggerLib/` | логирование |
| SqlSettingsProvider | `SqlSettingsProvider/` | чтение настроек из MySQL |
| SocketServer | `SocketServer/` | тестовый WebSocket сервер (ws://0.0.0.0:8181) |
| MySqlDebug | `MySqlDebug/` | диагностический проект для отладки MySQL |

## MainLogic project структура

```
MainLogic/
├── MainLogic.csproj            (SDK-style, net461, UseWindowsForms, LangVersion 8/Preview)
├── Class63.cs
├── Properties/AssemblyInfo.cs  (AssemblyVersion 7.0.0.0, AssemblyFileVersion 2.0.0.0)
├── Properties_M/Settings.cs
├── MMPS/FormManagerSocket.cs   (WebSocket UI)
└── IBP/
    ├── MainLogic.cs            (главный контроллер, ~4300 строк)
    ├── Setting.cs              (настройки, ~1436 строк, SqlSettingsProvider)
    ├── Emonitor.cs             (мониторинговый канал)
    ├── Class59.cs              (поток отправки платежей)
    ├── Class60.cs, Class62.cs  (файлы/обновления)
    ├── Scenarii.cs             (сценарии)
    ├── BaseStep.cs, PutMoney.cs, steps/ (шаги сценариев)
    ├── Class3..Class9, Class36, Class45..Class60  (шаги/помощники)
    ├── TaskScheduller (в Class52?)  — периодические задачи
    └── ...
```

## Особенности исходников

- **C# 8** используется (switch expressions в `Class49.cs`, using declarations в `BaseStep.cs`).
- Windows-специфичный код присутствует: `Class49` (LogonUser/WindowsIdentity — admin form), `OleDbConnectionStringBuilder` (BaseStep), `OpenFileDialog` (LoggerForm), Registry (только `Logic/logictest/Class0.cs` — тестовая утилита).
- `MainLogic.cs` содержит Linux-port маркеры:
  - `Console.WriteLine($"Setting.Default.PrinterName=...")` (line ~2156)
  - `file:///c|` → `file:///C:` replace в `onJustEvent` (Linux path fix)
  - `BanknoteStacked` test hook (line ~2984, `//todo: удалить после тестов`)
  - `"=========== Logis is STARTED ==============="` (line ~2612)
- `MainLogic.cs` использует `//FormManager...` (закомментированный WinForms код) — UI вынесен.

## Тестовые исходники

| Файл | Назначение |
|------|-----------|
| `/root/Logic/_audit/DbIntegrationTest.cs` | DB тесты (10 checks) |
| `/root/Logic/_audit/DeviceLayerTest.cs` | Device layer тесты (13 checks) |
| `/tmp/CashInE2ETest.cs` → `MonoDebugTemp/CashInE2ETest.exe` | Cash-in E2E (16 checks) |
| `/tmp/PrinterE2ETest.cs` → `MonoDebugTemp/PrinterE2ETest.exe` | Printer stub E2E (13 checks) |
| `/root/Logic/mock_servers/monitor_mock.py` | mock monitoring TCP :333 |
| `/root/Logic/mock_servers/payment_mock.py` | mock payment HTTP :14111 |

## Сборка

- Сборка производится из **Windows** (Visual Studio / Roslyn) — исходники содержат C# 8.
- На Linux: `mcs` (Mono 6.8) НЕ поддерживает C# 8 (max 7.2) → rebuild MainLogic.dll невозможен без Roslyn/.NET SDK.
- Частичная пересборка на Linux выполнялась для `CommonLib.dll`, `DataBase.dll`, `Devices.dll`, `LoggerLib.dll` (см. `MonoDebugTemp/*.new` и `_originals/`).
