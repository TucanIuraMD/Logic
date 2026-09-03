# WIKI_VALIDATION

## Дата создания
2026-09-02

## Количество созданных документов

**22** Markdown-файла:

| # | Файл |
|---|------|
| 1 | `00_PROJECT_OVERVIEW.md` |
| 2 | `01_ARCHITECTURE.md` |
| 3 | `02_SOURCE_TREE.md` |
| 4 | `03_BINARY_ARTIFACTS.md` |
| 5 | `04_ORIGINAL_WINDOWS_VERSION.md` |
| 6 | `05_LINUX_PORT.md` |
| 7 | `06_DATABASE_MYSQL.md` |
| 8 | `07_DEVICE_ARCHITECTURE.md` |
| 9 | `08_CASHCODE_CCNET.md` |
| 10 | `09_PRINTER_CITIZEN_PPU700.md` |
| 11 | `10_NETWORK_MONITOR_PROTOCOL.md` |
| 12 | `11_PAYMENT_PROTOCOL.md` |
| 13 | `12_RSA_AND_KEYS.md` |
| 14 | `13_TESTING_AND_E2E.md` |
| 15 | `14_CONFIGURATION.md` |
| 16 | `15_PRODUCTION_DEPLOYMENT.md` |
| 17 | `16_SECURITY_FINDINGS.md` |
| 18 | `17_KNOWN_LIMITATIONS.md` |
| 19 | `18_TROUBLESHOOTING.md` |
| 20 | `19_BUILD_AND_REBUILD.md` |
| 21 | `20_CHANGELOG.md` |
| 22 | `LLM_CONTEXT.md` (главный входной файл) |

## Проверенные документы

Все 22 документа проверены:

1. **Существование файлов** — все 22 файла присутствуют в `/root/Logic/LLM_WIKI/`.
2. **Перекрёстные ссылки** — проверено, что все упоминаемые в документах файлы Wiki существуют (OK, 0 отсутствующих).
3. **Хеш MainLogic.dll** — SHA256 `6fd34e555367c00e20f87a0ce2c046e1215db07b02ab70c663fff96db6c724c9` упоминается в 4 документах (03, 15, 19, 20) + LLM_CONTEXT, все значения идентичны.
4. **Версия MainLogic.dll** — 7.0.0.0 указана согласованно (03, 20, LLM_CONTEXT, 01, 05). Старая версия 4.2.5.4 везде помечена как pre-port / НЕ используется.
5. **Старый MainLogic.dll не описан как current** — нигде `Source/MainLogic.dll` (v4.2.5.4) не представлен как текущий; во всех упоминаниях явно указано "pre-port, НЕ используется" (03, 04, 20, LLM_CONTEXT).
6. **TEST RSA keys помечены как TEST ONLY** — 21+ упоминаний "TEST ONLY"/"test keys" в документах; в 12_RSA_AND_KEYS.md, 16_SECURITY_FINDINGS.md, LLM_CONTEXT.md явно указано, что `cashin.keys` содержит TEST-ключи и не должны использоваться в production.
7. **Рекомендация не заменять MainLogic.dll** — зафиксирована в 19_BUILD_AND_REBUILD.md, 20_CHANGELOG.md, LLM_CONTEXT.md ("RULES FOR FUTURE AI ENGINEERS" правило №1).
8. **C# 8 rebuild limitation** — задокументирована в 17_KNOWN_LIMITATIONS.md и 19_BUILD_AND_REBUILD.md (mcs 6.8 max C# 7.2; требуется Roslyn/.NET SDK).

## Найденные противоречия

**Не найдено.** Все ключевые факты (версии, хеши, статусы тестов, порты, пути, названия классов/DLL/таблиц) согласованы между документами.

## UNKNOWN / требует проверки

Явно помечены в `17_KNOWN_LIMITATIONS.md`:
- Реальное поведение Citizen PPU 700 на RS-232 (только по коду, нет оборудования).
- Реальное поведение CCNet-приёмника (только по коду).
- Совместимость с реальным мониторинговым сервером (только mock).
- Совместимость с реальным payment server (только mock + формат).
- Назначение DLL `Askopm.dll`, `CryptoLib.dll`, `FlashControl.dll`, `ParserFlash.dll`, `RasWrapper.dll` в Linux-сборке.
- Необходимость `PaymentManager.dll` (не используется в текущей архитектуре).

Также отмечено в `LLM_CONTEXT.md` (раздел UNKNOWN-контекст): производственные адреса и ключи недоступны.

## Заключение о полноте Wiki

Wiki **полна** для целей onboarding: охватывает архитектуру, исходники, binary artifacts, БД, устройства (CashCode/CCNet, Citizen PPU 700), сетевые протоколы (monitoring :333, payment :14111), RSA/keys, тестирование/E2E, конфигурацию, production deployment, security, известные ограничения, troubleshooting, build/rebuild, changelog и правила для будущих AI-инженеров.

Все данные основаны на фактическом состоянии файлов/исходников/DLL/SQL/конфигураций и выполненных audit-отчётах; выдуманной информации не выявлено; рабочий baseline не изменён.

**Итог: Wiki готова к использованию как самостоятельная база знаний.**
