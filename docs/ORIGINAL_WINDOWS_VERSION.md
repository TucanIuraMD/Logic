# 04 — Original Windows Version

## Источник

- ZIP-архив `/root/Logic/Terminal.zip` — исходная Windows-сборка платёжного терминала.
- `/root/Logic/_audit/app/logiclinux-master/Source/` — оригинальные pre-port DLL из этой сборки.
- Репозиторий: `LogicLinux` (GitHub/branch master), путь `c:\Users\hilov\Documents\Git\LogicLinux\`.

## Оригинальные DLL (Source/)

| DLL | Размер | Версия |
|-----|--------|--------|
| MainLogic.dll | 307712 | 4.2.5.4 |
| CommonLib.dll | 120832 | - |
| DataBase.dll | 173056 | - |
| Devices.dll | 257024 | - |
| LoggerLib.dll | 15872 | - |
| SinkLib.dll | 27136 | - |
| Protocol.dll | 181248 | - |
| CashCodeWrapper.dll | 262144 | - |
| Printer.dll | 262144 | - |
| PaymentManager.dll | 262144 | - |
| + прочие DLL (всего ~50 файлов) | - | - |

**Важно:** `Source/MainLogic.dll` (v4.2.5.4) — **pre-port Windows-сборка**.  
Она НЕ содержит Linux-port маркеры (`Setting.Default.PrinterName=`, `file:///C:`, `BanknoteStacked`).  
Текущий рабочий `MainLogic.dll` (v7.0.0.0) — пересборка из портированных исходников.  
**Source/MainLogic.dll НЕ ИСПОЛЬЗОВАТЬ.**

## Отличия Source/ от рабочих MonoDebugTemp/

| Характеристика | Source/ (оригинал) | MonoDebugTemp/ (рабочий) |
|---------------|-------------------|--------------------------|
| MainLogic.dll | v4.2.5.4, 307712 B | v7.0.0.0, 319488 B |
| CommonLib.dll | 120832 B | 124928 B |
| DataBase.dll | 173056 B | 168448 B (SQL→MySQL) |
| Devices.dll | 257024 B | 266240 B |
| LoggerLib.dll | 15872 B | 16896 B |
| Has Linux markers? | НЕТ | ДА |

## Дополнительные оригинальные файлы

- `Source/AdminForm.dll` — административная форма (Windows-зависимости)
- `Source/PaymentManager.dll` — не используется в текущей архитектуре (платёж через SinkLib)
- `Source/ExternalLib.dll` — внешние библиотеки
- `Source/CardClient.dll`, `Source/CardReader.dll` — кардридеры (orig)
- `Source/Protocol.dll`, `Source/SinkLib.dll` — сетевые протоколы (одинаковые с рабочими)