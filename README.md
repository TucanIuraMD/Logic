# Logic Linux

## 1. Назначение проекта

**Logic Linux** — Linux-порт платежного терминального приложения Logic, первоначально разработанного для Windows.

Проект предназначен для переноса существующей бизнес-логики, работы с базой данных, сетевого взаимодействия, устройств терминала и HTML-интерфейса на Linux/Ubuntu с максимальным сохранением исходного поведения.

Основной принцип проекта:

> Не переписывать работающую систему без необходимости. Сначала сохранить и воспроизвести существующее поведение, затем заменять только те платформенные компоненты, которые действительно требуют замены.

---

## 2. Текущее состояние

| Этап | Состояние |
|---|---|
| Phase 3 — PoC | **FROZEN / PASS** |
| Phase 4A — Ubuntu test environment | **PASS** |
| Phase 4B — Linux deployment / E3000N | **PASS** |
| Phase 4C — autostart / recovery / load | **PASS** |
| Phase 4D — physical hardware validation | **PENDING** |
| Phase 5 — final robustness / cleanup | **PENDING** |

Основные результаты на текущий момент:

- Linux/Mono runtime работает на реальном Ubuntu E3000N.
- `Logic.exe` запускается на Linux.
- `MainLogic.dll` работает под Mono.
- HTML UI запускается через Chromium.
- WebSocket-транспорт работает.
- MySQL-версия базы данных работает.
- Основные сценарии E2E проверены.
- Production database не используется приложением на тестовом E3000N.
- Autostart и recovery после перезапуска проверены.
- Load test: **15/15 full-sequence клиентов**, 195 сообщений.
- Физические CashCode и Citizen устройства ещё требуют отдельной валидации.

**Полный rewrite UI не требуется.**

---

## 3. Основные компоненты

Проект состоит из следующих основных частей:

- `Logic.exe` — основное приложение / launcher.
- `MainLogic.dll` — основная бизнес-логика.
- `LogicForm.dll` — UI/form layer.
- `DataBase.dll` — работа с базой данных.
- `CommonLib.dll` — общие компоненты.
- `Devices.dll` — работа с устройствами терминала.
- `SqlSettingsProvider.dll` — настройки через БД.
- HTML UI — пользовательский интерфейс терминала.
- WebSocket adapter — Linux-транспорт между HTML UI и backend.
- MySQL — Linux-совместимая база данных.
- Monitor protocol — взаимодействие с monitor-сервисом.
- Payment protocol — взаимодействие с payment-сервисом.
- `info.xml` — информация, получаемая от payment/backend-сервера.
- systemd/autostart infrastructure — запуск Logic после загрузки Linux.

---

## 4. Архитектура

### Backend

Исходная логическая структура:

```text
Logic.exe
    │
    ▼
MainLogic.dll
    │
    ├── PaymentManager
    ├── DeviceCenter
    ├── DataBase
    └── network/payment services
```

### HTML UI

Основной UI:

```text
Chromium
    │
    ▼
main.html
    │
    ├── HTML screens
    ├── JavaScript
    ├── CSS
    └── PNG/assets
```

Транспорт UI → backend:

```text
HTML Screen
    │
    ▼
Kernel API
    │
    ▼
window.external.* compatibility layer
    │
    ▼
WebSocket adapter
    │
    ▼
ws://127.0.0.1:8585
    │
    ▼
MainLogic
```

Используемая WebSocket-инфраструктура:

```text
MainLogic
    │
    ▼
FormManagerSocket
    │
    ▼
WebSocketControl
    │
    ▼
Fleck
    │
    ▼
WebSocket :8585
```

### UI compatibility layer

Для запуска старого HTML UI на Chromium были реализованы compatibility shims для legacy IE/COM behavior, включая:

* `window.external`;
* `window.external` внутри iframe/frame;
* `createElement("<html>")`;
* `ActiveXObject("Msxml2.DOMDocument")`;
* синхронную загрузку динамических JavaScript-файлов через `inc()`.

Эти компоненты необходимы для сохранения существующей HTML/JavaScript логики без полного переписывания UI.

---

## 5. База данных

Исходная система использовала SQL Server.

Linux-порт использует **MySQL 8**.

Основные изменения SQL-совместимости включают:

* `@@IDENTITY` → `LAST_INSERT_ID()`;
* `SCOPE_IDENTITY()` → `LAST_INSERT_ID()`;
* `IDENT_CURRENT()` → `LAST_INSERT_ID()`;
* `ISNULL` → `IFNULL`;
* `PARSENAME` → `SUBSTRING_INDEX`;
* `newid()` → `UUID()`;
* `GETDATE()` → `NOW()`;
* `sysobjects` → `INFORMATION_SCHEMA`;
* `sp_helptext` → `SHOW CREATE PROCEDURE`;
* `@@ROWCOUNT` → `ROW_COUNT()`;
* SQL Server `UPDATE ... FROM` → MySQL-compatible `UPDATE ... JOIN`;
* удаление SQL Server `[dbo]`;
* адаптация `DECLARE @variable`;
* адаптация функций и stored procedures.

База тестового Linux-терминала работает локально:

```text
127.0.0.1:3306
database: cashin
user: terminal
```

Приложение не должно подключаться к production database во время тестирования Linux-порта.

Подробности находятся в:

```text
docs/DATABASE_STATUS.md
```

---

## 6. Устройства

Поддерживаемые терминальные устройства включают:

### CashCode

Ожидаемая конфигурация:

```text
Device: CashCode
Logical name: Cup
Driver: Class394
Protocol: CCNet
Serial: ttyS0 / COM1
Speed: 9600
Format: 8N1
```

### Citizen PPU 700

Ожидаемая конфигурация:

```text
Device: Citizen
Printer: printer
Driver: Class402
Serial: ttyS1 / COM2
Speed: 19200
Format: 8N1
Width: 80
```

UART-инфраструктура на E3000N проверена.

**Физическая валидация устройств остаётся отдельным этапом Phase 4D.**

Нельзя считать физическое устройство рабочим только на основании наличия `/dev/tty*` или успешного открытия serial-порта.

Подробности:

```text
docs/DEVICE_STATUS.md
```

---

## 7. Сетевые протоколы

### Monitor

Monitor использует binary TCP protocol:

```text
TCP :333
```

Основные особенности:

* 4-byte little-endian length prefix;
* packet flags/type/code/number/body;
* deflate для больших payload;
* Initialization;
* CurrentDateTime;
* ACK messages.

### Payment

Payment использует HTTP POST:

```text
HTTP :14111
```

Payload содержит:

```text
keyId
data length
UTF-16LE XML
RSA-MD5 signature
```

Тестовые mock servers существуют для проверки протоколов.

**Production payment/monitor endpoints не должны заменяться mock-данными без явного указания.**

Подробности:

```text
docs/NETWORK_PAYMENT_STATUS.md
```

---

## 8. `info.xml`

`info.xml` используется для получения информации от payment/backend-сервера.

Основной путь:

```text
MainLogic
    │
    ▼
taskGetInfoHandler
    │
    ▼
Formatter.Process(frame)
    │
    ▼
parseInfoXml
    │
    ▼
terminal identity / services / account / balance / limits / blocking
```

Получение информации выполняется периодически.

Подробности:

```text
docs/INFO_XML_STATUS.md
```

---

## 9. Требования

Основная тестовая платформа:

```text
Hardware: E3000N
OS: Ubuntu 24.04.3 Desktop
Runtime: Mono 6.8
Database: MySQL 8
Browser: Chromium
```

Также используются:

* Python 3;
* Git;
* curl;
* MySQL client;
* systemd;
* X11;
* serial/UART infrastructure.

Исходный проект ориентирован на .NET Framework 4.6.1.

Текущий `MainLogic.dll`:

```text
Assembly version: 7.0.0.0
Target framework: .NET Framework 4.6.1
Runtime: Mono 6.8
```

---

## 10. Установка и запуск

Основной production-like тестовый deployment находится на отдельном Ubuntu E3000N.

Типовая структура:

```text
/home/terminal/LogicTest/app/
```

Основной запуск выполняется через:

```text
mono Logic.exe
```

Для автоматического запуска используется systemd/autostart infrastructure.

Основной E2E test script:

```text
/root/Logic/start_logic_linux_test.sh
```

Подробности deployment и окружения:

```text
docs/UBUNTU_TEST_ENVIRONMENT.md
docs/DEPLOYMENT_STATUS.md
```

Не следует считать локальную рабочую директорию `/root/Logic/MonoDebugTemp/` production deployment.

---

## 11. Проверка и тестирование

Перед изменением проекта необходимо понимать существующие baseline и результаты тестов.

Основные проверенные направления:

* application startup;
* database connectivity;
* terminal identity;
* service loading;
* HTML UI;
* WebSocket transport;
* MoveNext;
* MoveBack;
* Admin;
* Event;
* Barcode;
* Cardreader;
* Print;
* language switching;
* logging;
* Info XML;
* payment mock;
* monitor mock;
* autostart;
* restart/recovery;
* load testing.

Phase 4C load test:

```text
3 waves
5 clients per wave
15/15 full-sequence clients
195 messages
No systemd restart during load
```

### Известный robustness issue

Обнаружен сценарий:

```text
10 simultaneous raw moveNext
        │
        ▼
NullReferenceException
        │
        ▼
FATAL
        │
        ▼
Logic process exits
        │
        ▼
systemd restart
```

При нормальном одиночном сценарии и установленной паузе между UI actions проблема не воспроизводится.

Это остаётся кандидатом на исправление в Phase 5.

Подробности:

```text
docs/TEST_MATRIX.md
docs/RISKS_AND_BLOCKERS.md
docs/PHASE4C_HARDWARE_AUTOSTART_LOAD_REPORT.md
```

---

## 12. Документация

**`docs/` является источником истины для технической документации проекта.**

Основные документы:

```text
docs/
├── README.md
├── PROJECT_STATUS.md
├── PROJECT_TIMELINE.md
├── ARCHITECTURE_CURRENT.md
├── DATABASE_STATUS.md
├── HTML_UI_MIGRATION_STATUS.md
├── NETWORK_PAYMENT_STATUS.md
├── INFO_XML_STATUS.md
├── DEVICE_STATUS.md
├── UBUNTU_TEST_ENVIRONMENT.md
├── DEPLOYMENT_STATUS.md
├── TEST_MATRIX.md
├── RISKS_AND_BLOCKERS.md
├── SECURITY_STATUS.md
├── BASELINES.md
└── ...
```

Перед изменением проекта AI должен:

1. Прочитать этот `README.md`.
2. Найти соответствующий документ в `docs/`.
3. Изучить текущий статус и существующие ограничения.
4. Проверить source code и тестовые evidence при необходимости.
5. Только после этого предлагать или выполнять изменения.

Подробная техническая информация не должна дублироваться в README.

README предназначен для навигации, а не для хранения всей технической истории проекта.

---

## 13. Источник истины и WIKI

Для проекта используется следующая модель:

```text
Logic repository
        │
        └── docs/
              │
              │ synchronization
              ▼
       WIKI-LLM repository
              │
              └── projects/Logic/
```

### Источник истины

```text
Logic/docs/
```

### Общая AI knowledge base

```text
WIKI-LLM/projects/Logic/
```

Документы из `Logic/docs/` могут синхронизироваться в центральную WIKI для использования другими AI/агентами.

**WIKI не заменяет Git-документацию проекта.**

При конфликте project-specific информации приоритет имеет актуальное содержимое:

```text
Logic/docs/
```

---

## 14. Ограничения и известные проблемы

Текущие ограничения:

* физическая CashCode validation ещё не завершена;
* физическая Citizen validation ещё не завершена;
* Phase 5 robustness fixes ещё не завершены;
* реальный production Monitor не используется в тестовом E2E;
* реальный production Payment не используется в тестовом E2E;
* mock servers применяются только для контролируемого тестирования;
* legacy Flash остаётся в исходной системе как отдельный legacy-компонент;
* часть старого Windows-specific behavior реализуется compatibility layer;
* текущий Linux runtime основан на Mono.

### Flash

Исходная Windows UI цепочка:

```text
Logic.exe
    │
    ▼
MainLogic.dll
    │
    ▼
LogicForm / MainForm
    │
    ▼
FlashControl.cs
    │
    ▼
PSCB.FlashControl.FlashPlayer
    │
    ▼
ShockwaveFlash ActiveX
    │
    ▼
Adobe Flash Player
    │
    ▼
FlashLogic.swf
```

Flash не является основой основного HTML UI.

Основной HTML UI использует HTML/JavaScript/CSS/PNG assets.

Поэтому наличие Flash в исходном архиве само по себе не означает необходимость полного UI rewrite.

---

## 15. Безопасность

Проект содержит конфиденциальные материалы.

Следующие категории **не должны попадать в Git**:

```text
Terminal.zip
logiclinux-master.zip
db/
backup_*/
OriginalLinux/
production database dumps
production credentials
production private keys
cashin.keys
SSH credentials
.tunnel credentials
generated secrets
temporary test credentials
```

Особенно важно:

> Production private keys, database credentials и исходные коммерческие архивы никогда не должны публиковаться в repository или WIKI.

Перед commit необходимо проверить:

```text
git status
git diff
git ls-files
```

и убедиться, что секретные материалы не добавлены.

Подробности:

```text
docs/SECURITY_STATUS.md
```

---

## 16. Baselines

Существующие baseline нельзя изменять или заменять без явного разрешения.

Особенно важен frozen Phase 3 PoC baseline:

```text
/root/Logic/backup_20260903_144844_phase3_poc_baseline/
```

Baseline содержит:

```text
167 files
SHA256 manifest
167/167 verified
```

Baseline используется для сравнения и regression control.

Подробности:

```text
docs/BASELINES.md
```

---

## 17. Правила для AI / WIKI

AI-агент, работающий с проектом, обязан соблюдать следующие правила.

### Перед началом работы

1. Прочитать `README.md`.
2. Определить, какой компонент затрагивается.
3. Прочитать соответствующие документы из `docs/`.
4. Проверить существующий baseline.
5. Проверить текущий статус тестов.
6. Не делать предположений о поведении системы, если его можно проверить по source/evidence.

### При изменении кода

* Сначала определить существующий механизм.
* Не создавать новую архитектуру, если существующая может быть адаптирована.
* Не делать полный rewrite без доказанной необходимости.
* Не удалять legacy code без проверки зависимостей.
* Не заменять baseline без явного разрешения.
* Не изменять production infrastructure во время обычного тестирования.
* Не подключать тестовое приложение к production database.
* Не использовать production credentials или keys в тестовых артефактах.

### При работе с устройствами

* Не подменять физическое устройство mock-ом, если задача требует hardware validation.
* Не создавать fake `/dev/tty*`.
* Не маскировать ошибки serial communication.
* Не выполнять реальные финансовые операции без явного разрешения.
* Не выполнять необратимые команды устройства без явного разрешения.

### При работе с базой

* Сначала использовать изолированную test database.
* Не выполнять destructive SQL на production database.
* Не replay-ить database/binlog в live database без отдельной restore target.
* После существенных изменений проверять database integrity.

### После изменения

AI должен:

1. Запустить соответствующие тесты.
2. Сравнить результат с baseline.
3. Зафиксировать обнаруженные изменения.
4. Обновить соответствующий документ в `docs/`.
5. Не записывать неподтверждённые предположения как факты.

---

## 18. Evidence Principle

Для проекта действует принцип:

> **Evidence over assumption.**

Приоритет источников:

```text
1. Actual source code
2. Reproducible test result
3. Runtime logs
4. Verified configuration
5. Existing technical documentation
6. Historical notes
7. Assumption / inference
```

Если поведение системы неизвестно, AI должен явно обозначить это как неизвестное и предложить способ проверки.

Нельзя превращать предположение в утверждение только потому, что оно выглядит логичным.

---

## 19. Главное правило проекта

Основная цель Linux-порта:

```text
Preserve behavior
        +
Replace platform-specific dependencies
        +
Verify with evidence
        =
Reliable Linux port
```

Не следует переписывать систему ради переписывания.

Приоритет:

```text
Existing behavior
        ↓
Compatibility
        ↓
Controlled migration
        ↓
Regression testing
        ↓
Cleanup / optimization
```

---

## 20. Быстрый вход для AI

Если AI впервые работает с проектом, рекомендуемый порядок чтения:

```text
README.md
    ↓
docs/PROJECT_STATUS.md
    ↓
docs/ARCHITECTURE_CURRENT.md
    ↓
docs/TEST_MATRIX.md
    ↓
docs/RISKS_AND_BLOCKERS.md
    ↓
документ конкретного компонента
```

Для задач:

```text
Database
    → docs/DATABASE_STATUS.md

HTML / UI
    → docs/HTML_UI_MIGRATION_STATUS.md

Network / Payment
    → docs/NETWORK_PAYMENT_STATUS.md

Info XML
    → docs/INFO_XML_STATUS.md

Devices
    → docs/DEVICE_STATUS.md

Ubuntu / deployment
    → docs/UBUNTU_TEST_ENVIRONMENT.md
    → docs/DEPLOYMENT_STATUS.md

Security
    → docs/SECURITY_STATUS.md

Tests
    → docs/TEST_MATRIX.md

Known problems
    → docs/RISKS_AND_BLOCKERS.md
```

---

## 21. Current conclusion

Linux-порт Logic уже прошёл основные этапы доказательства работоспособности:

* backend запускается на Ubuntu/Mono;
* database layer адаптирован под MySQL;
* HTML UI работает через Chromium;
* legacy IE-dependent UI behavior адаптирован compatibility layer;
* WebSocket transport работает;
* E2E сценарии воспроизводятся;
* autostart/recovery проверены;
* load testing проведён.

Оставшиеся задачи относятся преимущественно к:

```text
Physical hardware validation
        +
Robustness fixes
        +
Final cleanup
        +
Release/security finalization
```

Подробный текущий статус всегда смотреть в:

```text
docs/PROJECT_STATUS.md
```

а не в этом README.
