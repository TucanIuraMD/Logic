# 19 — Build and Rebuild

## Текущий toolchain

| Компонент | Версия | Путь |
|-----------|--------|------|
| Mono | 6.8.0.105 | /usr/bin/mono |
| mcs | 6.8.0.105 | /usr/bin/mcs |
| xbuild | - | /usr/bin/xbuild |
| msbuild | НЕТ | - |
| dotnet SDK | НЕТ | - |
| csc (Roslyn) | НЕТ | - |

## Сборка MainLogic.dll

**НЕВОЗМОЖНО на текущем Ubuntu/Mono toolchain.**

Причины:
1. Исходники используют **C# 8**: switch expressions (`Class49.cs:37`), using declarations (`BaseStep.cs:314,822`), и др.
2. Mono `mcs 6.8` поддерживает max C# 7.2 (`-langversion:8` rejected; `-langversion:7.2` даёт 12 errors).
3. Проект — SDK-style (`Microsoft.NET.Sdk.WindowsDesktop`, net461, UseWindowsForms) — xbuild не может его собрать.
4. Нет msbuild / dotnet SDK / Roslyn csc.
5. Требуется полный Roslyn + .NET Framework 4.6.1 reference assemblies build environment (Windows + Visual Studio или Linux + .NET SDK + NuGet ref assemblies).

## Как был собран текущий MainLogic.dll

- На Windows (Visual Studio / Roslyn), path: `c:\Users\hilov\Documents\Git\LogicLinux\MainLogic\obj\Debug\net461\MainLogic.pdb`
- AssemblyVersion 7.0.0.0 (Properties/AssemblyInfo.cs), net461, PE32/AnyCPU.
- Текущий `MonoDebugTemp/MainLogic.dll` — Linux-port rebuild (содержит маркеры Setting.Default.PrinterName=, file:///C:, BanknoteStacked, Logis is STARTED), но собран тем же Windows-окружением (или совместимым).

## Частичная пересборка, выполненная на Linux

Для портирования на Linux были пересобраны (через совместимый toolchain / mcs):
- `CommonLib.dll` (124928 B) — изменён
- `DataBase.dll` (168448 B) — SQL→MySQL патч
- `Devices.dll` (266240 B) — изменён
- `LoggerLib.dll` (16896 B) — изменён

Эти DLL собраны из соответствующих исходников в `_audit/app/logiclinux-master/`.

## Как пересобрать (если потребуется)

### Вариант 1: Windows + Visual Studio (рекомендуемый)
1. Открыть `LogicLinux.sln` на Windows.
2. Restore NuGet packages.
3. Build solution (Debug/Release AnyCPU).
4. Скопировать новые DLL в рабочий каталог ТОЛЬКО после полного E2E теста.

### Вариант 2: Linux + .NET SDK
1. Установить .NET SDK (6/8).
2. Добавить `Microsoft.NETFramework.ReferenceAssemblies` NuGet package.
3. `dotnet build LogicLinux.sln` (WindowsDesktop targeting может потребовать Windows).
4. Прогнать все тесты.

## Правила

- **НЕ заменять проверенный MainLogic.dll** без полного E2E.
- Собирать в отдельную копию, не трогать рабочий каталог.
- После сборки: DB 10/10, Device 13/13, CashIn 16/16, Printer 13/13, Startup, E2E 13/13.
- Хеш текущего проверенного DLL: SHA256 `6fd34e555367c00e20f87a0ce2c046e1215db07b02ab70c663fff96db6c724c9`.

## Сборка тестовых exe на Linux (выполняется)

```bash
cd /root/Logic/MonoDebugTemp
mcs -out:DbIntegrationTest.exe -r:System.Data.dll -r:CommonLib.dll -r:DataBase.dll -r:MySql.Data.dll -r:LoggerLib.dll ...
mcs -out:DeviceLayerTest.exe -r:System.Data.dll -r:CommonLib.dll -r:DataBase.dll -r:MySql.Data.dll -r:DeviceCenter.dll -r:CashCodeWrapper.dll ...
mcs -out:CashInE2ETest.exe -r:System.Data.dll -r:CommonLib.dll -r:DataBase.dll -r:MySql.Data.dll -r:LoggerLib.dll ...
mcs -out:PrinterE2ETest.exe -r:System.Data.dll -r:CommonLib.dll -r:Devices.dll -r:DeviceCenter.dll -r:LoggerLib.dll -r:Printer.dll ...
```
(тестовые исходники: `_audit/DbIntegrationTest.cs`, `_audit/DeviceLayerTest.cs`, `/tmp/CashInE2ETest.cs`, `/tmp/PrinterE2ETest.cs`)
