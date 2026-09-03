# 17 — Known Limitations

## Production blockers

1. **Нет production RSA ключей** — `cashin.keys` содержит TEST-ключи. Для production требуется замена на provider-issued keys.
2. **Нет production адресов** — `settings` указывают на localhost (MockURL=localhost:333, PaymentUri=http://localhost:14111). Для production нужны инфраструктурные адреса мониторинга и платежей.
3. **Hardcoded DB connection string** — пароль вшит в `SqlSettingsProvider.dll`; production требует либо совпадение с MySQL user password, либо пересборку DLL.
4. **Нет реального оборудования** — CashCode/CCNet и Citizen PPU 700 не тестировались на физических serial-портах.

## Toolchain ограничения

5. **MainLogic.dll rebuild невозможен на текущем Ubuntu/Mono** — исходники используют C# 8 (switch expressions в `Class49.cs`, using declarations в `BaseStep.cs`); Mono `mcs 6.8.0.105` поддерживает максимум C# 7.2. Требуется Roslyn / .NET SDK build environment. Проверенный DLL не заменять.
6. **Полная пересборка всего solution** требует Windows + Visual Studio (или .NET SDK + .NET Framework reference assemblies).

## Поведенческие особенности

7. **Лог-каталог с обратным слэшем** — `LoggerLib` создаёт каталог `<appDir>\logs` (имя содержит обратный слэш) из-за `"\\logs"` в `Logger.cs:296`. Работает, но нестандартно.
8. **`DAL.WriteLogToFile`** пишет в файл с именем `C:\dabaseDLL_<date>.txt` (Windows path) в рабочем каталоге — debug remnant.
9. **Device folders** `\Devices\CashCodes` и т.д. — Windows paths; на Linux каталоги с обратным слэшем (в текущей конфигурации внешние device DLL не используются).
10. **WebSocket UI port 8585** — `ws://0.0.0.0:8585` (Design=WebSocket) — слушает все интерфейсы; в production ограничить.
11. **WinForms MessageBox в PrinterEMPTY** — при печати с содержимым на headless Linux требует Xvfb; в PrinterE2E используется пустой документ.
12. **Class49 (LogonUser)** — Windows API (admin form); на Linux вызов вернёт ошибку, но в обычном потоке не используется.

## Тестовые ограничения

13. **Mock-тесты не являются production acceptance** — network/payment mocks воспроизводят формат, но не гарантируют совместимость с реальными серверами.
14. **RSA PASS — только с TEST keys** — подпись/верификация проверены, но реальный production key pair не проверен.
15. **CashCode/Printer hardware acceptance не проводился** — требуется реальное оборудование.

## DB/прочее

16. **Сумма наличных vs платежей** — при тестах может быть рассинхрон (проверяется `checkSumMoneyAndPayments`), это не ошибка, а особенность тестовых прогонов (сбрасывается).
17. **`cashin_test` БД** — тестовая копия; production grants не должны её включать.

## UNKNOWN / требует проверки

- Реальное поведение принтера Citizen PPU 700 на RS-232 (только по коду).
- Реальное поведение CCNet-приёмника (только по коду).
- Совместимость с реальным мониторинговым сервером (только mock).
- Совместимость с реальным payment server (только mock + формат).
- `Askopm.dll`, `CryptoLib.dll`, `FlashControl.dll`, `ParserFlash.dll`, `RasWrapper.dll` — назначение в Linux-сборке требует проверки.
- `PaymentManager.dll` — не используется в текущей архитектуре; требуется проверка необходимости.
