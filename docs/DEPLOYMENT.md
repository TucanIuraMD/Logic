# 15 — Production Deployment

## Файлы deployment (TEST-независимые)

| Файл | Назначение |
|------|-----------|
| `/root/Logic/_audit/deploy/install_logic_linux.sh` | production installer (Ubuntu 24.04) |
| `/root/Logic/_audit/deploy/logic-linux.service` | systemd unit |
| `/root/Logic/_audit/deploy/README.md` | полный deployment guide |
| `/root/Logic/_audit/deploy/TEST_MOCK_AUDIT.md` | классификация test/mock компонентов |
| `/root/Logic/_audit/deploy/CONFIG_AUDIT.md` | аудит конфигурации |
| `/root/Logic/_audit/deploy/SECURITY_AUDIT.md` | security findings |

## Целевая среда

- Ubuntu 24.04 LTS (Noble), x86_64
- Mono 6.8.0.105 (`mono-runtime mono-complete libmono-system-windows-forms4.0-cil libmono-system-io-ports4.0-cil libgdiplus`)
- MySQL 8.0 (bind 127.0.0.1)
- user `logic` (group `dialout`), app dir `/opt/logic/run`

## Установка

```bash
sudo ./install_logic_linux.sh --src /path/to/runtime-set --db-setup --db-pass '<secret>'
sudo mysql cashin < db/script.sql   # fresh install
# configure settings + keys в MySQL (см. 06, 12, 14)
systemctl enable --now logic-linux
journalctl -u logic-linux -f
```

## systemd unit (logic-linux.service)

- `User=logic`, `Group=logic`, `SupplementaryGroups=dialout`
- `WorkingDirectory=/opt/logic/run`, `ExecStart=/usr/bin/mono /opt/logic/run/Logic.exe`
- `After=mysql.service network-online.target`, `Requires=mysql.service`
- `Restart=on-failure`, `RestartSec=10`
- `KillSignal=SIGTERM`, `TimeoutStopSec=30`, `KillMode=process` (graceful shutdown)
- `NoNewPrivileges=true`, `ProtectSystem=full`, `PrivateTmp=true`, пустой capability set
- DeviceAllow: /dev/ttyS0, /dev/ttyS1, /dev/ttyUSB*, /dev/ttyACM*
- `Environment=DISPLAY=:0` (kiosk UI)
- **Mock servers НЕ подключены к unit.**

## udev rules

`/etc/udev/rules.d/99-logic-terminal.rules`:
```
KERNEL=="ttyS*", MODE="0660", GROUP="dialout"
KERNEL=="ttyUSB*", MODE="0660", GROUP="dialout"
KERNEL=="ttyACM*", MODE="0660", GROUP="dialout"
KERNEL=="lp*", MODE="0660", GROUP="lp"
```

## Порты

| direction | port | protocol | purpose |
|-----------|------|----------|---------|
| out | 333 | TCP | monitoring |
| out | 14111 | TCP/HTTP | payment |
| out | 14444 | HTTP proxy | optional |
| in (opt) | 8585 | WS | kiosk UI |
| in | 3306 | TCP | MySQL (127.0.0.1) |

## Production configuration шаги

1. Load schema.
2. Set settings: MonitorURL, MonitorPort, PaymentUri, PointId, ClientId.
3. Set device COM ports (real hardware wiring).
4. Install production RSA keys.
5. Убедиться, что MySQL user password совпадает с вшитым connection string (или пересобрать).
6. Start service.

## Runtime DLL set для поставки

Logic.exe + MainLogic.dll + CommonLib.dll + DataBase.dll + LogicForm.dll + DeviceCenter.dll + Devices.dll + CashCodeWrapper.dll + Printer.dll + LoggerLib.dll + SqlSettingsProvider.dll + Protocol.dll + SinkLib.dll + SettingParser.dll + AdminForm.dll + CardClient.dll + CardReader.dll + SmartCardReader.dll + PinPad.dll + CoinValidator.dll + Dispenser.dll + Hopper.dll + BarCodeReaderWrapper.dll + ExitSystem.dll + ExternalLib.dll + IUpdate.dll + Fleck.dll + сторонние (BouncyCastle, MySql.Data, Newtonsoft.Json, SharpZipLib, PowerCollections, System.*, netstandard, Google.Protobuf, K4os.*, Zstandard.Net).

**Исключить:** тестовые exe, mocks, PDB/MDB/.new, _originals, Windows leftovers.