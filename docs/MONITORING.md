# 10 — Network Monitoring Protocol (TCP :333)

## Полное описание

См. `/root/Logic/_audit/network/MONITOR_PROTOCOL_SPEC.md` — полная спецификация.

## Кратко

- **Transport:** Raw TCP, persistent connection, client-initiated.
- **Framing:** 4-byte LE length prefix + package bytes.
- **Package format:** `byte0(flags)` + `byte1(code)` + `bytes2-5(number LE)` + `body` (if NewVersion flag 0x80).
  - flags: bit0-1 = PackageType (0=Initialization, 1=Message, 2=Command, 3=Pool); bit3=Ack(0x08); bit4=Compressor(0x10); bit7=NewVersion(0x80).
  - Compressor: raw deflate (SharpZipLib) при body > 350 байт.
- **Поток:** Client → Initialization → CurrentDateTimeMessage → unsent messages. Server → ACK (0x88|type + number). Server может посылать Command (type 2).
- **MessageCode:** VersionLogic=1, VersionFlash=2, VersionService=3, ChequeLenta=4, CashCode=5, CurrentDateTime=6, Currency=10, Balance=20, TextMessage=99, MetkaInfo=100, Response=300, ResponseEx=301, и др.
- **TextMessage body:** `[pointState:u8][messageType:u8][UTF8 text]`
- **CurrentDateTimeMessage body:** `Int64 LE DateTime.Ticks`
- **Команды:** CheckUpdate(0), DownLoadSettingsXml(1), UploadZip(2), RestartLogic(3), ExecuteSql(8), SetBlock(9), SetConfig(5), и др.

## Реализация клиента

- `IBP.Emonitor` (MainLogic) — обёртка над `ClientFormatter` (Protocol.dll).
- `ClientFormatter.Start()` → PoolThread + CommandThread.
- `MonitorChannel` — TCP connect.
- `Package` — сериализация/десериализация пакетов.

## Mock сервер

`/root/Logic/mock_servers/monitor_mock.py` — TEST ONLY. Принимает соединения, парсит пакеты, отправляет ACK, логирует.