# info.xml Audit

## 1. Executive Summary

**info.xml** is a **terminal configuration descriptor** downloaded from the payment server via the HTTP payment protocol (SinkLib). It contains the terminal's identity, account info, available services, scenarios, configuration items, and status definitions. It is the primary mechanism for the server to configure a terminal's behavior.

The file is NOT a static deployment artifact — it is dynamically fetched at runtime by the `TaskGetInfo` periodic task. The `infoshort.xml` variant is a shortened version (without full service/scenario lists) used when the server indicates a short update is sufficient.

## 2. ORIGINAL Evidence

In the pristine ORIGINAL source (`/root/Logic/OriginalLinux/src/`):

- **File references:** `MainLogic/IBP/MainLogic.cs` lines 849, 852, 857
- **File on disk:** `MonoDebugTemp/info.xml` (359702 bytes, 2022-05-23), `infoshort.xml` (738 bytes, 2021-11-19)
- **Runtime code path:** `taskGetInfoHandler` → `Formatter.Process(frame)` (SinkLib) → `Frame.method_1` saves XML to disk → `parseInfoXml` reads and applies config

## 3. Exact Code Paths

### Download path (SinkLib.dll, binary, not in C# source)
```
TaskGetInfo task (MainLogic)
  → taskGetInfoHandler (MainLogic.cs:793)
    → Frame frame = new Frame(ref payment, frameType)  // frameType = Info or InfoShort
    → Formatter.Process(frame)                          // HTTP request to payment server
      → Class607 sends HTTP POST to PaymentUri
      → Server returns XML response
      → Frame.method_1(xmlDocument)                     // SinkLib Frame class, IL
        → if frameType == Info (2):  XmlDocument.Save("info.xml")
        → if frameType == InfoShort (6):  XmlDocument.Save("infoshort.xml")
```

### Parse/apply path (MainLogic.cs, original source)
```
parseInfoXml(frameType) (MainLogic.cs:846)
  → text = "info.xml"  (or "infoshort.xml" if InfoShort)
  → if (!File.Exists(text)) throw Exception1("Файл со спискам услуг не найден (info.xml)")
  → xmlDocument.Load(text)
  → /info/@protocol_version → if != Setting.Default.VersionProtocol → update and save
  → /info/point/@id → guid → if != Setting.Default.ClientId → update and save
  → /info/point/@serial → Setting.Default.SerialPoint
  → /info/client/@serial → Setting.Default.SerialClient
  → /info/client → method_14 → Requisite (legal name, address, taxpayer ID...) → DAL.SaveClientRequisite()
  → /info/point/@blocked, /info/client/@blocked → set blocked flags
  → /info/point/@address, @name → Setting.Default.Address, PointName
  → /info/client/@inn, @name, @deal_number, @deal_date → Setting.Default.Inn, ClientName, DealNumber, DateDealNumber
  → /info/client/@kpp, /info/point/@type, @input_date → settings
  → /info/account → balance, limit, allowed_limit → settings
  → /info/services → ServiceContainer → DAL.SaveServices(), ParserMain parsing
  → /info/scenarios, /info/steps, /info/print_blocks, /info/config, /info/users,
    /info/languages, /info/bar_codes, /info/help, /info/statuses, /info/modules,
    /info/procedures, /info/tasks, /info/intricate
  → If full Info: method_21(true) — marks task as "full info received"
  → If InfoShort: method_21(false) — marks as "short info received"
```

## 4. XML Structure (from actual `/root/Logic/MonoDebugTemp/info.xml`)

```
<info protocol_version="2.5.1" time="20220523120052">
  <client id="..." serial="1" name="MMPS" blocked="False" inn="..." kpp="4" deal_number="1" deal_date="25/04/2008">
    <requisites legal_name="..." post_address="..." taxpayer_id_number="..." rr_code="..." contact_phone_number="..." />
  </client>
  <account id="5" serial="5" name="..." allowed_limit="..." balance="..." limit="..." />
  <point id="..." serial="3" blocked="False" address="..." name="..." type="3" input_date="..." />
  <states>
    <state id="1000" meaning="Decline" />
    ... (1001-1021: various rejection codes with descriptions)
  </states>
  <services> ... </services>        <!-- available services with aliases, commissions, limits -->
  <scenarios> ... </scenarios>      <!-- scenario definitions with steps -->
  <steps> ... </steps>              <!-- step definitions -->
  <print_blocks> ... </print_blocks> <!-- receipt templates -->
  <config> ... </config>            <!-- configuration items -->
  <users> ... </users>              <!-- operator users -->
  <languages> ... </languages>      <!-- UI languages -->
  <bar_codes> ... </bar_codes>      <!-- barcode definitions -->
  <help> ... </help>                <!-- help items -->
  <statuses> ... </statuses>        <!-- status labels -->
  <modules> ... </modules>          <!-- module definitions -->
  <procedures> ... </procedures>    <!-- stored procedures -->
  <tasks> ... </tasks>              <!-- scheduler tasks -->
  <intricate> ... </intricate>      <!-- intricate scenarios -->
</info>
```

**infoshort.xml** (738 bytes) contains only the header (`<info>`, `<client>`, `<account>`, `<point>`) — no full service/scenario lists.

## 5. Runtime Behaviour

- **When called:** The `TaskGetInfo` periodic task (every ~60 seconds by default, configurable via task parameters in DB).
- **Who calls:** `Class52` (TaskScheduller) in MainLogic — runs in its own thread.
- **What triggers:** Timer-based, every task cycle. Also triggered at startup? Let me check: `Thread get info started` appears in the log at startup, suggesting the task is queued early.
- **Network dependency:** Requires payment server connection (HTTP POST to PaymentUri). If the server is unreachable, the task catches the exception, reschedules with a random delay, and retries.
- **File is temporary:** info.xml is overwritten each time the task runs (the server returns the current state). It is not a permanent configuration file — it's a cache of the current server response.

## 6. Creation / Modification

- **Created by:** `Frame.method_1` in SinkLib.dll (binary) — `XmlDocument.Save("info.xml")`.
- **Modification:** The file is **overwritten** each time the GetInfo task runs. No other code modifies it.
- **Deletion:** The file is NOT deleted by the application. If the task fails, the old file remains on disk and may be stale.
- **The app does NOT create info.xml itself** — it always comes from the server. The `parseInfoXml` method throws `Exception1` if the file doesn't exist.

## 7. Network / Update Relationship

- **Protocol:** The info.xml is fetched using the same HTTP payment protocol (SinkLib) as payment processing, via `FrameType.Info` or `FrameType.InfoShort`.
- **Server endpoint:** The same `PaymentUri` (default `http://<MONITORING_SERVER>:14111`) — the server distinguishes frame type.
- **Update relationship:** The `DownLoadSettingsXml` monitoring command is a separate mechanism for updating scenarios. info.xml is the "current state" descriptor, not an update package. However, the scenario/service/step data in info.xml IS used to update the local DB (via `ParserMain` and `DAL.SaveServices()`, `DAL.SaveScenario()`, etc.).
- **Is an update manifest?** PARTIALLY — it carries the current terminal configuration, but the `UploadZip` / `DownLoadSettingsXml` commands are the actual update delivery mechanism.

## 8. Missing / Invalid File Behaviour

- **Missing file:** `parseInfoXml` throws `Exception1("Файл со спискам услуг не найден (info.xml)")`. This is caught by the caller (`taskGetInfoHandler`), which reschedules the task with a random delay. The terminal will keep retrying until the file is successfully downloaded.
- **Invalid XML:** `XmlException` is caught, an error message is sent via `emonitor.SaveMessage()`, and `Exception1` is re-thrown (caught by the task handler, same retry).
- **Stale file:** If the task fails after the old file exists, the old file is used. The `protocol_version` check in `parseInfoXml` updates settings if the version changed.
- **No fallback/default:** The file is required. Without it, the services/steps/scenarios from the server are not applied, and the terminal operates with whatever was previously in the DB.

## 9. ORIGINAL vs MonoDebugTemp

**No differences found in the code path.** The `parseInfoXml` method in ORIGINAL source (`MainLogic.cs:846-1100`) is identical to the `_audit` version. The SinkLib DLL is the same (the file in MonoDebugTemp is the original Windows binary, not rebuilt). The `info.xml` and `infoshort.xml` files in MonoDebugTemp are the same files that were in the original deployment (Terminal.zip).

The only difference: in the current MonoDebugTemp environment, the GetInfo task fails with "Ошибка ключей: System error." because the payment mock returns a generic response, not a valid info.xml. The file is not updated because the Formatter.Process() fails before Frame.method_1 saves the file.

## 10. Production Importance

**HIGH.** info.xml is the mechanism by which the payment server delivers:
- Terminal identity (ClientId, point serial)
- Blocked/active status
- Account balance and limits
- Available services (with aliases, commissions, limits)
- Scenario definitions (UI flow)
- Step definitions
- Receipt templates
- Configuration items
- Operator users
- UI languages
- Status definitions

Without a successful GetInfo task, the terminal cannot receive updated service/scenario configurations from the server. The file is required for normal operation.

## 11. Unknowns

- **Exact XML structure of `<services>`, `<scenarios>`, `<steps>`** — these are complex and contain the full terminal configuration. Only the parser (`ParserMain`) can fully interpret them.
- **How `infoshort.xml` is triggered** — the `taskGetInfoHandler` calls `tgi.method_22()` to determine whether to use Info or InfoShort. The exact criteria are in `TaskGetInfo.method_22()`, which is in MainLogic.cs (the task class).
- **What happens if the server returns a different XML structure** — the parser may throw exceptions or silently ignore unknown elements.
- **The `parserMain` processing of the services/scenarios sections** — this is a complex XML parsing pipeline that imports the entire terminal configuration into the DB.

## 12. Evidence — Files/Classes/Methods/Lines

| File | Class | Method | Line | Role |
|------|-------|--------|------|------|
| `MainLogic/IBP/MainLogic.cs` | `MainLogic` | `taskGetInfoHandler` | 793 | Entry point for GetInfo task |
| `MainLogic/IBP/MainLogic.cs` | `MainLogic` | `parseInfoXml` | 846 | Reads and applies info.xml/infoshort.xml |
| `MainLogic/IBP/MainLogic.cs` | `MainLogic` | `method_14` | 1192 | Reads Requisite from XML node |
| `MainLogic/IBP/MainLogic.cs` | `MainLogic` | `method_15` | 1208 | XPath helper: `SelectSingleNode(xpath).InnerText` |
| `SinkLib.dll` (binary) | `Frame` | `method_1` | IL | Saves XmlDocument to info.xml or infoshort.xml |
| `SinkLib.dll` (binary) | `Class607` | `imethod_4` / `method_2` | IL | HTTP POST, receives response, passes to Frame |
| `SinkLib.dll` (binary) | `Formatter` | `Process` | IL | Orchestrates request/response/Frame |
| `MonoDebugTemp/info.xml` | - | - | - | Actual file (359702 B, 2022-05-23) |
| `MonoDebugTemp/infoshort.xml` | - | - | - | Short version (738 B, 2021-11-19) |

## Conclusion

**info.xml отвечает за получение и применение конфигурации терминала с сервера платежей: identity, блокировки, баланс, услуги, сценарии, настройки, пользователи, языки.**

Файл динамически загружается от платежного сервера по HTTP протоколу (SinkLib, FrameType.Info) и используется для обновления локального состояния терминала (settings, БД). Это не статический deployment artifact, а runtime-кэш ответа сервера. `infoshort.xml` — сокращённая версия без полных списков услуг/сценариев.

ORIGINAL и MonoDebugTemp код path идентичен — изменения при Linux-порте не вносились. Единственная разница: в тестовом окружении (с mock-серверами) GetInfo задача не может загрузить info.xml из-за ошибки ключей в payment mock, и файл не обновляется.