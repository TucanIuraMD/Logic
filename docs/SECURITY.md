# 16 — Security Findings

Based on `/root/Logic/_audit/deploy/SECURITY_AUDIT.md`.

## Credentials

| находка | где | серьёзность | рекомендация |
|---------|-----|------------|-------------|
| MySQL пароль `<DB_PASSWORD>` hardcoded в `ConnectionStringHelper.cs` (скомпилирован в SqlSettingsProvider.dll) | source | medium | должен совпадать с MySQL user; или пересобрать DLL |
| Тестовая RSA private key (mock_servers/priv.hex) | файл | medium | никогда не поставлять на production |
| Пароль payment канала Psw = "123456" | MySQL keys | low | только если TypeAuthentication=3 |
| MySQL user grants: `cashin_test.*` | MySQL | low | отозвать в production |

## RSA keys

- `priv.hex` — TEST private key file. Не поставлять.
- `keys` Public/Secret — test keys. Заменить на provider keys.
- В исходниках приватных ключей нет (grep: `RSAKeyValue` in sources — none).

## MySQL

- `bind_address=127.0.0.1` — OK (localhost only).
- Нет пользователей с host `%`.
- `terminal@localhost` имеет `ALL ON cashin.*` + `cashin_test.*`; без глобальных привилегий.

## Открытые порты (dev environment)

`12121, 8790/8791, 8787, 3000, 6333/6334, 8080, 3306(localhost), 3081, 53, 25, 3080` — dev services.  
Logic production порты: 3306 (MySQL), 8585 (WebSocket UI, inbound).

## Debug logging

| что | где | утечка |
|-----|-----|--------|
| `Console.WriteLine(SELECT cashin.savePayment(...))` | DAL.cs:2570 | account/total в stdout → journald |
| `Console.WriteLine("Setting.Default.PrinterName="...)` | MainLogic.cs:2156 | device name |
| `Console.WriteLine("WriteToComPort/ReadFromComPort")` | Class402.cs:42-47 | serial traffic (шум) |
| `DAL.WriteLogToFile` → `C:\dabaseDLL_*.txt` | DAL.cs:52 | payment totals/dates |
| `LogRequest=true` (SinkSetting) | если включён | полный XML платежа (account, total) |
| `BanknoteStacked` test hook | MainLogic.cs:2984 | потенциальный backdoor через сценарий |

## File permissions

- Dev tree `/root/Logic/MonoDebugTemp` — 775 group-writable (OK для dev, не для production).
- Production target: `/opt/logic/run` — 750 logic:logic.
- `priv.hex` — 644 (readable by all); только на build host.

## Test/mock remnants

4 test exe, 2 mock servers, test keys, PDB/MDB/.new, _originals, ~25 Windows leftovers — все исключаются installer-ом.

## Immediate safe fixes (deploy pipeline)

1. Исключить test/mock/leftover при упаковке.
2. Установить с user `logic`, 750 perms, dialout group.
3. MySQL localhost-bound, grants только `cashin.*`.
4. `LogRequest=false` в settings.
5. Заменить test RSA keys.
6. Отозвать `cashin_test` grant.