# Production Deployment Checklist — Logic Linux

## 1. RSA Keys

**Текущее состояние:** в `cashin.keys` находятся test-ключи (совпадают с `mock_servers/pub.hex`/`priv.hex`).
Их hex-префикс: `3C5253414B657956616C75653E3C4D6F64756C75733E72526358775078484A2B31444551764B4263...`

**Перед production заменить ключи на ключи провайдера:**

```bash
# 1.1 Проверить текущие ключи (должны быть НЕ равны mock_servers):
mysql -u root cashin -e "SELECT id_key, HEX(LEFT(_key,40)) AS prefix FROM \`keys\`;"
MOCK_PUB=$(cat /root/Logic/mock_servers/pub.hex 2>/dev/null)
MOCK_SEC=$(cat /root/Logic/mock_servers/priv.hex 2>/dev/null)
DB_PUB=$(mysql -u root -N cashin -e "SELECT HEX(_key) FROM \`keys\` WHERE id_key='Public';" 2>/dev/null)
DB_SEC=$(mysql -u root -N cashin -e "SELECT HEX(_key) FROM \`keys\` WHERE id_key='Secret';" 2>/dev/null)
if [ "$DB_PUB" = "$MOCK_PUB" ]; then echo "WARN: Public key = TEST key!"; fi
if [ "$DB_SEC" = "$MOCK_SEC" ]; then echo "WARN: Secret key = TEST key!"; fi

# 1.2 Загрузить production ключи (XML-файлы от провайдера):
#   Public.xml  — <RSAKeyValue><Modulus>...</Modulus><Exponent>AQAB</Exponent></RSAKeyValue>
#   Secret.xml — полный приватный ключ <RSAKeyValue><Modulus>...<D>...</D>...<InverseQ>...</InverseQ></RSAKeyValue>
# Формат: .NET RSA XML (UTF-8, без BOM).
# Пример (1024-бит ключ из оригинальной поставки: Terminal/key/Tucan3.dat, 934 B):
#   mysql -u root cashin -e "UPDATE \`keys\` SET _key=LOAD_FILE('/path/to/Secret.xml') WHERE id_key='Secret';"
#   mysql -u root cashin -e "UPDATE \`keys\` SET _key=LOAD_FILE('/path/to/Public.xml') WHERE id_key='Public';"

# 1.3 Проверить загрузку (размеры):
#   Public:  ~415 B (для 2048-бит RSA) или ~250 B (1024-бит)
#   Secret: ~1679 B (2048-бит с CRT) или ~934 B (1024-бит с CRT)
mysql -u root cashin -e "SELECT id_key, LENGTH(_key) AS bytes FROM \`keys\`;"
```

## 2. Serial Ports & udev

```bash
# 2.1 Создать пользователя logic (если не существует):
useradd -r -s /bin/false -G dialout logic

# 2.2 Убедиться, что logic в группе dialout:
usermod -a -G dialout logic

# 2.3 Создать udev-правило (опционально, для фиксации /dev/ttyS*):
cat > /etc/udev/rules.d/99-logic.rules << 'RULE'
# CashCode CCNet — /dev/ttyS0
KERNEL=="ttyS0", OWNER="logic", GROUP="dialout", MODE="0660"
# Citizen PPU 700 — /dev/ttyS1
KERNEL=="ttyS1", OWNER="logic", GROUP="dialout", MODE="0660"
RULE
udevadm control --reload-rules

# 2.4 Проверить права:
ls -la /dev/ttyS[01]
# должно быть: crw-rw---- 1 logic dialout ...
```

## 3. Production Monitor/Payment адреса

```bash
# 3.1 Установить real production адреса (проверить у провайдера):
mysql -u root cashin -e "
UPDATE settings SET _value='http://PRODUCTION_PAYMENT_URL:PORT' WHERE _name='PaymentUri';
UPDATE settings SET _value='PRODUCTION_MONITOR_HOST' WHERE _name='MonitorURL';
UPDATE settings SET _value='PRODUCTION_MONITOR_PORT' WHERE _name='MonitorPort';
UPDATE settings SET _value='true' WHERE _name='UseProxy';
UPDATE settings SET _value='PROXY_HOST' WHERE _name='ProxyName';
UPDATE settings SET _value='PROXY_PORT' WHERE _name='ProxyPort';
"
#   (Текущие production-значения см. в backup_20260903_082204_hw_clean/cashin_full_clean.sql)
```

## 4. PointId, Password (eKassir auth)

```bash
# 4.1 Установить PointId и Login/Psw (из документации провайдера):
mysql -u root cashin -e "
UPDATE settings SET _value='PROVIDER_POINTID' WHERE _name='PointId';
UPDATE settings SET _value='PROVIDER_LOGIN' WHERE _name='Login';
"
#   Psw (базовый пароль для eKassir-Password header) хранится в keys:
#   mysql -u root cashin -e "UPDATE \`keys\` SET _key='PROVIDER_PSW' WHERE id_key='Psw';"
```

## 5. Финальная проверка

```bash
# 5.1 MainLogic.dll SHA256
sha256sum /root/Logic/MonoDebugTemp/MainLogic.dll
#   Ожидается: 6fd34e55...

# 5.2 Preflight
bash /root/Logic/hw_acceptance/preflight_hardware.sh
#   Ожидается: 0 FAIL

# 5.3 Hardware acceptance
bash /root/Logic/hw_acceptance/capture_evidence.sh
#   Проверить: CashCode init, Printer init, Monitor ACK, GetInfo Success

# 5.4 E2E regression
cd /root/Logic && bash start_logic_linux_test.sh
#   Ожидается: 13/13 PASS
```