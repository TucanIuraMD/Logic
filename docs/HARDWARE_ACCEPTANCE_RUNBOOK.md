# HARDWARE ACCEPTANCE RUNBOOK — Logic Linux (реальный терминал)

Дата подготовки: 2026-09-03. Статус: **NOT READY — ожидает реального терминала с железом и production-доступа.**

> ⚠️ В среде подготовки (LXC-контейнер deepseek-harness) НЕТ физического оборудования,
> НЕТ доступа к production-портам, и в БД установлены TEST RSA ключи.
> Данный runbook выполняется НА РЕАЛЬНОМ ТЕРМИНАЛЕ, где подключены CashCode и Citizen PPU 700.

---

## 0. Pre-flight (обязательно перед стартом)

```bash
cd /root/Logic
# 0.1 SHA256 MainLogic.dll должен быть ровно:
sha256sum MonoDebugTemp/MainLogic.dll
#    6fd34e555367c00e20f87a0ce2c046e1215db07b02ab70c663fff96db6c724c9

# 0.2 Применённые миграции (проверить в БД):
#   save_setting_fix_active.sql  -> save_setting: IF EXISTS/UPDATE + active=0
#   service_schema_fix.sql       -> service: idd_operator, check_timeout, external_id, idd_requisite
#   limitation_create.sql        -> таблица limitation
#   add_limitation_fix.sql       -> addLimitation: порядок параметров = порядку вызова DAL
mysql -u root cashin -e "
  SHOW CREATE PROCEDURE save_setting\G
  SHOW COLUMNS FROM service WHERE Field IN ('idd_operator','check_timeout','external_id','idd_requisite');
  SHOW TABLES LIKE 'limitation';
  SHOW CREATE PROCEDURE addLimitation\G"

# 0.3 mock/test данные изолированы:
mysql -u root cashin -e "SELECT 'service' t,COUNT(*) FROM service
  UNION ALL SELECT 'operators',COUNT(*) FROM operators
  UNION ALL SELECT 'requisites',COUNT(*) FROM requisites
  UNION ALL SELECT 'limitation',COUNT(*) FROM limitation;"
#  ожидание: 0 во всех

# 0.4 production адреса (проверить у провайдера; ниже пример):
mysql -u root cashin -e "SELECT _name,_value FROM settings WHERE _name IN ('PaymentUri','MonitorURL','MonitorPort');"

# 0.5 PRODUCTION RSA keys — установить ДО acceptance:
#   Public и Secret в cashin.keys должны быть ключами провайдера, НЕ совпадать с mock_servers:
mysql -u root cashin -e "SELECT id_key, HEX(LEFT(_key,40)) FROM \`keys\`;"
#   hex-префикс не должен равняться началу mock_servers/pub.hex|priv.hex (3C5253414B657956616C75653E3C4D6F64756C75733E72526358...)
```

## 1. Фактические serial-порты и права

```bash
ls -la /dev/ttyS* /dev/ttyUSB* /dev/ttyACM* /dev/serial/by-id 2>/dev/null
getent group dialout
id logic            # пользователь должен существовать
# ожидаемая конфигурация устройств (DB):
#   id_device=109 CashCode 'CCNet'      name='Cup'     ComPort=/dev/ttyS0 9600 8N1 enc=866
#   id_device=110 Printer  'Citizen PPU 700' name='printer' ComPort=/dev/ttyS1 19200 8N1 enc=866 paper=80
```

## 2. A. REAL CASHCODE (без приёма денег)

Запустить штатный Logic.exe (DeviceCenter инициализирует CashCode при старте):

```bash
cd /root/Logic/MonoDebugTemp && DISPLAY=:99 mono Logic.exe &
# в логе ожидается:
#   "Инизиализация устройства BillAcceptor. Тип: CCNet. Имя устройства - 'Cup'."
#   CashCode статус/инициализация БЕЗ ошибок COM (никаких NullReference/COMPortException)
```

Status/enable/disable через штатный протокол (ручной обмен допустим только по подтверждённой процедуре
оператора; минимальный объём — статус и enable, затем disable). Банкноту НЕ подавать.

## 3. B. REAL PRINTER (Citizen PPU 700)

```bash
# Штатная инициализация при старте Logic.exe:
#   "Инизиализация устройства Printer. Тип: Citizen PPU 700. Имя устройства - 'printer'."
# Безопасный status/query — штатный драйвер (Class402), 19200 8N1.
# Минимальный тест печати — только с разрешения оператора (пустой/сервисный чек).
```

## 4. C. REAL MONITOR

```bash
# production Monitoring Server: MonitorURL (пример <MONITORING_SERVER>:333)
# в логе Logic.exe ожидаются сообщения мониторинга:
#   CurrentDateTime -> "Message was send successfully"
#   VersionLogic     -> "Message was send successfully"
# Без reconnect/protocol errors (смотреть лог на 'Failed', 'reconnect', таймауты)
```

## 5. D. REAL PAYMENT INFO (только Info, БЕЗ payment)

При успешном мониторинге задача GetInfo (периодическая ~60 сек) запросит у production Payment Server только Info.
Ожидания в логе: `Task [GetInfo(1)] ... Type of info is Info` → `... Success.`
Проверить после:
- info.xml создан/обновлён
- settings: VersionProtocol/ClientId/SerialPoint/ClientName обновлены
- operators, services, limitation в БД заполнены
- никаких KeyNotFoundException / Unknown column / SQL exceptions
Банкноту НЕ принимать; денежный Process НЕ выполнять.

## 6. E. GATE

Банкнота не подаётся ни на каком этапе. Приём денег — только отдельным подтверждённым тестом после
полного PASS всех A–D и подтверждения оператора.

## 7. Итоговый отчёт

CashCode: PASS/FAIL
Printer: PASS/FAIL
Monitor: PASS/FAIL
Production Info: PASS/FAIL
DB update: PASS/FAIL
E2E: PASS/FAIL (start_logic_linux_test.sh 13/13 — в т.ч. на реальном терминале)
RSA production keys: YES/NO
Real payment readiness: READY/NOT READY

При полном PASS вывести: "Готово к отдельному подтверждённому тесту реальной банкнотой."
