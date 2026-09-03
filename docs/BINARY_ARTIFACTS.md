# 03 — Binary Artifacts

## Текущие рабочие artifacts (MonoDebugTemp/)

| DLL | Размер (байт) | Версия | SHA256 (для проверенных) | Статус |
|-----|--------------|--------|--------------------------|--------|
| Logic.exe | 10752 | - | - | рабочий entry point |
| **MainLogic.dll** | **319488** | **7.0.0.0** | `6fd34e555367c00e20f87a0ce2c046e1215db07b02ab70c663fff96db6c724c9` | **Linux-port rebuild, проверен** |
| CommonLib.dll | 124928 | - | - | пересобран при портировании |
| DataBase.dll | 168448 | - | - | пересобран (SQL→MySQL патч) |
| Devices.dll | 266240 | - | - | пересобран |
| LoggerLib.dll | 16896 | - | - | пересобран |
| DeviceCenter.dll | 41984 | - | - | оригинал (не пересобран) |
| CashCodeWrapper.dll | 57856 | - | - | оригинал |
| Printer.dll | 66560 | - | - | оригинал |
| Protocol.dll | 181248 | - | - | оригинал (Windows) |
| SinkLib.dll | 27136 | - | - | оригинал (Windows) |
| LogicForm.dll | 55808 | - | - | оригинал |
| SqlSettingsProvider.dll | 9216 | - | - | оригинал |
| SettingParser.dll | 265216 | - | - | оригинал |
| AdminForm.dll | 321536 | - | - | оригинал |
| MySql.Data.dll | 1758208 | - | - | MySQL connector |
| Newtonsoft.Json.dll | 701992 | - | - | JSON |
| BouncyCastle.Crypto.dll | 2531328 | - | - | crypto |
| Fleck.dll | 44032 | - | - | WebSocket server |
| ICSharpCode.SharpZipLib.dll | 188416 | - | - | zip |
| PowerCollections.dll | 200704 | - | - | collections |
| CardClient.dll | 316416 | - | - | optional |
| CardReader.dll | 47616 | - | - | optional |
| CoinValidator.dll | 43008 | - | - | optional |
| Dispenser.dll | 89088 | - | - | optional |
| Hopper.dll | 22528 | - | - | optional |
| PinPad.dll | 59904 | - | - | optional |
| SmartCardReader.dll | 14848 | - | - | optional |
| BarCodeReaderWrapper.dll | 6144 | - | - | optional |
| Barcods.dll | 44032 | - | - | optional |
| ExitSystem.dll | 20480 | - | - | shutdown/restart |
| ExternalLib.dll | 11264 | - | - | external helper |
| IUpdate.dll | 4096 | - | - | update |
| RasWrapper.dll | 31744 | - | - | modem |
| Askopm.dll | 6144 | - | - | unknown/original |
| CryptoLib.dll | 28672 | - | - | crypto (unknown usage) |
| FlashControl.dll / ParserFlash.dll / FlashCommandWrapper.dll | - | - | - | flash (не используется на Linux) |
| netstandard.dll | 98616 | - | - | compat |
| System.Buffers.dll / System.Memory.dll / System.Numerics.Vectors.dll / System.Runtime.CompilerServices.Unsafe.dll | - | - | - | BCL backports |
| K4os.*.dll / Zstandard.Net.dll / Ubiety.Dns.Core.dll / Google.Protobuf.dll | - | - | - | сторонние |

## Важные отличия от `Source/` (pre-port Windows DLL)

| DLL | Source (pre-port) | MonoDebugTemp (рабочий) |
|-----|-------------------|--------------------------|
| **MainLogic.dll** | v4.2.5.4, 307712 B, md5 `87690bf9dbd0cd420f759ba723f8830a` | **v7.0.0.0, 319488 B, sha256 `6fd34e55...`** |
| CommonLib.dll | 120832 | 124928 |
| DataBase.dll | 173056 | 168448 |
| Devices.dll | 257024 | 266240 |
| LoggerLib.dll | 15872 | 16896 |

## Проверка целостности

- **MainLogic.dll** (рабочий): `sha256sum /root/Logic/MonoDebugTemp/MainLogic.dll` → `6fd34e555367c00e20f87a0ce2c046e1215db07b02ab70c663fff96db6c724c9`; md5 `02014cc1d919b0b723f1723dc83c00d5`.
- Старый Source/MainLogic.dll: md5 `87690bf9dbd0cd420f759ba723f8830a`, v4.2.5.4 — **pre-port, НЕ используется**.
- Одинаковые хеши рабочего MainLogic.dll во всех backups (backup_20260902_100909, _141756_network, _182000_prod_audit) — стабильный artifact.

## Target Framework / Architecture (MainLogic.dll)

- Target framework: **.NET Framework 4.6.1** (`TargetFrameworkAttribute` в DLL, `net461` в csproj)
- Architecture: **PE32 / AnyCPU (IL-only)**, работает на Mono 6.8 x86_64
- Сборка: csc (Roslyn) на Windows, path `c:\Users\hilov\Documents\Git\LogicLinux\MainLogic\obj\Debug\net461\MainLogic.pdb`
- AssemblyVersion 7.0.0.0, AssemblyFileVersion 2.0.0.0

## Тестовые executables (TEST ONLY)

| Файл | Назначение |
|------|-----------|
| DbIntegrationTest.exe | DB тесты (10) |
| DeviceLayerTest.exe | Device тесты (13) |
| CashInE2ETest.exe | Cash-in E2E (16) |
| PrinterE2ETest.exe | Printer stub E2E (13) |

## Leftover / debug artifacts (не поставлять)

`*.pdb`, `*.mdb`, `*.new`, `_originals/`, `C:\dabaseDLL_*.txt`, `C:\Temp\SocketServer.log`, `c:\logic.txt`, `c:\parser\*.xml`, `check.xml`, `info.xml`, `request.xml`, `response.xml`, `res_check.txt`, `encrypt.txt`, `decrpt.txt`, `db.udl`, `*.bat`, `NetTimeSetup-314.exe`, `mono_crash*.json`, `StopCashUpdate/`, `.uploadCache.cache`, `printer.log`, `logs/`.
