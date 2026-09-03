# 18 — Troubleshooting

## Startup / runtime

| Симптом | Причина | Решение |
|---------|---------|---------|
| "Ошибка ключей: System error." / `XmlSyntaxException` | невалидные RSA ключи в `cashin.keys` | Записать валидные .NET RSA XML ключи (hex UTF-8 bytes) в `Public`/`Secret` (см. 12_RSA_AND_KEYS) |
| "Мониторинг: ... Connection refused" | мониторинговый сервер недоступен | Запустить `monitor_mock.py` (test) или настроить реальный MonitorURL; проверить firewall |
| "Канал приема платежей ..." пусто/недоступен | payment сервер недоступен | Запустить `payment_mock.py` или настроить реальный PaymentUri |
| CashCode Open() COMPortException | /dev/ttyS0 отсутствует/нет прав | Проверить физический порт, udev rules, группу dialout; на dev использовать DB-слой тестов |
| Printer Open() ошибка | /dev/ttyS1 отсутствует | То же; для тестов использовать PrinterEMPTY |
| "=========== Logis is STARTED" не появляется | DB недоступен / конфигурация | Проверить MySQL, connection string, настройки |
| Лог не пишется в ожидаемый файл | LoggerLib создаёт каталог `<appDir>\logs` (обратный слэш в имени) | Проверить `/root/Logic/MonoDebugTemp\logs\` (с обратным слэшем) — файл там |
| "Блокировать — Fatal error, Printer Error" | принтер недоступен при старте | В dev-среде ожидаемо (нет HW); в production проверить принтер |

## DB

| Симптом | Причина | Решение |
|---------|---------|---------|
| savePayment возвращает NULL/0 | неправильные параметры/типы | Проверить вызов: `SELECT savePayment(...)` — id GUID, суммы decimal, service varchar |
| Сумма наличных ≠ сумме платежей | рассинхрон money/payments | Проверить `checkSumMoneyAndPayments`; использовать `Money`/`money_history` для диагностики |
| Нет соединения с MySQL | bind/port/grant | Проверить `bind_address=127.0.0.1`, grant `cashin.*` для `terminal`@localhost |
| Balance не меняется при AddMoney | balance таблица отдельная | `AddMoney` пишет в `money`/`money_history`; `GetBalance()` читает `cashin.balance` (после SetBalance) |

## Тесты

| Симптом | Причина | Решение |
|---------|---------|---------|
| CashInE2E: деньги аккумулируются между прогонами | тест не идемпотентен по абсолютной сумме | Тесты используют дельты; при необходимости `DELETE FROM money_history; DELETE FROM money;` |
| PrinterE2E: MessageBox при Print | PrinterEMPTY показывает содержимое | Использовать пустой документ или Xvfb |
| mono не запускается | нет моно-пакета | `apt install mono-runtime mono-complete` |

## Сеть

| Симптом | Причина | Решение |
|---------|---------|---------|
| Mock не принимает подключения | порт занят | `ss -tlnp | grep -E ":333|:14111"`, убить процесс |
| Логи "Сообщение Message повторное" | повторная отправка без ACK | Проверить, что mock отвечает ACK (тип+номер) |
| Payment mock: "XML parse error" | запрос не в ожидаемом формате | Проверить формат: `<process ...>` / `<check ...>` |

## Файлы

| Симптом | Решение |
|---------|---------|
| Случайные `C:\dabaseDLL_*.txt` в каталоге | Debug-артефакт DAL, можно игнорировать/удалить |
| Файл `MonoDebugTemp\logs\...` (обратный слэш) | Лог приложения, нормальное поведение LoggerLib на Linux |

## Инструменты

```bash
# проверка портов
ss -tlnp | grep -E ":333|:14111|:3306|:8585"
# проверка лога
tail -f '/root/Logic/MonoDebugTemp\logs/2026-09-02.log'
# хеш рабочего MainLogic
sha256sum /root/Logic/MonoDebugTemp/MainLogic.dll   # → 6fd34e55...
# проверка БД
mysql -u <DB_USER> -p'<DB_PASSWORD>' cashin -e "SELECT _name,_value FROM settings"
```