# 06 — Database MySQL

## Доступ

- Host: `localhost` (bind 127.0.0.1)
- Port: 3306
- User: `terminal`@localhost
- Password: `<DB_PASSWORD>` (hardcoded в `ConnectionStringHelper.cs`)
- Database: `cashin` (и `cashin_test` — тестовая копия)
- Charset: utf8mb4
- Строка подключения (hardcoded):
  `server=localhost;userid=<DB_USER>;password=<DB_PASSWORD>;database=cashin;CharSet=utf8mb4;CheckParameters=false;`

## Гранты

```
GRANT USAGE ON *.* TO 'terminal'@'localhost'
GRANT ALL PRIVILEGES ON `cashin`.* TO 'terminal'@'localhost'
GRANT ALL PRIVILEGES ON `cashin_test`.* TO 'terminal'@'localhost'
```
(нет глобальных привилегий, bind только localhost — OK; в production отозвать `cashin_test`)

## Таблицы

| Таблица | Назначение |
|---------|-----------|
| settings | настройки приложения (`_name`, `_value`) |
| SettingsSection / SettingsAddon / SettingsAtm / DeviceSettings | доп. настройки |
| keys | RSA ключи: id_key ∈ {Public, Secret, Psw}, `_key` BLOB |
| devices | устройства (id_device, type, description, name) |
| properties4devices | свойства устройств (idd_device, _key, _value), UNIQUE(idd_device,_key) |
| payments | платежи (row_id auto, id, pointId, service, _value, _total, _account, input_date, sent, doc_number, _status, _balance, _source, _total_cheque, gateway, idd_incasso) |
| money | кассовая наличность (_nominal='amount_N', _global_count, _current_count, _sended_count) |
| money_history | история внесения купюр (_date, _nominal) |
| balance | баланс (id=1, balance) |
| messages / message_big / messagetomonitor | сообщения мониторинга |
| traffic / traffic_new | трафик сети |
| attr4payments | атрибуты платежей |
| card_payment | карточные платежи |
| fiscal_payment | фискальные данные |
| incasso | инкассация |
| operators / users | пользователи |
| languages | языки |
| help | справка |
| cheque / cheque_service | чеки |
| commissions | комиссии |
| barcodes | штрих-коды |
| defcodes | коды |
| tasks / attr4tasks | задачи |
| mess_step_status | статусы шагов |
| CashUnits | денежные единицы |

## Функции/процедуры

| Имя | Назначение |
|-----|-----------|
| `savePayment(row_id, id, pointID, service, _value, _total, _total_cheque, _account, input_date, doc_number, _balance, _source, gateway, idd_incasso)` | сохраняет платеж, возвращает row_id (TicketNumber) |
| `getSumMoney` | сумма наличных |
| `addMoney` / DAL.AddMoney (inline INSERT) | внесение купюры в money + money_history |
| `get_balance` (inline SELECT balance FROM cashin.balance) | баланс |
| `get_settings` | настройки секции |
| `MMPS_updateMainMenu` | главное меню |
| `checkSumMoneyAndPayments` | сверка наличных и платежей |
| `saveIncasso`, `saveModule`, `addLimitation`, `addRequisites`, `addServiceRequisite`, `addOperatorRequisite`, `begin_store_settings`, `end_store_settings`, `save_setting` | админ-операции |

## DAL (DataBase.dll)

- `IBP.DAL` singleton, `DAL.Instance`.
- Методы: `GetSettings()`, `GetDevices()`, `GetDeviceProperties()`, `GetDeviceByName()`, `SavePayment(ref Payment)`, `GetNotSentPayments()`, `SetSendedPaymet()`, `AddMoney(decimal)`, `GetCountsMoney()`, `SetBalance()/GetBalance()`, `GetTicketNumberPayment()`, `GetPublicKey()/GetSecretKey()/GetPsw()`, `SaveKey()`, `GetTasks()`, `GetListServices()`, `MMPS_*` и др.
- `DAL.ConnectionString` — статическое поле, устанавливается тестами.
- `WriteLogToFile` — пишет debug-лог в файл `C:\dabaseDLL_<date>.txt` (Linux: файл с именем, содержащим `C:\`, в рабочем каталоге).

## Схема

- Полный schema: `/root/Logic/db/script.sql`
- Дамп: `/root/Logic/db/cash-in.bak`
- Тестовая БД `cashin_test` — копия схемы для изолированного тестирования.

## Примечания

- `keys._key` — BLOB с UTF-8 bytes .NET RSA XML.
- `payments._status` — StatusPayment int (-1 Underfine, 0 New, 2 Ready, 2000 Accepted и т.д.).
- `payments.sent` — 0/1 (отправлен/не отправлен).
- Настройки читаются через `SqlSettingsProvider` (класс `IBP.Setting`), defaults заданы в `MainLogic/IBP/Setting.cs` через `DefaultSettingValue`.

## Schema-fixes 2026-09-03 (полный Info flow)

### `service` — добавлены 4 колонки (по ORIGINAL SQL Server, db/script.sql)

| Колонка | Тип | NULL/Default | Назначение |
|---------|-----|--------------|------------|
| `idd_operator` | INT | NULL | оператор услуги (FK-логика ORIGINAL, нет FK в MySQL) |
| `check_timeout` | INT | NOT NULL DEFAULT 30 | таймаут проверки услуги |
| `external_id` | VARCHAR(100) | NULL | внешний id (nullable, DBNull) |
| `idd_requisite` | INT | NULL | ссылка на requisites.id_requisite |

Файл: `/root/Logic/db/service_schema_fix.sql`. Колонки используются `DAL.SaveServicesFromInfo`, `addServiceRequisite`, `GetUsluga`.

### `limitation` — таблица (была в ORIGINAL SQL Server, отсутствовала в MySQL)

| Колонка | Тип |
|---------|-----|
| id_limitation | INT AUTO_INCREMENT PK |
| idd_service | INT NOT NULL (KEY FK_limitation_service → service.id_service) |
| start_date | DATETIME NOT NULL |
| min_rate / max_rate / min_amount / max_amount | DECIMAL(18,2) NULL |
| type_com / type_limit | INT NULL |
| flag_edit | TINYINT(1) NOT NULL DEFAULT 0 |

Файл: `/root/Logic/db/limitation_create.sql`. Используется `DAL.SaveServicesLimits` → SP `addLimitation` (Info flow, protocol >= 2.3.2).

### `save_setting` — IF EXISTS/UPDATE/INSERT + `active=0`

Workaround MySql.Data 8.0.26/Mono: ODKU (`INSERT ... ON DUPLICATE KEY UPDATE`) вызывает `KeyNotFoundException('0')`. Вариант IF EXISTS/UPDATE/INSERT устраняет это. **Обязателен сброс `active=0` в UPDATE** (ORIGINAL SQL Server делает DELETE+INSERT с active=0 по умолчанию; без сброса `end_store_settings` удаляет все пересохранённые настройки). Файл: `/root/Logic/db/save_setting_fix_active.sql`.

### `addLimitation` — порядок параметров

MySql.Data 8.0.26 на Mono передаёт параметры SP по позиции (не по имени). Порядок параметров `addLimitation` приведён к порядку вызова в `DAL.SaveServicesLimits` (`_minRate,_maxRate,_minAmount,_maxAmount,_typeCom,_startDate,_alias,_typeLimit`). Файл: `/root/Logic/db/add_limitation_fix.sql`.
