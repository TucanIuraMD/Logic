# 09 — Printer Citizen PPU 700

## Устройство

Термопринтер **Citizen PPU 700** (не фискальный в текущей конфигурации).

## Как выбирается

- DB `settings.PrinterName` = `printer`
- DB `devices` строка: type=`Printer`, description=`Citizen PPU 700`, name=`printer`
- `DAL.GetDeviceByName(devices, typeof(Printer), "printer")` → `Citizen PPU 700`
- `BaseFactory.GetPrinter("Citizen PPU 700")` → класс `Class402` в Devices.dll
  (`[DescriptionMpl("Citizen PPU 700")]`, internal class, extends `Class401`)

## Физический интерфейс

| Параметр | Значение |
|----------|----------|
| Интерфейс | RS-232 serial (native ttyS1) или USB→serial (ttyUSB1) |
| ComPort (DB) | `/dev/ttyS1` |
| BaudRate (DB) | 19200 |
| DataBits | 8 |
| Parity | None |
| StopBits | One |
| Encoding | 866 |
| PaperWidth (DB) | 80 |
| PrinterCodeTable (DB) | 7 |
| Permissions | user `logic` в группе `dialout`, udev `MODE="0660" GROUP="dialout"` |

## Классы драйвера

- `Class402` (Citizen PPU 700) — `GetState()` опрашивает регистры командой `0x10 0x04 0x01..0x06` (ReadFromComPort 1 байт), декодирует флаги:
  - 0x01: биты — ErrorWaitRecover(1_5), ErrorFeed(1_6), ... (покрытие открыто и т.д.)
  - 0x02, 0x03, ...: состояние ленты (LentaStatus), нет бумаги и т.д.
- `Class401` — базовый класс Citizen (CBM1000 / CT-S 2000 / PPU 700), работа с `COMPortWrapper`, ESC/POS-подобная печать.
- `PrinterEMPTY` — EMPTY stub (`[DescriptionMpl("Заглушка")]`) для тестов.
- `PrinterVKP` — другой принтер (не используется).
- `PrinterComPort_3`, `PrinterFiscal_3` — фискальные варианты (не используются).

## Интерфейс печати (IBP.Devices.Printer)

- `Printer.Print(PrinterDocument document)`
- `PrinterDocument(PresenterMode)` — DocumentType (NotFiscal/Fiscal), Body (List<Page>), AddPrintObjects(List<PrintObject>)
- `Page` — objects (List<PrintObject>), FiscalOperation, Summa
- `PrintObject` — text/barcode/image, Align
- `PrinterState` — LentaStatus (Ok/Short/ShortAndNull/Null/Unknown), Error, HasError, State (FiscalState...)
- `PrinterInfo` — IsFiscal, CanPrintImage, Output (PresenterMode list), BarTypes
- `PrintObjectType`: Text, Barcode, Image

## Интеграция с MainLogic

- `PrinterDevice.Instance` (singleton).
- `MainLogic.Init()`: `PrinterDevice.DeviceNames = [PrinterName]`, `Open()`, `GetState()`; если `state.HasError` и принтер фискальный → `lockType_1 |= PrinterError`.
- Печать: `method_75()` → формирует список PrintObject → `printer.Print(new PrinterDocument(...))` или FiscalDocument.
- `method_77()` — проверка состояния после печати (`LentaStatus`), отправка `ChequeLentaMessage` в мониторинг, блокировка при NoPaper/PrinterError.
- `PrinterDevice.GetPrinter(name)` — выбор принтера по имени (для нескольких принтеров).

## Статус тестирования

- PrinterE2ETest (13/13 PASS) — через `PrinterEMPTY` ("Заглушка"): Open, GetState (LentaStatus=Ok, HasError=false), Print с пустым документом (без MessageBox).
- `PrinterEMPTY.Print()` с текстом показывает MessageBox → на headless Linux требует Xvfb или пустой документ.
- **Production hardware acceptance — отдельный этап, требует реального Citizen PPU 700.**
