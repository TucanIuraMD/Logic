# 20 — Changelog

## Linux port baseline validated

Текущая точка состояния: **Linux port baseline validated**.

### Проверенный artifact — MainLogic.dll

| Параметр | Значение |
|----------|----------|
| Assembly version | **7.0.0.0** |
| Размер | **319488 bytes** |
| SHA256 | **`6fd34e555367c00e20f87a0ce2c046e1215db07b02ab70c663fff96db6c724c9`** |
| MD5 | `02014cc1d919b0b723f1723dc83c00d5` |
| Target framework | **.NET Framework 4.6.1** |
| Architecture | **PE32/AnyCPU (IL)** |
| Source tree | `/root/Logic/_audit/app/logiclinux-master/MainLogic/` |
| Current artifact | `/root/Logic/MonoDebugTemp/MainLogic.dll` |
| PDB | `/root/Logic/MonoDebugTemp/MainLogic.pdb` (build path `c:\Users\hilov\Documents\Git\LogicLinux\MainLogic\obj\Debug\net461\`) |

### Статус тестов на этой точке

```
DB             10/10 PASS
DEVICE         13/13 PASS
NETWORK        PASS
PAYMENT        PASS
MONITOR        PASS
RSA            PASS (test keys)
CASH-IN        16/16 PASS
PRINTER        13/13 PASS
STARTUP        PASS
E2E            13/13 PASS
```

### Хронология

| Дата | Событие |
|------|---------|
| 2022-03-30 | Windows-сборка LogicLinux (MainLogic v4.2.5.4, Source/) |
| ~2022 | Linux-port исходники (C# 8, маркеры Setting.Default.PrinterName=, file:///C:, Logis is STARTED) |
| 2026-09-01 | Импорт исходников в `/root/Logic/_audit/app/logiclinux-master/` |
| 2026-09-02 | MySQL миграция (DataBase.dll), device layer fix, RSA keys fix, mock servers, E2E |
| 2026-09-02 | **Linux port baseline validated** (MainLogic.dll v7.0.0.0, все тесты PASS) |
| 2026-09-02 | Production readiness audit (deploy/, network/ specs) |
| 2026-09-02 | LLM Knowledge Wiki создана |

### Важные факты

- Текущий `MainLogic.dll` (v7.0.0.0) — **Linux-port rebuild**, соответствует исходникам LogicLinux.
- Старый `Source/MainLogic.dll` (v4.2.5.4, 307712 B) — **pre-port build**, НЕ используется.
- Rebuild текущего MainLogic.dll на текущем Ubuntu/Mono (mcs 6.8) **невозможен** без полного Roslyn/.NET Framework build environment → проверенный DLL НЕ заменять.
- TEST RSA keys в `cashin.keys` — TEST ONLY.
