# 08 — CashCode / CCNet

## Устройство

Купюроприёмник (bill acceptor) CashCode по протоколу **CCNet**.

## Как выбирается

- DB `settings.CashCodeName` = `Cup`
- DB `devices` строка: type=`CashCode`, description=`CCNet`, name=`Cup`
- `DAL.GetDeviceByName(devices, typeof(CashCode), "Cup")` → `CCNet`
- `BaseFactory.GetCashCode("CCNet")` → класс `Class394` в CashCodeWrapper.dll
  (`[DescriptionMpl("CCNet")]`, internal class)

## Физический интерфейс

| Параметр | Значение |
|----------|----------|
| Интерфейс | RS-232 serial (native ttyS0) или USB→serial (ttyUSB0) |
| ComPort (DB) | `/dev/ttyS0` |
| BaudRate (DB) | 9600 |
| DataBits | 8 |
| Parity | None |
| StopBits | One |
| Encoding | 866 (DOS Cyrillic) |
| Протокол | CCNet (полудуплекс, последовательные команды/ответы через SerialPort) |
| Permissions | user `logic` в группе `dialout`, udev `MODE="0660" GROUP="dialout"` |

## Классы драйвера (CashCodeWrapper)

- `IBP.Devices.BillAcceptor.Models.Class394` — реализация CCNet (extends `CashCode`)
  - `CashPoll()` → `Class219.vmethod_5()` — опрос устройства
  - `CashPollStop()`, `CashReset()`, `GetBillTable()` (vmethod_26)
  - `Open()` → `Class219.vmethod_25()` → `Class346.vmethod_0()` → `COMPortWrapper.Open()`
  - `CurrentCurr` → `vmethod_4((long)currency)` — установка валюты/маски
- `IBP.Devices.BillAcceptor.Infrastructure.Protocol.*` — Class218..Class350 (протокольные классы CCNet)
- `CashCodeEmpty` — EMPTY stub для тестов (`[DescriptionMpl("EMPTY")]`)
- `CashCodeException`, `CashCodeEmptyForm` (WinForms, для stub-ввода купюр)

## Настройки (класс `SettingsCashCode`)

Свойства из `properties4devices`: ComPort, BaudRate, DataBits, Parity, StopBits, Encoding, CassetteCapacity, + SettingsStrings.

## Интеграция с MainLogic

- `BillAcceptor.Instance` (singleton).
- События: `OnError` → `MainLogic.method_6` (block NoMoney/FatalError), `OnCurrency` → `method_7` (AddMoney), `OnEscrow`, `OnAccepting`, `OnLog`.
- `MainLogic.Init()` (line ~2125): `BillAcceptor.DeviceNames = [CashCodeName]`, `Open()`, `CurrentCurr = (Currency)CashCodeMask`, `CashReset()`, `CashPoll()`.
- Ошибка Open на терминале без устройства логируется; `lockType_1 |= FatalError`.

## Аутодетект

`Class394.Autodetect()` — перебирает `Device.GetComPortNames()`, пишет `billacc_avtodetect.log`, пытается `vmethod_0()` на каждом порту.

## Статус тестирования

- Без реального оборудования: Open() бросает `COMPortException` (port not found) — ожидаемо.
- EMPTY stub — работает (WinForms не показывается headless; в CashInE2E используется DB-слой, не реальный приёмник).
- **Production hardware acceptance — отдельный этап, требует реального CCNet-устройства.**
