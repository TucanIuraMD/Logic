# 11 — Payment Protocol (HTTP :14111)

## Полное описание

См. `/root/Logic/_audit/network/PAYMENT_PROTOCOL_SPEC.md` — полная спецификация.

## Кратко

- **Transport:** HTTP POST к `SinkSetting.Uri` (= DB `settings.PaymentUri`).
- **Request body:** `base64( keyId[16] + dataLen[4 BE] + UTF16LE(XML) + RSA-MD5-signature )`
  - keyId = первые 16 байт RSA-модуля public key
  - подпись = RSA-SignData(MD5) над XML bytes (data), секретным ключом
  - кодировка тела: ASCII (TypeAuthentication=2) или UTF8 (=3)
- **Request XML (FrameType):**
  - `Process (0)`: `<process ticket="N" number="" id="<guid>" service="T" account="a" total="11.00" value="10.00" source="C" balance="0.00" date="MM/dd/yyyy HH:mm:ss" encashment_number="0" />`
  - `Check (1)`: `<check id="<guid>" service="test_service" account="1234567890" />`
  - `Info (2)`, `GetBalance (3)`, `CheckPayment (4)`, `CheckStatus (5)`, `InfoShort (6)`, `Extended (7)`
- **Response:** plain XML (НЕ подписан):
  ```xml
  <response><result state="2000" account="..." id="..." [extra] /></response>
  ```
  Client парсит элемент `result`, атрибут `state` (int) → StatusPayment; остальные атрибуты → payment.Properties.
- **Content-Encoding:** клиент принимает gzip/deflate; отправляет `Accept-Encoding: deflate`.
- **Ошибки:** HTTP/Web ошибки → `SinkException(Web)`; `SinkExceptionType.Web`.
- **Ключевые state коды:** 2000=Accepted, 1000=Rejected, 3000=Processing, 6000=CantCheckAcc, 7000=AccExists, 8000=AccNotExists, 1099=RejectedBlock.

## Реализация клиента (SinkLib.dll)

- `IBP.SinkSetting` — статические настройки: Uri, Login, Password, PoinyId, TypeAuthentication, PublicKey, SecretKey, Proxy*, TimeOut, LogRequest, UseProxy.
- `IBP.ContainerUri` — список URI.
- `IBP.Formatter` — `Process(IFrame)`.
- `IBP.Frame` — сборка/разборка XML, method_7: state → StatusPayment.
- `IBP.Class606/607/608` — key loading, HTTP POST, XML построение.
- `IBP.Class59` (MainLogic) — поток отправки платежей.

## Логика отправки в MainLogic

`Class59` → `Formatter.Process(frame)` → HTTP POST на PaymentUri → ответ парсится → `payment.State` → `DAL.SetSendedPaymet(payment)` / `SaveMessage`.

## Mock сервер

`/root/Logic/mock_servers/payment_mock.py` — TEST ONLY. Декодирует запрос, логирует XML, отвечает plain XML (Check → state=7000, Process → state=2000, Info → state=10000).

## Проверено

- PaymentProbe (C#): запрос `GET`/формат подписанного сообщения, верификация подписи — PASS.
- Реальный Logic: `<process ticket="6" .../>` отправлен в mock; mock ответил state=2000 — PASS.