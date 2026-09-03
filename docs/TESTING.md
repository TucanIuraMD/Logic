# 13 — Testing and E2E

## Тестовые executables

| Executable | Источник | Checks | Назначение |
|-----------|----------|--------|-----------|
| `DbIntegrationTest.exe` | `_audit/DbIntegrationTest.cs` | 10 | DAL: settings, balance, SavePayment, GetNotSentPayments, AddMoney, GetTicketNumberPayment, GetTasks, GetListServices, MMPS_updateMainMenu |
| `DeviceLayerTest.exe` | `_audit/DeviceLayerTest.cs` | 13 | GetDevices, GetDeviceByName (CCNet/Citizen), GetDeviceProperties (Cup/printer), no duplicates |
| `CashInE2ETest.exe` | `/tmp/CashInE2ETest.cs` | 16 | Cash-in flow через DAL: AddMoney→money_history, SavePayment→TicketNumber, multi-denominations, invalid denom (0), ledger consistency, balance roundtrip, payment status, cleanup |
| `PrinterE2ETest.exe` | `/tmp/PrinterE2ETest.cs` | 13 | PrinterEMPTY: Open, GetState (LentaStatus=Ok), Print (пустой документ), Info, Close, re-open |

**Все — TEST ONLY, не входят в production install.**

## Mock servers (TEST ONLY)

- `mock_servers/monitor_mock.py` — TCP :333, парсит пакеты, отвечает ACK, логирует.
- `mock_servers/payment_mock.py` — HTTP :14111, декодирует signed request, логирует XML, отвечает plain XML (2000/7000/10000).

## E2E скрипт

`/root/Logic/start_logic_linux_test.sh` — полный прогон:

| Фаза | Что делает | Проверка |
|------|-----------|----------|
| 0 | Environment (MySQL, Mono, Xvfb) | PASS |
| 1 | DbIntegrationTest | 10/10 |
| 2 | DeviceLayerTest | 13/13 |
| 3 | start monitor+payment mocks | PASS |
| 4 | Network test (TCP ACK + HTTP respond) | PASS |
| 5 | CashInE2ETest | 16/16 |
| 6 | PrinterE2ETest (с Xvfb) | 13/13 |
| 7 | Start Logic.exe, ждать "Logis is STARTED" | PASS |
| 8 | Проверка логов мониторинга (client connected) | PASS |
| 9 | Итоговый отчёт | 13 checks |

Запуск: `bash /root/Logic/start_logic_linux_test.sh`  
Логи: `/tmp/logic_test_logs/<timestamp>/`

## Итоговый статус

```
DB             10/10 PASS
DEVICE         13/13 PASS
NETWORK        PASS
PAYMENT        PASS
MONITOR        PASS
RSA            PASS (test keys)
CASH-IN        16/16 PASS
PRINTER        13/13 PASS
STARTUP        PASS
E2E            13/13 PASS
```

## Как воспроизвести

1. Убедиться: MySQL запущен, `cashin` БД настроена (settings: MonitorURL=localhost, MonitorPort=333, PaymentUri=http://localhost:14111).
2. `bash /root/Logic/start_logic_linux_test.sh`
3. Дождаться финального отчёта.

## Ограничения

- Mock-тесты НЕ являются production acceptance.
- Hardware acceptance (реальный CashCode/Citizen) — отдельный этап.
- E2E предполагает наличие проверенных DLL в MonoDebugTemp.