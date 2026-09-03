# 07 — Device Architecture

## Фабрика устройств (DeviceCenter)

- `IBP.BaseFactory` — фабрика устройств.
  - `GetDevice(DeviceType, description)` → `CashCodeFactory` / `PrinterFactory` / etc.
  - `GetCashCode("CCNet")`, `GetPrinter("Citizen PPU 700")`.
- Каждая фабрика: `BaseDeviceFactory<T, TType>` — ищет классы с атрибутом `[DescriptionMpl("...")]` в указанной assembly.
  - `CashCodeFactory` → CurrentAssembly = `CashCodeEmpty` assembly (CashCodeWrapper.dll)
  - `PrinterFactory` → CurrentAssembly = `PrinterEMPTY` assembly (Devices.dll)
- External devices: сканирует папки `\Devices\CashCodes`, `\Devices\Printers` (Windows path; на Linux каталог с обратным слэшем в имени).

## Ключевые устройства

| Логическое имя (settings) | type (DB devices) | description (DB devices) | класс драйвера |
|---------------------------|-------------------|--------------------------|----------------|
| `Cup` (CashCodeName) | CashCode | `CCNet` | `Class394` (CashCodeWrapper) |
| `printer` (PrinterName) | Printer | `Citizen PPU 700` | `Class402` (Devices.dll) |

## Singletons (DeviceCenter)

- `PrinterDevice` (IBP.Devices.Printer.Printer singleton wrapper)
  - `PrinterDevice.Instance`, `PrinterDevice.DeviceNames` (string[]), `PrinterDevice.GetPrinter(name)`
- `BillAcceptor` (IBP.Devices.BillAcceptor.CashCode singleton wrapper)
  - `BillAcceptor.Instance`, `BillAcceptor.DeviceNames`, `EnableStackTestDevice`
  - события: `OnError`, `OnLog`, `OnEscrow`, `OnCurrency`, `OnAccepting`
- Аналогично: CardReaderDevice, PinPadDevice, SmartCardReaderDevice, DispenserDevice, CoinValidatorDevice, KeyboardDevice.

## Driver интерфейсы

- `Device` (базовый): `Open()`, `Close()`, `Autodetect()`, `GetState()`, события `OnLog`.
- `CashCode` (BillAcceptor): `CashPoll()`, `CashPollStop()`, `CashReset()`, `GetBillTable()`, `CurrentCurr`, `BanknoteCompited()`, `EnableBA()/DisableBA()`.
- `Printer`: `Open()`, `Print(PrinterDocument)`, `GetState()`, `Info` (PrinterInfo: IsFiscal, CanPrintImage, BarTypes, Output).

## COM-port слой

- `COMPortWrapper` (Device/IBP.Devices/COMPortWrapper.cs):
  - оборачивает `System.IO.Ports.SerialPort` (Mono)
  - defaults: ComPort="COM1", DataBits=8, BaudRate=19200, StopBits=One, Parity=None, Encoding=866
  - переопределяется настройками из `properties4devices` (ComPort=/dev/ttyS0 и т.п.)
- `Device.GetComPortNames()` — `SerialPort.GetPortNames()` (Linux: /dev/ttyS*, /dev/ttyUSB*).

## EMPTY stubs (встроенные, для тестов)

| Класс | DescriptionMpl | Поведение |
|-------|----------------|-----------|
| `CashCodeEmpty` (CashCodeWrapper, IBP.Devices.BillAcceptor.Models) | "EMPTY" | Open() создаёт WinForms CashCodeEmptyForm; CashPoll пустой; GetBillTable → {}; vmethod_2 «Banknote accepted» |
| `PrinterEMPTY` (Devices, IBP.Devices.Printer.Models) | "Заглушка" | Open() логирует; GetState() → LentaStatus.Ok; Print() собирает строки в список; method_2 показывает MessageBox если есть содержимое (headless: использовать пустой документ) |

**Производственные DB-строки НЕ указывают на EMPTY** (Cup→CCNet, printer→Citizen PPU 700).  
Stubs доступны через фабрику если description = "EMPTY"/"Заглушка".

## Device test hooks в MainLogic

- `MainLogic.EnableStackTestBollAcceptior = false` (static; test-only, off by default)
- `BanknoteStacked` scenario context hook (MainLogic.cs ~2984, `//todo: удалить после тестов`) — имитация приёма купюры через сценарий.

## Настройки устройств (из DB properties4devices)

**Cup (id_device=109):** ComPort=/dev/ttyS0, BaudRate=9600, DataBits=8, Parity=None, StopBits=One, Encoding=866, CassetteCapacity=1000  
**printer (id_device=110):** ComPort=/dev/ttyS1, BaudRate=19200, DataBits=8, Parity=None, StopBits=One, Encoding=866, PaperWidth=80, PrinterCodeTable=7
