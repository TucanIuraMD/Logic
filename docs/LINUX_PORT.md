# 05 — Linux Port

## Что было сделано

Портирование Windows-терминала на Linux/Mono (Ubuntu 24.04 LTS, Mono 6.8.0.105).

## Выполненные этапы

### 1. MySQL миграция (SQL Server → MySQL)
- Причина: приложение изначально писало SQL Server-запросы.
- Исправлено: `DataBase/IBP/DAL.cs` (пересобран `DataBase.dll`).
- Таблицы и процедуры MySQL: `cashin.*` (schema из `db/script.sql`).
- Проверено: **10/10 DB integration tests PASS**.
- Процедуры/функции исправлены: savePayment, AddMoney, get_balance, getSumMoney, settings, requisites, limitations, scheduler, main menu.

### 2. Device layer fix (GetDeviceProperties)
- Причина: дубликаты в `properties4devices` и отсутствующие `CashCodeName/PrinterName`.
- Исправление: добавлен UNIQUE INDEX `(idd_device, _key)`; данные исправлены.
- Настройки: `CashCodeName=Cup`, `PrinterName=printer`.
- Device Factory: `Cup → CCNet`, `printer → Citizen PPU 700`.
- Проверено: **13/13 Device layer tests PASS**.

### 3. RSA keys fix
- Причина: в `cashin.keys` лежали обрезанные hex-значения (`<RSAKeyValue><Modulus>vm2TMESKEY` и `<RSAKeyValue><Modulus>SECRETKEY`) → `RSA.FromXmlString` бросала `XmlSyntaxException` ("Ошибка ключей: System error.").
- Исправление: сгенерированы валидные 2048-битные RSA-ключи в .NET XML-формате, записаны в `cashin.keys` (Public=415 B, Secret=1679 B).
- Проверено: Payment probe + реальный запрос Logic к payment mock — PASS. **Ключи TEST ONLY.**

### 4. Mock servers (network)
- `monitor_mock.py` — TCP :333 (mock мониторинга).
- `payment_mock.py` — HTTP :14111 (mock платежей).
- Протоколы исследованы по IL (Protocol.dll, SinkLib.dll) — см. `_audit/network/*.md`.
- Проверено: Logic подключается, шлёт VersionLogic/CurrentDateTime, ACK получены; реальный платеж отправлен в mock, ответ Accepted.

### 5. E2E сценарий
- `start_logic_linux_test.sh` — полный E2E (DB, device, network, cash-in, printer, startup, cleanup).
- **13/13 PASS.**

## Тестирование приложения

- `Logic.exe` запускается под Mono, доходит до "Logis is STARTED", стабильно работает.
- CashCode/Printer не могут открыть COM-порт без реального оборудования — это ожидаемо (log содержит ошибки Open, но приложение работает).

## Проверенные Linux artifacts

- `MainLogic.dll` — v7.0.0.0 (Linux-port rebuild, содержит маркеры).
- `CommonLib.dll`, `DataBase.dll`, `Devices.dll`, `LoggerLib.dll` — пересобраны/изменены.
- Остальные DLL — оригинальные Windows-сборки, работают под Mono без изменений.

## Чем НЕ занимались

- Пересборка MainLogic.dll на Linux (невозможно с mcs 6.8 — C# 8).
- Полная замена драйверов устройств.
- Production-подключение к серверам мониторинга/платежей (нет ключей/адресов).
