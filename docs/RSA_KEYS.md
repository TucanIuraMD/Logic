# 12 — RSA and Keys

## Формат ключей

.NET RSA XML (RSACryptoServiceProvider.ToXmlString):
- Public: `<RSAKeyValue><Modulus>...</Modulus><Exponent>AQAB</Exponent></RSAKeyValue>`
- Secret (private): `<RSAKeyValue><Modulus>...</Modulus><Exponent>AQAB</Exponent><P>...</P><Q>...</Q><DP>...</DP><DQ>...</DQ><InverseQ>...</InverseQ><D>...</D></RSAKeyValue>`

## Хранение

MySQL `cashin.keys`:

| id_key | _key (BLOB) | Назначение |
|--------|-------------|-----------|
| `Public` | UTF-8 bytes public RSA XML (~415 B) | верификация / keyId |
| `Secret` | UTF-8 bytes private RSA XML (~1679 B) | подпись платежей |
| `Psw` | пароль канала платежей (12 байт, hex "313233343536" = ASCII "123456") | только для TypeAuthentication=3 |

**Текущее состояние:** TEST ONLY. Public=415 B, Secret=1679 B, 2048-bit RSA ключи, сгенерированы локально.  
Production: заменить на ключи, выданные/зарегистрированные платежным провайдером.

## Использование

- `DAL.GetPublicKey()` / `DAL.GetSecretKey()` — загрузка из БД.
- `MainLogic.Init()`: загружает ключи, устанавливает в `SinkSetting.PublicKey/SecretKey`.
- `SinkSetting.TypeAuthentication=2` (по умолчанию): RSA подпись.
- `Class606`: загружает ключи в `RSACryptoServiceProvider.FromXmlString()`.
- `Class607`: строит запрос: keyId[16] + dataLen[4 BE] + UTF16LE(XML) + RSA-MD5-подпись.

## Проблема (решена)

Изначально в `cashin.keys` были невалидные hex-строки:
- Public: `3C5253414B657956616C75653E3C4D6F64756C75733E766D32544D45534B4559` = `<RSAKeyValue><Modulus>vm2TMESKEY` (обрезано)
- Secret: `<RSAKeyValue><Modulus>SECRETKEY` (обрезано)

→ `RSA.FromXmlString()` → `XmlSyntaxException` → "Ошибка ключей: System error."

**Исправление:** сгенерированы и записаны валидные 2048-битные ключи.

## TEST ONLY компоненты

- `mock_servers/pub.hex` — hex public key (831 байт hex = 415 B XML)
- `mock_servers/priv.hex` — hex private key (3359 байт hex = 1679 B XML) — **никогда не поставлять**
- Запись `cashin.keys` Public/Secret — test keys

## Установка production keys

```bash
mysql -u root cashin -e "
INSERT INTO \`keys\`(id_key, _key)
VALUES('Public', UNHEX('<public-xml-hex>')),
      ('Secret', UNHEX('<secret-xml-hex>'))
ON DUPLICATE KEY UPDATE _key=VALUES(_key);
"
```
Где `<public-xml-hex>` и `<secret-xml-hex>` — hex UTF-8 bytes соответствующих .NET RSA XML.  
Не выводить ключи в лог, не сохранять в файлы с permissive permissions.

## Примечание

- `Psw` в `keys` (hex "313233343536") при декодировании = строковой пароль "123456" — используется только если `TypeAuthentication=3` (иначе 2=RSA).
- `DAL.GetPsw()` → `SinkSetting.Password` (MainLogic.cs:2012).