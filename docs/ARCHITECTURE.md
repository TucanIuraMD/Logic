# 01 — Architecture

## Компонентная схема

```
┌─────────────────────────────────────────────────────────────┐
│                      Logic.exe (entry)                       │
│  WinForms/WebSocket UI  ←→  LogicForm.dll (FormManagerSocket)│
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    MainLogic.dll (7.0.0.0)                   │
│  • Scenarii / сценарии (XML из SettingParser.dll)            │
│  • Payment flow: cash-in → SavePayment → send                │
│  • Emonitor: мониторинговый канал                            │
│  • Class59: поток отправки платежей                          │
│  • Device init: BillAcceptor, PrinterDevice                  │
└──┬───────────┬───────────┬───────────┬───────────────────────┘
   │           │           │           │
   ▼           ▼           ▼           ▼
CommonLib  DataBase.dll DeviceCenter  SinkLib.dll
(Payment,  (MySQL DAL)  (фабрика    (HTTP payment
 Card,     MySql.Data   устройств)   client: Formatter,
 Logger)       │          │          Frame, Class607)
   │           │          ▼           │
   │           │      Devices.dll     │
   │           │      (COMPort,       │
   │           │       CashCode,      │
   │           │       Printer)       │
   │           │      CashCodeWrapper │
   │           │      Printer.dll     │
   │           ▼                      ▼
   │      MySQL cashin            Payment server
   │      (settings, keys,        (HTTP POST :14111)
   │       devices, payments,
   │       money, messages)       Protocol.dll
   │                              (monitoring TCP :333)
   │
LoggerLib.dll (лог в appDir\logs + stdout)
SqlSettingsProvider.dll (настройки из MySQL settings table)
```

## Потоки и процессы (MainLogic)

| Класс | Назначение |
|-------|-----------|
| `IBP.MainLogic` | главный контроллер, Init(), сценарии, таймеры |
| `IBP.Emonitor` | мониторинговый канал (TCP :333), `ClientFormatter` из Protocol.dll |
| `IBP.Class59` | поток отправки платежей (Formatter из SinkLib) |
| `IBP.Class60` | обработчик файлов/обновлений |
| `IBP.TaskScheduller` (class52) | периодические задачи (GetInfo, SendMessage, CheckSum) |
| `IBP.FormManagerSocket` | WebSocket UI-мост (LogicForm) |
| `IBP.PutMoney` / step-классы | шаги сценариев (ввод суммы, приём купюр) |

## Поток данных при cash-in (упрощённо)

1. Сценарий `PutMoney` → `BillAcceptor` принимает купюру (`OnCurrency` / `OnEscrow`).
2. `MainLogic.method_7` (OnCurrency) → `DAL.Instance.AddMoney(nominal)` → `money` + `money_history` таблицы.
3. После завершения сценария → `SavePayment(ref payment)` → `SELECT cashin.savePayment(...)` → payments таблица + TicketNumber.
4. `Class59` поток подхватывает неотправленные платежи → `Formatter.Process(frame)` → HTTP POST на `PaymentUri`.
5. Ответ сервера парсится → `payment.State` (Accepted=2000 и т.д.) → `sent=1`.

## Поток мониторинга

1. `Emonitor` → `ClientFormatter.Start()` → TCP connect :333.
2. Отправка Initialization + CurrentDateTimeMessage + неотправленные сообщения из БД (`GetNotSentMessNew`).
3. Сервер отвечает ACK; периодически присылает Command (type 2).
4. Команды исполняются колбэками (SetConfig, GetConfig, ExecuteSql, SetBlock, и т.д.).

## Ключевые особенности порта

- **DB connection string hardcoded** в `SqlSettingsProvider/IBP/Configuration/ConnectionStringHelper.cs`:
  `server=localhost;userid=<DB_USER>;password=<DB_PASSWORD>;database=cashin;CharSet=utf8mb4;CheckParameters=false;`
- **Настройки** читаются из MySQL `cashin.settings` через `SqlSettingsProvider` (класс `IBP.Setting`).
- **UI**: `Design=WebSocket` → `FormManagerSocket` + `Fleck` WebSocket server на `ws://0.0.0.0:8585`.
- **Лог**: `LoggerLib` пишет в `appDir\logs\yyyy-MM-dd.log` (имя каталога содержит обратный слэш — Linux quirk) и в stdout.
- **Windows-зависимости**: `ExitSystem.dll` (shutdown), `Class49` (Win32 LogonUser — admin-форма, не используется в обычном потоке).

## Зависимости DLL (MainLogic.dll assembly refs)

AdminForm, Askopm, Barcods, CardClient, CardReader, CashCodeWrapper, CommonLib, DataBase, DeviceCenter, Devices, ExitSystem, ExternalLib, ICSharpCode.SharpZipLib, IUpdate, LoggerLib, LogicForm, PowerCollections, Protocol, RasWrapper, SettingParser, SinkLib, SmartCardReader, SqlSettingsProvider, System.*, mscorlib.
