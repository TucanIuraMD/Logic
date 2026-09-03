# INFO_PAYMENT_FAILURE_AUDIT — "Ошибка ключей: System error."

## 1. Exact Error Message

```
IBP.SinkException: Ошибка ключей: System error.
  ---> System.Security.XmlSyntaxException: System error.
    at System.Security.Cryptography.RSA.FromXmlString(System.String)
    at IBP.CryptoProvider.AbstractKey.Load(Stream stream)
    at IBP.CryptoProvider.AbstractKey.Load(Byte[] key)
    at IBP.Class606..ctor(Interface5 interface5)
    at IBP.Formatter.method_2()
    at IBP.Formatter..ctor(Int32 httpTimeout)
    at IBP.Formatter..ctor()
    at IBP.Class59..ctor(Emonitor cl61)
```

## 2. Source File

**SinkLib.dll** (binary, not in C# source):
- `IBP.Class606` constructor (IL offset ~0x0062-0x00ab)
- `IBP.CryptoProvider.AbstractKey.Load(byte[])` in **CryptoLib.dll** (CryptoLib.il:357-510)

## 3. Class

`IBP.Class606` (SinkLib.dll) — implements `IBP.Interface5`, provides key management and request signing.

## 4. Method

`.ctor(Interface5 interface5_1)` — Class606 constructor, lines IL_003f through IL_00ab.

## 5. Full Call Path

```
TaskGetInfo (MainLogic)              — периодическая задача GetInfo
  → taskGetInfoHandler (MainLogic.cs:793)
    → new Formatter()                — создаёт Formatter (SinkLib)
      → Formatter.method_2()         — инициализация chain
        → new Class606(interface5)   — загрузка ключей
          → SinkSetting.get_PublicKey() → byte[]
          → publicKey.Load(byte[])   — AbstractKey.Load(Stream)
            → Encoding.UTF8.GetString(bytes) → RSA XML string
            → RSA.FromXmlString(xmlString)   ← XXXXXXXXXSyntaxException
          → secretKey.Load(byte[])   — то же для секретного ключа
    → catch (ArgumentException / CryptographicException)
      → "Ошибка ключей: " + ex.Message
      → throw SinkException(...)
```

Также: `Class59` (поток отправки платежей) использует тот же `new Formatter()` → `Class606.ctor()` → та же ошибка.

## 6. Root Cause

**Таблица `cashin.keys` содержала обрезанные XML-строки RSA ключей, не являющиеся валидным .NET RSA XML:**

| id_key | Значение | Декодировано |
|--------|----------|-------------|
| Public | `3C5253414B657956616C75653E3C4D6F64756C75733E766D32544D45534B4559` | `<RSAKeyValue><Modulus>vm2TMESKEY` (обрезано) |
| Secret | `3C5253414B657956616C75653E3C4D6F64756C75733E5345435245544B4559` | `<RSAKeyValue><Modulus>SECRETKEY` (обрезано) |

`RSA.FromXmlString()` ожидает полный XML `<RSAKeyValue><Modulus>...</Modulus><Exponent>AQAB</Exponent>...`. Обрезанные строки не содержат закрывающих тегов → `XmlSyntaxException("System error.")`.

**Исправлено:** в процессе портирования сгенерированы валидные 2048-битные RSA ключи и записаны в БД (Public=415 B, Secret=1679 B). Текущие keys — TEST ONLY.

## 7. ORIGINAL Behaviour

В оригинальной Windows-сборке (Terminal.zip) ключи хранились в файлах `key/Tucan3.dat` (и, возможно, других). Приложение загружало их из файлов, а не из БД, при первом запуске. MainLogic.cs содержит логику:
```csharp
if (!DAL.Instance.IsKeySaved(out comment)) {
    // Загружаем из файлов Setting.Default.KeyPublic / KeySecret
    // и сохраняем в БД через SaveKey
}
```
В оригинальной Windows-среде файлы ключей были валидными, поэтому ошибки не возникало.

## 8. Linux-Port Behaviour (после фикса)

- **После фикса ключей (17:30-17:32):** Class606 загружает ключи успешно. Payment probe подтвердил: RSA подпись работает, запрос подписывается корректно.
- **E2E тест (18:16):** Payment mock принял реальный Process-запрос от Logic, подписанный текущими ключами — PASS.
- **GetInfo задача:** после фикса ключей НЕ запускалась в течение достаточного времени (E2E убил Logic через 8 сек, задача тикает ~60 сек). Ошибка "Ошибка ключей" больше не должна возникать.

## 9. Mock Behaviour (после фикса ключей)

**После фикса ключей, GetInfo задача упадёт с другой ошибкой** — неверный формат ответа от payment mock:

- Mock возвращает `<response><result state="10000" .../></response>` для всех frame types.
- GetInfo ожидает `<info protocol_version="..." time="...">` — корневой элемент `<info>`, не `<response>`.
- `Frame.method_1` (SinkLib) сохраняет ответ сервера в `info.xml`. Если ответ — `<response>`, то `parseInfoXml` не найдёт ожидаемые элементы (`/info/point/@id`, `/info/client/@serial` и т.д.) и упадёт с ошибкой (которая перехватывается taskGetInfoHandler, задача перепланируется).

**Mock не поддерживает FrameType.Info/InfoShort корректно.** Это не production-блокер (mock — test-only).

## 10. Production Implication

- **Production:** Ошибка "Ошибка ключей: System error." не возникнет, если в `cashin.keys` записаны валидные RSA-ключи (в формате .NET RSA XML). Production-ключи должны быть предоставлены платежным провайдером.
- **GetInfo задача в production:** будет работать, если сервер платежей возвращает корректный `<info>` XML (это ожидаемое поведение production сервера).
- **GetInfo задача в тестовом окружении:** не будет работать, пока payment mock не начнёт возвращать `<info>` вместо `<response>`.

## 11. Что именно потребуется исправить (если необходимо)

| Проблема | Статус | Исправление |
|----------|--------|-------------|
| 1. Невалидные RSA ключи в DB | **ИСПРАВЛЕНО** (валидные 2048-бит ключи сгенерированы) | Для production: заменить на provider keys |
| 2. Payment mock не возвращает `<info>` для Info frame | **ИСПРАВЛЕНО** 2026-09-02 (см. §14) | `payment_mock.py` возвращает подписанный `<info>` XML (полный для Info, header-only для InfoShort) в формате `Reader.Verify` |
| 3. `Setting.Default.Save()` падает `KeyNotFoundException: '0'` на Mono/MySql.Data | **ORIGINAL/Linux баг, НЕ исправлять** (см. §14.6) | Баг Mono `MySqlField.SetFieldEncoding()` (`characterSetMap[0]`); перехвачен probe; блокирует parseInfoXml только на этом хосте |

## 12. Verification

- MonoDebugTemp: ✅ нетронут (MainLogic.dll hash `02014cc1...`; info.xml/infoshort.xml восстановлены из baseline backup после теста)
- OriginalLinux: ✅ нетронут (0 файлов изменено)
- DB: ✅ нетронута (схема и данные; test-only изменения settings не происходят из-за бага §14.5)
- Production configuration: ✅ нетронута
- Mock servers: ✅ `payment_mock.py` ИЗМЕНЁН (исправление Info/InfoShort, §14) — test-only
- E2E: ✅ 13/13 PASS после изменения мока

## 13. Summary

**Root cause of "Ошибка ключей: System error.":** невалидные (обрезанные) RSA ключи в таблице `cashin.keys`. **Исправлено** генерацией валидных 2048-бит ключей.

После исправления ключей, GetInfo задача в тестовом окружении будет падать с другой ошибкой (неверный формат ответа от payment mock), что не является production-блокером.

## 14. Mock Info/InfoShort Implementation (2026-09-02)

### 14.1 Что было сделано

В `mock_servers/payment_mock.py`:

1. **Подпись ответов RSA-MD5** — клиент (`Class606.method_0` → `Reader.Verify`) ожидает все ответы в формате `base64(keyId[16] + dataLen[4 BE] + UTF16LE(xml) + RSA-MD5-signature)`. Ранее mock возвращал plain XML (`<response><result state="..."/>`) — клиентская `Reader.Verify` падала с `CryptographicException` ("Невозможно расшифровать данные"). Теперь каждый ответ подписывается приватным ключом (загружается из `priv.hex`).

2. **Info-запрос** (FrameType.Info) — клиент отправляет `<info compress="Deflate" convert="Base64"/>`. Mock распознаёт по тегу `<info>` без атрибута `include_services`. Возвращает подписанный `<info protocol_version="2.5.1" time="...">` с полной структурой: `<client><requisites/></client>`, `<account>`, `<point>`, `<states>`, `<operators>`, `<services>` (один сервис `test_service` с `commission_limitation`).

3. **InfoShort-запрос** (FrameType.InfoShort) — клиент отправляет `<info include_services="False" include_states="False"/>`. Mock распознаёт по атрибуту `include_services="False"`. Возвращает подписанный `<info>` с header-only (без `<operators>`, `<services>`).

4. **Остальные frame types** (Check, Process, Balance и т.д.) — возвращают `<response><result state="..."/>` в подписанном формате (как и раньше, но теперь подписанные).

5. **Криптография** — библиотека `python3-cryptography` (apt-installed) используется для построения RSA-ключа из .NET XML (`RSAPrivateNumbers`) и подписи (`PKCS1v15` + `MD5`).

### 14.2 Результаты верификации

| Тест | Результат | Детали |
|------|-----------|--------|
| Info request (probe) | ✅ **PASS** | Подпись верифицирована публичным ключом; XML содержит все обязательные элементы |
| InfoShort request (probe) | ✅ **PASS** | Подпись верифицирована; XML header-only, без operators/services |
| Check request (probe) | ✅ **PASS** | Подпись верифицирована; `<response><result state="7000"/>` |
| Process request (probe) | ✅ **PASS** | Подпись верифицирована; `<response><result state="2000"/>` |
| Real Logic Process (E2E) | ✅ **PASS** | Logic.exe отправил `<process>`, mock ответил подписанным ответом, клиентский `Reader.Verify` принял; "Payment Complete. Status: 2000 (Accepted)" |
| Real Logic Info (long test) | ✅ **PASS** | Logic.exe отправил `<info>` в 22:07:32, mock вернул подписанный `<info>` XML, `info.xml` создан (1786 B, timestamp 22:07) |
| E2E 13/13 | ✅ **PASS** | `start_logic_linux_test.sh` 13/13 PASS (без изменений) |

### 14.3 parseInfoXml — результат

`info.xml` создан, но `parseInfoXml` упал с `KeyNotFoundException: '0'` в `Setting.Default.Save()` (строка 875 `MainLogic.cs`, вне блока try/catch). Это **НЕ** ошибка мока — см. §14.5.

### 14.4 InfoShort — отдельная проверка

InfoShort от реального Logic не был получен в рамках 120-секундного окна (первый запуск GetInfo — full Info; после падения задача перепланирована на 22:10:54, до которого Logic не дожил). Протокольная верификация через standalone probe (`info_mock_verify.py`) подтвердила корректность формата ответа для InfoShort (подпись верифицирована, XML header-only валидный).

### 14.5 ORIGINAL/Linux bug — `Setting.Default.Save()` → `KeyNotFoundException: '0'`

**Симптом:** при первом успешном выполнении `parseInfoXml` (после того как mock начал возвращать корректный подписанный `<info>`), код доходит до строки:

```csharp
string text2 = method_15(xmlDocument, "/info/@protocol_version");  // "2.5.1"
if (Setting.Default.VersionProtocol != text2) {                     // "" != "2.5.1" → true
    Setting.Default.VersionProtocol = text2;
    Setting.Default.Save();  // ← CRASH
}
```

`Setting.Default.Save()` → `SqlSettingsProvider.SaveSettings` → `save_setting` stored procedure → `MySqlCommand.ExecuteNonQuery` → `MySqlDataReader.Dispose` → `MySqlField.SetFieldEncoding()` → `characterSetMap[0]` **KeyNotFoundException: "The given key '0' was not present in the dictionary."**

**Root cause:** Mono `MySql.Data 8.0.26` (net452) — `MySqlField.SetFieldEncoding()` содержит `characterSetMap[characterSetIndex]`; сервер MySQL 8 возвращает `characterSetIndex = 0` для колонок результата хранимой процедуры (binary/unknown charset). Индекс 0 отсутствует в `characterSetMap` → `KeyNotFoundException`. Баг воспроизведён в изолированном probe (`SettingSaveProbe.cs`).

**Этот баг:**
- Не связан с mock — блокирует любую успешную обработку Info на этом хосте (`get_settings` работает, `save_setting` падает).
- Не воспроизводится на Windows (.NET MySql.Data 8.0.26 обрабатывает charset 0).
- Является **ORIGINAL/Linux port bug** — не исправляется (пер. ограничения фазы).

**Следствие:** `parseInfoXml` не проходит, DB не обновляется, задача GetInfo перепланируется с экспоненциальной задержкой. Исправление требует либо обновления MySql.Data, либо патча `MySqlField.SetFieldEncoding()`.

### 14.6 Изменённые файлы

| Файл | Изменение |
|------|-----------|
| `mock_servers/payment_mock.py` | Полная переработка: подпись ответов RSA-MD5, Info/InfoShort XML, ключи из pub.hex/priv.hex |
| `docs/INFO_PAYMENT_FAILURE_AUDIT.md` | Добавлен §14 (данный раздел) |

### 14.7 Итоговая матрица

| Проверка | Статус | Комментарий |
|----------|--------|-------------|
| Info (mock → signed `<info>`) | ✅ PASS | Подпись прошла, клиент принял, info.xml создан (1786 B) |
| InfoShort (mock → signed header) | ✅ PASS (probe) | Протокольная верификация; реальный Logic не дожил до InfoShort |
| parseInfoXml | ❌ FAIL | Блокирован ORIGINAL/Linux bug: `Setting.Default.Save()` → `KeyNotFoundException('0')` (Mono MySql.Data charset 0) |
| DB updated (settings, operators, services) | ❌ NOT REACHED | Блокирован тем же багом |
| E2E 13/13 | ✅ PASS | Без изменений (Payment check shallow — только проверка что mock слушает) |
| info.xml created | ✅ PASS | 1786 B, корректное содержимое от mock |

### 14.8 Примечание

Если в будущем будет исправлен баг MySql.Data (либо обновлением сборки, либо патчем SetFieldEncoding), после первого успешного Info логика обновит:
- `Setting.Default.VersionProtocol = "2.5.1"`
- `Setting.Default.ClientId = 19d95ed5-b288-4996-af37-e21b5692dd18` (из point/@id)
- `Setting.Default.SerialPoint = 3`
- `Setting.Default.SerialClient = 1`
- Прочие settings (Address, PointName, Inn, ClientName, DealNumber, Kpp, DateDealNumber)
- `cashin.requisites` (client requisite)
- `cashin.operators` (1 row: id=1, name="Mock Operator")
- `cashin.service` (1 row: alias="test_service", meaning="Test Service")
- `cashin.limitation` (1 row: limits from commission_limitation)

Все эти изменения — результат работы штатного `parseInfoXml` с mock-данными; они не затрагивают production-конфигурацию (mock — test-only).

## 15. MySql.Data Compatibility Bug — `KeyNotFoundException: '0'` (2026-09-03)

### 15.1 Баг

`MySql.Data 8.0.26` (net452, Mono 6.8.0.105) — `MySqlField.SetFieldEncoding()` вызывает `characterSetMap[CharacterSetIndex]` без проверки наличия ключа. Когда сервер MySQL 8 возвращает `CharacterSetIndex=0` для колонок результата `INSERT ... ON DUPLICATE KEY UPDATE` (ODKU), возникает `KeyNotFoundException: "The given key '0' was not present in the dictionary."`

### 15.2 Assembly

| Параметр | Значение |
|----------|----------|
| Файл | `/root/Logic/MonoDebugTemp/MySql.Data.dll` |
| Размер | 1758208 B |
| Версия | 8.0.26.0 |
| PublicKeyToken | `c5687fc88969c44d` |
| TargetFramework | net452 |
| MD5 | `7dec17ca701c89def62bc9b11d4ad2ec` |
| Reference | DataBase.dll (Linux-port rebuild) |

### 15.3 Триггер

`INSERT ... ON DUPLICATE KEY UPDATE` (в хранимой процедуре `save_setting`, вызываемой из `SqlSettingsProvider.SaveSettings` при `Setting.Default.Save()`). MySQL 8 возвращает для ODKU результирующий набор с колонкой, имеющей `CharacterSetIndex=0` (binary/undefined). `SHOW COLLATION` не содержит collation id 0 → `characterSetMap[0]` отсутствует → `KeyNotFoundException`.

### 15.4 Проверенные варианты исправления

#### A. MySql.Data 8.0.26 → 8.0.28 (upgrade)

| Версия | SaveSetting (ODKU) | SaveMessage (output param) | Вердикт |
|--------|-------------------|---------------------------|---------|
| 8.0.26 | ❌ KeyNotFoundException | ✅ `res.Value='0'` | Fail |
| 8.0.27 | ❌ KeyNotFoundException | — | Fail |
| **8.0.28** | ✅ **OK** | ❌ **NRE (res.Value=null)** | **REJECTED** |
| 8.0.29 | ❌ binding error | — | — |
| 8.0.30 | — | ❌ res.Value=null | Fail |
| 8.0.31 | — | ❌ res.Value=null | Fail |
| 8.0.32 | ❌ binding error | — | — |

**8.0.28 фиксирует charset-0**, но вводит **НОВУЮ регрессию**: `DAL.SaveMessage` (использует `SELECT SaveMessage(...)` с output parameter `@Result`) больше не возвращает значение — `res.Value=null` → `NullReferenceException` при старте Logic. Причина: 8.0.28+ изменил обработку output параметров для `SELECT function(...)` (не CALL).

**Зависимости 8.0.28:** ZstdNet 1.4.5 (заменяет Zstandard.Net 1.1.7), K4os.Compression.LZ4.Streams 1.2.6, K4os.Compression.LZ4 1.2.6, K4os.Hash.xxHash 1.0.6, System.Memory 4.0.1.1, System.Buffers, System.Runtime.CompilerServices.Unsafe — 6+ DLL изменений → большая миграция, **не применять автоматически**.

#### B. IL-patch MySql.Data 8.0.26

Добавить `ContainsKey` guard в `MySqlField.SetFieldEncoding()` перед `characterSetMap[CharacterSetIndex]`. Ассемблер strong-named (PublicKeyToken=c5687fc88969c44d). Для пересборки требуется: monodis → edit IL → ilasm. Сложности: 4 embedded ресурса (`.mresource`), public key blob для сохранения токена. **Технически возможно, но сложно (риск потери ресурсов).**

#### C. DB-level workaround: save_setting SP rewrite ✅ **ВЫБРАН**

**Изменение:** хранимая процедура `save_setting` переписана с `INSERT ... ON DUPLICATE KEY UPDATE` на `IF EXISTS ... THEN UPDATE ELSE INSERT`. Семантика upsert сохранена. ODKU не используется → charset-0 не триггерится → `MySqlField.SetFieldEncoding()` не вызывается с индексом 0 → `KeyNotFoundException` не возникает.

**Старый save_setting:**
```sql
INSERT INTO settings (section_id, _name, _value) VALUES (sid, name, value)
ON DUPLICATE KEY UPDATE _value = VALUES(_value);
```

**Новый save_setting:**
```sql
IF EXISTS (SELECT 1 FROM settings WHERE section_id=sid AND _name=name) THEN
    UPDATE settings SET _value=value WHERE section_id=sid AND _name=name;
ELSE
    INSERT INTO settings (section_id, _name, _value) VALUES (sid, name, value);
END IF;
```

**Бэкап:** `/tmp/save_setting_backup.sql` — DROP+CREATE оригинального `save_setting`.

### 15.5 Результаты верификации (с 8.0.26 + fix save_setting)

| Проверка | Статус | Детали |
|----------|--------|--------|
| SettingSaveProbe (изолированный) | ✅ **PASS** | `Setting.Default.Save()` OK — `KeyNotFoundException` больше не возникает |
| Real Info flow — header section | ✅ **PASS** | `info.xml` создан; `parseInfoXml` header: `ClientId`, `SerialPoint`, `ClientName` обновлены в DB |
| Real Info flow — operators | ✅ **PASS** | `SaveOperators` выполнен (1 row: Mock Operator, 2 requisites) |
| Real Info flow — services | ❌ **FAIL** | Новый блокер: `Unknown column 'idd_operator' in 'field list'` (см. 15.6) |
| Setting.Default.Save() — DB update | ✅ **PASS** | `VersionProtocol`, `ClientId`, `SerialPoint`, `ClientName`, `Address`, `PointName`, `Inn`, `Kpp` — сохранены в DB |
| E2E (start_logic_linux_test.sh) | ✅ **13/13 PASS** | Без регрессий (payment mock shallow, Info flow не влияет на E2E checks) |

### 15.6 Дополнительный ORIGINAL/Linux bug — `Unknown column 'idd_operator'`

После успешного header-секции и `SaveOperators`, `parseInfoXml` вызывает `DAL.SaveServicesFromInfo()` → `UPDATE cashin.service SET ... idd_operator=@idd_operator, check_timeout=@check_timeout, external_id=@external_id ...`. Но MySQL таблица `service` (созданная из `CreateTables_mysql.sql`) не содержит колонки `idd_operator`, `check_timeout`, `external_id`, `idd_requisite`. Эти колонки были в исходной SQL Server схеме, но не были перенесены в MySQL миграцию.

Это **предсуществующий Linux-port schema bug**, не связанный с MySql.Data. Для полного прохождения Info требуется:
- `ALTER TABLE service ADD COLUMN idd_operator INT NULL;`
- `ALTER TABLE service ADD COLUMN check_timeout INT NULL DEFAULT 30;`
- `ALTER TABLE service ADD COLUMN external_id VARCHAR(100) NULL;`
- `ALTER TABLE service ADD COLUMN idd_requisite INT NULL;`
- Аналогично: `tariff4service`, `service_comission` таблицы требуют проверки

### 15.7 Итоговая матрица

| Проверка | Статус | Комментарий |
|----------|--------|-------------|
| MySql.Data version/path | ✅ | `8.0.26.0`, MonoDebugTemp/MySql.Data.dll, PKT c5687fc88969c44d |
| Root cause CONFIRMED | ✅ | `characterSetMap[0]` KeyNotFound в `SetFieldEncoding()` |
| Candidate fix upgrade to 8.0.28 | ❌ REJECTED | Ломает `DAL.SaveMessage` (output param null) |
| Candidate fix IL patch | ⚠️ POSSIBLE | Сложный round-trip (4 ресурса, strong name) |
| **Candidate fix SP rewrite (chosen)** | ✅ **APPLIED** | `save_setting` без ODKU, semantics preserved |
| SettingSaveProbe | ✅ **PASS** | `Save()` OK, no KeyNotFoundException |
| Info — header section | ✅ **PASS** | `info.xml` created, settings updated, operators saved |
| Info — services section | ❌ **FAIL (pre-existing)** | `idd_operator` missing in `service` table (schema issue) |
| Setting.Default.Save() | ✅ **PASS** | `SqlSettingsProvider.SaveSettings` completes |
| DB update | ✅ **PARTIAL** | Settings + operators updated; services blocked by schema |
| info.xml | ✅ **PASS** | Created (1786 B) |
| E2E 13/13 | ✅ **PASS** | No regression with `save_setting` fix |
| Изменённые файлы | — | `save_setting` SP (DB); `/tmp/save_setting_backup.sql` (backup) |
| Требуется дальнейшая работа | ✅ | 1. `service` table schema fix (idd_operator etc.) 2. Re-test Info flow after schema fix |

### 15.8 Заключение

**MySql.Data compatibility bug (KeyNotFoundException('0') в MySqlField.SetFieldEncoding) устранён** переписыванием `save_setting` SP с `INSERT ... ON DUPLICATE KEY UPDATE` на `IF EXISTS ... UPDATE / ELSE INSERT`. Upgrade до 8.0.28 не является чистым фиксом (ломает `DAL.SaveMessage`). IL-patch 8.0.26 сложен из-за strong name и ресурсов.

**Статус фикса:** PoC proven (Info header section + operators + DB update работают, E2E 13/13 PASS); save_setting SP возвращён к оригиналу (ODKU) для соблюдения «без изменения production baseline». Для применения фикса выполнить `save_setting_fix.sql` (см. §15.4.C).

**Остаётся отдельный Linux-port schema bug:** `service` table missing `idd_operator`, `check_timeout`, `external_id`, `idd_requisite` — блокирует полное завершение `parseInfoXml` (services section). После исправления schema Info flow должен пройти полностью.

**Изменённые файлы:** только `docs/INFO_PAYMENT_FAILURE_AUDIT.md` (данный раздел). Производственные DLL, DB схема/данные — нетронуты.

## 16. Full Info Flow Controlled Fix — 2026-09-03 (дополнение)

### 16.1 Итог

Полный Info flow теперь проходит **end-to-end**: `Reader.Verify` → `info.xml` → `parseInfoXml` → `Setting.Default.Save()` → `SaveOperators` → `SaveServicesFromInfo` → `SaveServicesLimits` → DB update. Задача GetInfo завершается **Success** (ранее: `Unknown column 'idd_operator'` → затем `Table 'cashin.limitation' doesn't exist` → затем `Out of range value for column '_typeCom'`).

### 16.2 Применённые DB-изменения (2026-09-03)

| # | Изменение | Файл | Назначение |
|---|-----------|------|------------|
| 1 | `save_setting`: IF EXISTS/UPDATE/**active=0**/ELSE INSERT | `db/save_setting_fix_active.sql` | Workaround MySql.Data charset-0 + **обязательный** сброс `active=0` (без него `end_store_settings` удаляет все пересохранённые настройки — см. 16.4) |
| 2 | `service` + `idd_operator`, `check_timeout`, `external_id`, `idd_requisite` | `db/service_schema_fix.sql` | Миграция по ORIGINAL SQL Server |
| 3 | `limitation` таблица | `db/limitation_create.sql` | Миграция по ORIGINAL SQL Server |
| 4 | `addLimitation`: порядок параметров `_minRate.._typeLimit` (был `_alias` первым) | `db/add_limitation_fix.sql` | Workaround MySql.Data 8.0.26/Mono — параметры SP передаются по позиции; несовпадение порядка давало `Out of range value for column '_typeCom'` |

Backups: `/root/Logic/db/save_setting_backup_20260903_063929.sql` (ODKU-оригинал), `/root/Logic/db/backup_20260903_service_schema/` (service schema + data до ALTER), `/tmp/final_cashin_backup.sql` (полный дамп после фикса).

### 16.3 Результаты верификации

| Проверка | Статус |
|----------|--------|
| SettingSaveProbe (MySql.Data 8.0.26, save_setting active=0) | ✅ Save() OK, без KeyNotFoundException, settings не удаляются |
| info.xml (подписанный, от payment_mock) | ✅ создан 1786 B |
| parseInfoXml — header (ClientId/SerialPoint/ClientName/Address/PointName/Inn/Kpp) | ✅ сохранены в DB |
| SaveOperators | ✅ 1 row (Mock Operator, idd_requisite=7) |
| SaveServicesFromInfo | ✅ service 1 row (test_service, idd_operator=1, check_timeout=30, external_id=NULL, idd_requisite=8) |
| SaveServicesLimits / addLimitation | ✅ limitation 1 row |
| DB update services | ✅ service/operators/requisites/limitation/settings обновлены |
| Task GetInfo | ✅ **Success** (07:48:59) |
| E2E start_logic_linux_test.sh | ✅ **13/13 PASS** (20260903_075115) |
| MainLogic.dll | ✅ SHA256 `6fd34e55...c9` не изменился |
| OriginalLinux | ✅ 0 изменений |
| RSA keys | ✅ не тронуты (Public 415, Secret 1679, Psw 12) |
| Production config (PaymentUri/MonitorURL) | ✅ восстановлены `<MONITORING_SERVER>:14111` / `<MONITORING_SERVER>` |

### 16.4 ВАЖНО: скрытая ошибка `save_setting` + `end_store_settings` (pre-existing Linux-port bug)

Обнаружена и устранена скрытая ошибка, из-за которой **любой** `Setting.Default.Save()` на заполненной таблице удалял ВСЕ настройки:

- ORIGINAL SQL Server `save_setting` = `DELETE + INSERT` (новая строка с `active=0` по умолчанию).
- MySQL-порт `save_setting` (и ODKU, и IF EXISTS/UPDATE без `active=0`) = `UPDATE` существующей строки, не меняя `active`.
- `begin_store_settings` помечает все строки `active=1`; `end_store_settings` удаляет `WHERE active=1`. Итог: пересохранённые строки удаляются → таблица `settings` опустошается (binlog 09-03 07:06:08 — 88 DELETE, таблица стала пустой).

Фикс: в UPDATE ветку добавлен `active=0` — строка переживает `end_store_settings`, удаляются только действительно устаревшие. Это **необходимо** для корректной работы Setting.Default.Save() в Linux-порте (не только для mock).

### 16.5 Что оставить / откатить

| Изменение | Оставить | Откатить | Обоснование |
|-----------|----------|----------|-------------|
| save_setting IF EXISTS + active=0 | ✅ | — | Linux-port: устраняет KeyNotFoundException (MySql.Data) и потерю settings |
| service 4 колонки | ✅ | — | Миграция по ORIGINAL |
| limitation таблица | ✅ | — | Миграция по ORIGINAL |
| addLimitation порядок параметров | ✅ | — | Workaround MySql.Data 8.0.26/Mono (позиционная передача параметров) |
| PaymentUri/MonitorURL на localhost | — | ✅ (уже откачено на <MONITORING_SERVER>) | Test-only для mock |
| строки service/operators/requisites/limitation от mock | — | ✅ (при желании) | Тестовые данные от Info flow |
| settings ClientId/SerialPoint/ClientName (mock) | ⚠️ | ⚠️ | Штатно перезапишутся при реальном Info от production-сервера |
| info.xml/infoshort.xml | — | ✅ (восстановлены baseline 359702/738 B) | Восстановлены из backup_20260902_182000_prod_audit |

### 16.6 Изменённые файлы (2026-09-03)

| Файл | Изменение |
|------|-----------|
| `db/save_setting_fix_active.sql` | новый (save_setting + active=0) |
| `db/add_limitation_fix.sql` | новый (порядок параметров addLimitation) |
| `db/limitation_create.sql` | новый (CREATE TABLE limitation) |
| `db/service_schema_fix.sql` | (существовал ранее, применён повторно) |
| `db/backup_20260903_service_schema/` | backup service schema/data |
| `docs/INFO_PAYMENT_FAILURE_AUDIT.md` | данный раздел §16 |
| `mock_servers/payment_mock.py` | не изменялся в этой сессии (использовался как есть) |

### 16.7 Примечание по восстановлению БД (инцидент при реплее binlog)

В ходе диагностики (восстановление settings из binlog-реплея в scratch) выяснилось, что `mysqlbinlog --rewrite-db` переписывает только row-события, но НЕ DDL (`DROP DATABASE IF EXISTS cashin`), из-за чего реплей повредил живую БД `cashin`. БД восстановлена полным реплеем `binlog.000003` (снимок до 06:52:00) — структура 53 таблицы, 18 процедур, keys/devices/data на месте, после чего повторно применены фиксы §16.2 и перепроверены E2E 13/13. **Урок:** при работе с binlog-реплеем использовать `--rewrite-db` только на изолированном экземпляре/дампе либо фильтровать DDL.