# 22. Redfish 通用知識

Redfish 是以 HTTPS、JSON 與標準化 resource model 為基礎的管理介面。Client 可透過固定 URI 查詢系統、機箱、BMC、帳號、事件與韌體，也能送出 power、reset、update 等管理要求。

OpenBMC 通常由 bmcweb 提供 Redfish API。bmcweb 接收 HTTP request，查詢或呼叫 D-Bus services，再依 Redfish schema 組成 response。

## 22.1 Redfish 是什麼

Redfish 將平台管理功能表達成 resources。每個 resource 都有 URI、JSON properties、links 與可執行的 actions。

例如：

```text
/redfish/v1/Systems/system
```

可能描述：

- 系統型號與序號。
- Power state。
- BIOS version。
- Processor 與 Memory links。
- Boot override。
- Reset action。
- Health status。

Client 使用 HTTPS 存取：

```bash
curl --cacert <ca.pem> \
     -H 'Accept: application/json' \
     -H 'X-Auth-Token: <token>' \
     https://<bmc>/redfish/v1/Systems/system
```

正式環境應驗證 BMC TLS certificate。`-k` 只適合隔離的開發環境，因為它會略過 certificate validation。

## 22.2 Redfish 的組成

Redfish 可分成四個部分：

```text
Protocol
HTTPS、HTTP methods、headers、status codes、authentication
        ↓
Data Model
Systems、Chassis、Managers、Sensors、Accounts、Software 等 resources
        ↓
Schemas
定義 properties、types、links、actions 與版本
        ↓
Registries
定義 errors、events 與其他 message 的 identifiers 和文字
```

### 22.2.1 Protocol

Protocol 規範 client 如何送出 request、service 如何回傳 response，以及 authentication、ETag、error response、asynchronous task 等行為。

### 22.2.2 Data Model

Data model 描述受管理系統的資源與關係。例如：

- `ComputerSystem`：Host system。
- `Chassis`：實體機箱或機構範圍。
- `Manager`：BMC 管理控制器。
- `PowerSupply`：電源供應器。
- `Sensor`：感測資料。
- `SoftwareInventory`：韌體版本。

### 22.2.3 Schema

Schema 定義一種 resource 的合法結構。Resource response 中的 `@odata.type` 表示它遵循的 schema type 與版本。

### 22.2.4 Message Registry

Message registry 定義標準 message identifiers，例如 property 格式錯誤、resource missing、authentication failure 或 update result。Service 以 registry ID 表示 machine-readable error / event，client 再依 registry 取得 message template、severity 與 resolution。

## 22.3 Redfish 與 OpenBMC

OpenBMC 常見資料路徑：

```text
Redfish Client
        ↓ HTTPS
bmcweb route / handler
        ↓ D-Bus query or method call
OpenBMC service
        ↓
Kernel driver / hardware
```

讀取 sensor 的情況：

```text
GET /redfish/v1/Chassis/<id>/Sensors/<sensor>
        ↓
bmcweb 查詢 D-Bus sensor object
        ↓
xyz.openbmc_project.Sensor.Value
        ↓
Redfish Sensor JSON
```

送出 power reset 的情況：

```text
POST ComputerSystem.Reset action
        ↓
bmcweb 驗證 privilege 與 payload
        ↓
呼叫 Host / Chassis state service
        ↓
Power-control flow
        ↓
回傳 success、error 或 Task
```

Redfish resource 通常會整合多個 D-Bus objects。Resource URI 與 D-Bus object path 的模型不同，兩者之間需要 bmcweb mapping、associations 與 policy。

## 22.4 Service Root

Redfish entry point：

```text
/redfish/v1/
```

Service Root 提供主要 collections 與 services 的 links，例如：

- `Systems`
- `Chassis`
- `Managers`
- `AccountService`
- `SessionService`
- `UpdateService`
- `EventService`
- `TaskService` / `Tasks`
- `TelemetryService`
- `Registries`
- `JsonSchemas`

檢查：

```bash
curl --cacert <ca.pem> \
     -H 'X-Auth-Token: <token>' \
     https://<bmc>/redfish/v1/
```

Client 應沿 response links 走訪 resources，避免假設所有實作都使用相同的 member ID。

## 22.5 Resource、Collection 與 Member

Collection 是同類 resources 的清單。

```text
/redfish/v1/Systems
        ↓
ComputerSystemCollection
        ↓ Members
/redfish/v1/Systems/system
```

典型 collection response：

```json
{
  "@odata.id": "/redfish/v1/Systems",
  "@odata.type": "#ComputerSystemCollection.ComputerSystemCollection",
  "Members": [
    {
      "@odata.id": "/redfish/v1/Systems/system"
    }
  ],
  "Members@odata.count": 1,
  "Name": "Computer System Collection"
}
```

Member link 的存在代表 client 可以繼續 GET 該 URI；collection 本身通常只提供清單，不包含每個 member 的完整內容。

## 22.6 常見頂層 Resources

### 22.6.1 Systems

```text
/redfish/v1/Systems
```

描述 Host system，常見內容：

- PowerState。
- Boot settings。
- BIOS version。
- Processor / Memory links。
- Storage / Ethernet interfaces。
- Host watchdog 與 actions，依實作。

### 22.6.2 Chassis

```text
/redfish/v1/Chassis
```

描述實體機構與其中的硬體，常見內容：

- Chassis identity。
- Power supplies。
- Fans。
- Sensors。
- PCIe slots / devices。
- Physical security。
- Thermal / power subsystem links。

### 22.6.3 Managers

```text
/redfish/v1/Managers
```

描述 BMC 本身，常見內容：

- BMC firmware version。
- Manager state。
- Date / time。
- NetworkProtocol。
- EthernetInterfaces。
- Serial interfaces。
- Reset action。
- Log services。

### 22.6.4 三者的關係

```text
ComputerSystem
Host 的邏輯系統

Chassis
實體機箱與硬體容器

Manager
管理 Host 和 Chassis 的 BMC
```

Links 與 associations 將它們連起來。System serial、chassis serial 與 BMC firmware version 應來自各自正確的資料來源。

## 22.7 OData Properties

Redfish 使用部分 OData conventions。

### 22.7.1 `@odata.id`

此 resource 的 canonical URI：

```json
"@odata.id": "/redfish/v1/Systems/system"
```

### 22.7.2 `@odata.type`

表示 schema type 與版本：

```json
"@odata.type": "#ComputerSystem.v1_25_0.ComputerSystem"
```

版本會隨 service 採用的 schema release 而不同。Client 應依 schema 相容規則處理可選的新 properties。

### 22.7.3 `@odata.etag`

ETag 用來識別 resource version，可配合 `If-Match` 或 `If-None-Match` 避免競爭更新或重複傳輸。

```http
If-Match: "<etag>"
```

修改資源前，client 可先 GET 保存 ETag，再以 `If-Match` 送出 PATCH。Resource 已被其他 client 更新時，service 可拒絕舊 ETag 的 request。

## 22.8 HTTP Methods

### 22.8.1 GET

讀取 resource 或 collection，不應改變受管理資源狀態。

```bash
curl --cacert <ca.pem> \
     -H 'X-Auth-Token: <token>' \
     https://<bmc>/redfish/v1/Managers/<id>
```

### 22.8.2 PATCH

修改 resource 的部分 writable properties。

```bash
curl --cacert <ca.pem> \
     -X PATCH \
     -H 'Content-Type: application/json' \
     -H 'X-Auth-Token: <token>' \
     -d '{"AssetTag":"Rack-A-Unit-10"}' \
     https://<bmc>/redfish/v1/Chassis/<id>
```

Service 應驗證 property 是否存在、是否 writable、type 與 value 是否合法，以及目前帳號是否具備 privilege。

### 22.8.3 POST

可用於：

- 建立 resource，例如 session 或 event subscription。
- 呼叫 action，例如 reset 或 firmware update。

### 22.8.4 DELETE

移除可刪除的 resource，例如 session、event subscription 或自訂 account，需依 schema 與 privilege 決定。

## 22.9 Actions

Action 是對 resource 執行命令。Actions 通常位於 response 的 `Actions` property。

ComputerSystem reset 範例：

```json
"Actions": {
  "#ComputerSystem.Reset": {
    "target": "/redfish/v1/Systems/system/Actions/ComputerSystem.Reset",
    "@Redfish.ActionInfo": "/redfish/v1/Systems/system/ResetActionInfo"
  }
}
```

呼叫：

```bash
curl --cacert <ca.pem> \
     -X POST \
     -H 'Content-Type: application/json' \
     -H 'X-Auth-Token: <token>' \
     -d '{"ResetType":"GracefulRestart"}' \
     https://<bmc>/redfish/v1/Systems/system/Actions/ComputerSystem.Reset
```

ActionInfo 可描述支援的 parameters 與 allowable values。Client 應讀取 service 提供的 ActionInfo 或 `@Redfish.AllowableValues`，避免寫死所有機型都支援相同選項。

## 22.10 HTTP Status Code

| Status | 常見意義 |
|---:|---|
| 200 | Request 成功並回傳完整 response |
| 201 | Resource 建立成功 |
| 202 | 已接受並開始非同步處理，通常提供 Task link |
| 204 | 成功且無 response body |
| 400 | Payload、property 或參數錯誤 |
| 401 | 尚未成功 authentication |
| 403 | 已識別身分，但 privilege 不足 |
| 404 | Resource 或 URI 不存在 |
| 405 | 該 resource 不支援此 HTTP method |
| 409 | Resource state conflict |
| 412 | ETag / precondition 失敗 |
| 415 | Content-Type 不支援 |
| 429 | Request rate 超過服務限制 |
| 500 | Service 內部錯誤 |
| 503 | Service 暫時不可用 |

Client 不應只依 status code 顯示模糊錯誤；Redfish error body 通常包含更精確的 message IDs 與 resolutions。

## 22.11 Error Response 與 Message Registry

典型 error response：

```json
{
  "error": {
    "code": "Base.1.0.GeneralError",
    "message": "A general error has occurred.",
    "@Message.ExtendedInfo": [
      {
        "MessageId": "Base.1.0.PropertyValueNotInList",
        "Message": "The value supplied for ResetType is not in the list of acceptable values.",
        "MessageArgs": ["InvalidValue", "ResetType"],
        "Severity": "Warning",
        "Resolution": "Choose a value from the allowable values and resubmit the request."
      }
    ]
  }
}
```

實際 message text、severity 與 resolution 依 registry version 而定。Client 應優先解析：

- HTTP status。
- `MessageId`。
- `MessageArgs`。
- Related properties，若提供。
- Resolution。

`MessageId` 常具有：

```text
RegistryName.Major.Minor.MessageKey
```

Service 可透過 `/redfish/v1/Registries` 提供 registry resources 與下載連結。

## 22.12 Authentication

### 22.12.1 Session Token

建立 session：

```bash
curl --cacert <ca.pem> \
     -i \
     -X POST \
     -H 'Content-Type: application/json' \
     -d '{"UserName":"<user>","Password":"<password>"}' \
     https://<bmc>/redfish/v1/SessionService/Sessions
```

成功後 response headers 通常提供：

- `X-Auth-Token`
- `Location`

後續 request 帶入：

```http
X-Auth-Token: <token>
```

結束時 DELETE `Location` 指向的 session URI。

### 22.12.2 Basic Authentication

Basic Authentication 將 credentials 放在每次 request。若產品允許使用，必須搭配有效 TLS；自動化工具仍較適合建立 session token，減少 credentials 重複傳送與散落在 logs / process arguments 的風險。

### 22.12.3 Mutual TLS

部分服務支援 client certificate authentication。其行為取決於 certificate mapping、trust store、AccountService 與產品設定。

### 22.12.4 Authentication 與 Authorization

Authentication 確認「你是誰」；Authorization 根據 role 與 privileges 判斷「你可以做什麼」。登入成功不代表可以執行 firmware update、修改帳號或控制 Host power。

## 22.13 AccountService 與 Roles

```text
/redfish/v1/AccountService
```

可描述：

- Accounts collection。
- Roles。
- Password policy。
- Account lockout。
- LDAP / Active Directory，依實作。
- Multi-factor authentication，依實作。

常見角色概念：

- Administrator。
- Operator。
- ReadOnly。
- OEM / custom role，依平台支援。

實際授權由 privilege registry 與 route requirement 決定。新增 Redfish route 時，需要同步指定正確 privileges。

帳號處理應驗證：

- Create / delete account。
- Password change。
- Disable / enable account。
- Lockout 與 recovery。
- Session invalidation。
- Role modification。
- External directory 暫時不可用時的行為。

## 22.14 SessionService

```text
/redfish/v1/SessionService
```

SessionService 管理 token-based sessions。需確認：

- Session timeout。
- Maximum sessions。
- 登出後 token 立即失效。
- Account disable / delete 後相關 sessions 的處理。
- BMC reboot 後 session 是否保留，通常依安全設計失效。
- Token 不出現在一般 journal 或 debug log。

## 22.15 UpdateService

```text
/redfish/v1/UpdateService
```

UpdateService 提供 firmware inventory 與更新入口。實際更新通常由 bmcweb 將 image 或 URI 資訊交給 OpenBMC software / update services。

```text
Client Upload / SimpleUpdate
        ↓
bmcweb 驗證 request 與權限
        ↓
Image staging
        ↓
Software update service 驗證 image
        ↓
建立 SoftwareInventory / Activation
        ↓
Write / Verify / Activate
        ↓
Task、SoftwareInventory 與 Event 更新
```

### 22.15.1 Firmware Inventory

常見 resources 位於 UpdateService 下的 FirmwareInventory 或 SoftwareInventory links。它們可描述：

- Version。
- Purpose / component。
- Updateable。
- Status。
- Related item。

### 22.15.2 Push Update

Client 上傳 image。需考量：

- Content-Type。
- Upload size limit。
- Staging storage。
- Request timeout。
- Image signature 與 machine compatibility。
- BMC reboot / network interruption。

### 22.15.3 SimpleUpdate

Client 提供可由 BMC存取的 image URI。若平台支援，需限制 protocols、certificate validation、credentials handling 與 network access policy。

### 22.15.4 Task

長時間 update 可回傳 `202 Accepted` 與 Task / TaskMonitor URI。Client 應追蹤 Task state，而不是保持單一 HTTP connection 等待整個更新完成。

## 22.16 TaskService

Task 表示非同步處理狀態，例如 firmware update、diagnostic collection 或長時間 action。

常見 properties：

- `TaskState`
- `TaskStatus`
- `PercentComplete`
- `StartTime`
- `EndTime`
- `Messages`
- `Payload`，依 schema 與實作

典型狀態：

```text
New / Starting
        ↓
Running
        ↓
Completed
```

也可能進入 Exception、Killed、Suspended 等狀態，依 schema 與服務支援。

Client 應使用 response 提供的 TaskMonitor / Location，不應自行猜測 Task URI。

## 22.17 EventService

```text
/redfish/v1/EventService
```

EventService 管理 subscriptions 與事件傳遞。Client 可設定 destination、event format、filters 與 retry policy，實際支援內容依 service capabilities。

```text
OpenBMC sensor / logging / state change
        ↓
Redfish event generation
        ↓
Subscription filter
        ↓
HTTP POST 到 subscriber destination
```

### 22.17.1 Event 與 Log Entry

Event 是主動通知；Log Entry 是保存在 BMC 上供後續查詢的紀錄。一次故障可能同時建立 log entry 並送出 event，但它們有不同生命週期。

### 22.17.2 Subscription

測試：

- 建立 subscription。
- 觸發符合 filter 的 event。
- 驗證 destination 收到 payload。
- Destination 暫時失敗時驗證 retry。
- 刪除 subscription。
- BMC reboot 後是否保留，依產品規格。

Subscriber endpoint 應使用 TLS，並依產品要求驗證其 certificate。

## 22.18 SSE 與 Metric Reports

部分實作可透過 Server-Sent Events（SSE）提供長連線事件串流。TelemetryService 也可建立 metric report definitions，定期或依 trigger 產生 metric reports。

SSE、Redfish event subscription 與傳統 log query 各有用途：

| 方式 | 用途 |
|---|---|
| Event subscription | 將事件 POST 到指定 destination |
| SSE | Client 維持串流連線接收事件 |
| LogService | Client主動查詢已保存紀錄 |
| Telemetry report | 定期或條件式輸出多個 metrics |

需測試斷線重連、event loss、buffer limit、authentication 與大量事件時的 backpressure。

## 22.19 Sensors、Power 與 Thermal

現代 Redfish schema 可透過 Sensor resources、PowerSubsystem、ThermalSubsystem、EnvironmentMetrics 等方式呈現資料；舊版 service 也可能提供經典 `Power` 與 `Thermal` resources。Client 應沿 links 與 schema capability 取得資料。

OpenBMC mapping 常包含：

```text
D-Bus sensor Value
        → Reading / ReadingType / Units

Threshold interfaces
        → Threshold properties / Status

Availability / Functional
        → Status.State / Status.Health

Inventory association
        → RelatedItem / Chassis placement
```

需要確認：

- Scale 與 unit。
- NaN / unavailable handling。
- Threshold state。
- Inventory association。
- Present=false 時的 sensor behavior。
- Resource URI 是否穩定。

## 22.20 Inventory 與 Status

Asset properties 常包含：

- Manufacturer。
- Model。
- PartNumber。
- SerialNumber。
- AssetTag。

`Status` 常包含 `State` 與 `Health`。它們通常彙整 inventory presence、functional state、sensor faults 與 service policy。

程式不應把 `Health=OK` 當成元件一定存在，也不應用單一 threshold直接代表整個 resource health。資料來源與聚合規則需在平台文件中定義。

## 22.21 Schema Version 與相容性

Redfish schemas 持續增加 properties、resources 與新版 schema types。Service 實作通常只支援其中一組明確版本。

相容策略：

- Client 忽略不認識的可選 properties。
- Client 不依賴 JSON property order。
- Service 保持已發布 URI 穩定。
- 新增 optional property 時維持既有 properties 的語意。
- 修改 enum、type 或 writable behavior 前確認 schema 規範。
- Client 依 `@odata.type` 與 Service Root `RedfishVersion` 判斷能力。
- 重要自動化需求可使用 Redfish interoperability profile 驗證。

### 22.21.1 JsonSchemas

```text
/redfish/v1/JsonSchemas
```

Service 可公開 schema resources，實際內容與 URI 依實作。開發時仍應以 DMTF 發布的 schema bundle 與 data model specification 作為主要依據。

### 22.21.2 Validator

Redfish Service Validator 可檢查 URI、schema、required properties、types、links 與 response behavior。通過 validator 是基本要求，平台仍需額外驗證產品語意、權限、update、events 與錯誤流程。

## 22.22 OEM Extension

標準 schema 無法呈現特定平台功能時，可使用 OEM extension。

```json
{
  "Oem": {
    "ExampleVendor": {
      "@odata.type": "#ExampleVendorComputerSystem.v1_0_0.ExampleVendorComputerSystem",
      "SpecialMode": "Enabled"
    }
  }
}
```

OEM extension 應具備：

- 清楚且不衝突的 vendor namespace。
- 正式 schema。
- `@odata.type`。
- Property type、writable behavior 與 privileges。
- Error messages。
- Versioning 與相容策略。
- Client documentation。

可泛化的功能應優先尋求標準 Redfish schema 或向 DMTF 提案，減少 client 綁定單一平台。

## 22.23 bmcweb 架構

bmcweb 在 OpenBMC 中常負責：

- HTTPS server。
- TLS termination。
- Authentication 與 sessions。
- Route registration。
- Redfish JSON 建立。
- D-Bus query / method call。
- Event、SSE、KVM、WebSocket 等功能，依 build options。

典型 route handler：

```text
註冊 URI 與 HTTP method
        ↓
驗證 authentication / privileges
        ↓
解析 path parameters 與 JSON body
        ↓
查詢 ObjectMapper / D-Bus properties
        ↓
組成 Redfish response
        ↓
必要時建立 Task 或送出 Message Registry error
```

### 22.23.1 非同步處理

bmcweb 大量使用非同步 D-Bus calls。Handler 需要確保：

- 所有 callbacks 完成前 response context仍有效。
- 多個 D-Bus requests 的 errors 能正確合併。
- 找不到 optional object 時不會誤報 internal error。
- Response 不會在資料尚未填完時提前送出。
- Timeout、client disconnect 與 service restart 有清楚行為。

### 22.23.2 D-Bus Mapping

新增 Redfish property 前應確認：

1. 權威 D-Bus interface / property。
2. Object discovery方式。
3. Inventory association。
4. Scale、unit 與 enum mapping。
5. Missing / unavailable behavior。
6. Writable flow 與 rollback。
7. Required privileges。
8. Schema 與 validator expectations。

## 22.24 Security

### 22.24.1 TLS

- 使用有效 server certificate。
- 停用不符合產品政策的舊 TLS versions / ciphers。
- 保護 private key。
- 驗證 certificate replacement 與 rollback。
- 檢查時間錯誤對 certificate validation 的影響。

### 22.24.2 Credentials 與 Tokens

- 不在 shell history、URL、journal 或 crash dumps 保存密碼與 token。
- Token 應有到期與撤銷機制。
- 登出、帳號刪除與權限變更後更新 sessions。
- Debug package 對外分享前遮蔽 secrets。

### 22.24.3 CSRF 與 Browser Client

Cookie-based browser session 需要 CSRF protection。自動化 client 使用 token header 時也應避免將 token暴露給非預期 origin 或 proxy logs。

### 22.24.4 Rate Limit 與 Resource Exhaustion

需限制或管理：

- Login attempts。
- Session count。
- Event subscriptions。
- Firmware uploads。
- Concurrent requests。
- Expensive collection expansion。
- Log / dump generation。

## 22.25 Query Parameters

Redfish 可支援部分 OData query parameters，實際能力依 service 版本與 resource 而異，例如：

- `$select`
- `$expand`
- `$filter`
- `$top`
- `$skip`
- `only`

Client 應依 service capabilities 使用，不應假設所有 resources 上的所有 query options 都可用。

大量使用 `$expand` 可能增加 D-Bus calls、response size 與延遲。bmcweb 需要限制 expansion depth 與資源消耗。

## 22.26 API 測試流程

### 22.26.1 基本 Read

```bash
curl --cacert <ca.pem> \
     -H 'X-Auth-Token: <token>' \
     https://<bmc>/redfish/v1/

curl --cacert <ca.pem> \
     -H 'X-Auth-Token: <token>' \
     https://<bmc>/redfish/v1/Systems

curl --cacert <ca.pem> \
     -H 'X-Auth-Token: <token>' \
     https://<bmc>/redfish/v1/Chassis
```

### 22.26.2 Header

保存：

- HTTP status。
- `Content-Type`。
- `Allow`。
- `ETag`。
- `Location`。
- `Retry-After`。
- `Link`。
- Task-related headers。

### 22.26.3 Negative Tests

- 無 token。
- 無權限帳號。
- 錯誤 Content-Type。
- Malformed JSON。
- Unknown property。
- Read-only property。
- Enum value 錯誤。
- Resource missing。
- Invalid ETag。
- Oversized request。
- Duplicate request。

每種錯誤都應檢查 HTTP status 與 message registry response。

## 22.27 OpenBMC 排查流程

```text
Client Request
        ↓
TLS / Authentication
        ↓
bmcweb Route Match
        ↓
Privilege Check
        ↓
D-Bus Discovery / Call
        ↓
OpenBMC Backend Service
        ↓
JSON Mapping / Response
```

### 22.27.1 Route 不存在

- 確認 URI 與大小寫。
- 讀取 Service Root 和 collection links。
- 確認 bmcweb build option 與 schema support。
- 查看 bmcweb source 的 route registration。

### 22.27.2 Resource 有 URI但資料缺少

- 檢查 D-Bus object / interface / property。
- 檢查 ObjectMapper。
- 檢查 inventory associations。
- 檢查 backend service journal。
- 檢查 bmcweb handler 的 optional / required 判斷。

### 22.27.3 PATCH / Action 失敗

- 檢查 role / privileges。
- 檢查 payload 與 allowable values。
- 檢查目前 resource state。
- 檢查 D-Bus method / property write error。
- 檢查 extended error message。

### 22.27.4 Response 緩慢

- 記錄 client latency。
- 查看 bmcweb CPU / memory。
- 找出 slow D-Bus service。
- 檢查 collection member數量與 `$expand`。
- 檢查 event storm、upload 或 update operation。

## 22.28 常見問題與判讀

| 現象 | 優先方向 | 第一輪檢查 |
|---|---|---|
| `/redfish/v1/` 無回應 | HTTPS / bmcweb | Port、TLS、service、journal |
| 401 | Authentication | Credentials、token、session timeout |
| 403 | Authorization | Role、privilege registry、route permissions |
| 404 | URI / resource discovery | Service Root link、member ID、D-Bus object |
| Property 缺少 | Mapping / backend | Schema support、D-Bus property、association |
| Health 錯誤 | Aggregation policy | Present、Functional、sensor、logging |
| PATCH 回 400 | Payload validation | Type、enum、writable property、message ID |
| PATCH 回 409 | Current state conflict | Power state、update state、resource lock |
| Update 回 202 後沒有進度 | Task / updater integration | Task URI、software service、journal |
| Event 收不到 | Subscription / transport | Destination、filter、TLS、retry |
| Redfish 與 IPMI 資料不同 | 資料來源 / mapping | D-Bus authority、Board / Product fields |
| Collection 偶發漏 member | Object discovery race | Service startup、ObjectMapper、hot-plug |
| Validator 報 schema error | JSON mapping | `@odata.type`、required type、links、enum |
| OEM client 壞掉 | Versioning | OEM schema、namespace、相容性 |

## 22.29 Debug Log 收集

```bash
#!/bin/sh

OUT=/tmp/redfish-debug
mkdir -p "$OUT"

cat /etc/os-release > "$OUT/os-release.txt" 2>&1
uname -a > "$OUT/uname.txt"
cat /proc/cmdline > "$OUT/proc-cmdline.txt"

systemctl status bmcweb --no-pager > "$OUT/bmcweb-status.txt" 2>&1
journalctl -u bmcweb -b --no-pager > "$OUT/bmcweb-journal.txt" 2>&1
systemctl --failed > "$OUT/systemctl-failed.txt" 2>&1

busctl tree xyz.openbmc_project.ObjectMapper \
    > "$OUT/objectmapper.txt" 2>&1

journalctl -b --no-pager | grep -Ei \
    'redfish|bmcweb|authentication|session|update|event|task' \
    > "$OUT/redfish-related-journal.txt" 2>&1

ss -lntp > "$OUT/listening-ports.txt" 2>&1

# 不自動收集帳號資料、session token、private key 或 TLS private material。

tar czf "/tmp/redfish-debug-$(date +%Y%m%d-%H%M%S).tar.gz" \
    -C /tmp redfish-debug
```

保存 client request 時應遮蔽：

- Authorization header。
- X-Auth-Token。
- Cookies。
- Passwords。
- Private keys。
- Firmware update credentials。
- Asset / customer data，依分享範圍評估。

## 22.30 Bring-up 順序

1. 確認 bmcweb 啟動、HTTPS port 與 TLS certificate。
2. 驗證 Service Root。
3. 建立 session，測試 token 與 logout。
4. 依 links 走訪 Systems、Chassis 與 Managers。
5. 比對 D-Bus inventory、state、sensor 與 Redfish resources。
6. 驗證 Status、Health、Asset 與 associations。
7. 測試 GET、PATCH、POST action 與 DELETE。
8. 驗證 AccountService、roles 與 privileges。
9. 驗證 UpdateService、Task 與 firmware inventory。
10. 驗證 EventService、subscription、retry 與 log entries。
11. 驗證 Telemetry / SSE，若平台支援。
12. 執行 malformed payload、unauthorized、conflict 與 ETag tests。
13. 執行 Redfish Service Validator 與產品 profile tests。
14. 測試 service restart、BMC reboot、Host state transition 與 hot-plug。
15. 保存 schema release、bmcweb commit、logs、test output與已知限制。

## 22.31 平台實測紀錄表

| 項目 | URI / 來源 | 實測值 | 備註 |
|---|---|---|---|
| RedfishVersion | Service Root | [待填] | Service 宣告版本 |
| bmcweb version | Package / commit | [待填] | Build options |
| TLS certificate | HTTPS | [待填] | Issuer / expiry |
| Authentication | SessionService | [待填] | Token / mTLS / Basic policy |
| Systems | `/redfish/v1/Systems` | [待填] | Member IDs |
| Chassis | `/redfish/v1/Chassis` | [待填] | Member IDs |
| Managers | `/redfish/v1/Managers` | [待填] | Member IDs |
| AccountService | Service Root link | [待填] | Roles / lockout |
| UpdateService | Service Root link | [待填] | Push / SimpleUpdate |
| TaskService | Service Root link | [待填] | Asynchronous tasks |
| EventService | Service Root link | [待填] | Subscription / retry |
| TelemetryService | Service Root link | [待填] | Supported / unsupported |
| Registries | `/redfish/v1/Registries` | [待填] | Base / Task / Update 等 |
| Schema release | DMTF / build | [待填] | Bundle version |
| Validator | Test result | [待填] | Errors / warnings |
| Interop profile | Test result | [待填] | Product requirements |

## 22.32 驗收 Checklist

Protocol 與 Security：

- [ ] HTTPS、certificate chain、TLS policy與 secure time 已驗證。
- [ ] Session token 建立、使用、timeout 與刪除正常。
- [ ] Authentication 與 authorization 可分開判讀。
- [ ] Roles、privileges、lockout 與 account lifecycle 已測試。
- [ ] Logs 與 debug package 不包含 credentials、tokens 或 private keys。

Resource Model：

- [ ] Service Root links 正確。
- [ ] Systems、Chassis、Managers 的身分與關係正確。
- [ ] Resource URIs 與 member IDs 在 reboot / hot-plug 後保持預期穩定。
- [ ] `@odata.id`、`@odata.type`、ETag 與 links 正確。
- [ ] Inventory、sensor、state 與 D-Bus 權威資料一致。
- [ ] Status.State 與 Health 聚合規則已驗證。

Methods 與 Errors：

- [ ] GET、PATCH、POST action 與 DELETE 依 schema運作。
- [ ] Writable properties、allowable values 與 ActionInfo 正確。
- [ ] HTTP status 與 Message Registry error 能正確對應。
- [ ] ETag / `If-Match` concurrency 行為已測試。
- [ ] Malformed、unauthorized、conflict 與 oversized requests 已測試。

Services：

- [ ] UpdateService 能呈現 inventory、progress、Task 與失敗原因。
- [ ] EventService subscription、filter、TLS、retry 與刪除已測試。
- [ ] Task completion、exception、timeout 與 cleanup 已測試。
- [ ] Telemetry / SSE 依產品支援完成測試。

相容性：

- [ ] Schema bundle 與 bmcweb支援版本已記錄。
- [ ] Redfish Service Validator 無未處理的 errors / warnings。
- [ ] OEM extensions 具備 namespace、schema、privilege 與 version policy。
- [ ] 必要的 interoperability profile 測試已完成。
- [ ] Firmware update、BMC reboot與 backend service restart 後資源可恢復。

## 22.33 本章重點

1. Redfish 使用 HTTPS、JSON、resources、schemas 與 message registries建立標準管理介面。
2. Service Root 是 API 導覽入口，client 應沿 links 找到 collections 與 members。
3. Systems 描述 Host，Chassis 描述實體容器，Managers 描述 BMC。
4. `@odata.id` 表示 URI，`@odata.type` 表示 schema type 與版本，ETag 支援條件式存取。
5. GET 讀取、PATCH 修改、POST 建立 resource 或呼叫 action、DELETE 移除可刪除 resource。
6. Error handling 應同時判讀 HTTP status 與 Message Registry ID。
7. Authentication 確認身分，authorization 透過 roles 與 privileges限制行為。
8. UpdateService、TaskService 與 EventService 需要跨 bmcweb 和 backend services 協作。
9. bmcweb 將多個 D-Bus objects 與 services 映射成 Redfish resource。
10. Schema validation、negative tests、安全測試與產品語意驗證都屬於交付條件。

## 22.34 本章參考資料

- DMTF Redfish Standards: https://www.dmtf.org/standards/redfish
- Redfish Specification DSP0266: https://www.dmtf.org/dsp/DSP0266
- Redfish Data Model Specification DSP0268: https://www.dmtf.org/dsp/DSP0268
- Redfish Schema Bundle DSP8010: https://www.dmtf.org/dsp/DSP8010
- Redfish Standard Registries DSP8011: https://www.dmtf.org/dsp/DSP8011
- Redfish Schema Index: https://redfish.dmtf.org/redfish/schema_index
- DMTF Redfish Publications: https://github.com/DMTF/Redfish-Publications
- OpenBMC bmcweb: https://github.com/openbmc/bmcweb
- bmcweb Redfish documentation: https://github.com/openbmc/bmcweb/blob/master/docs/Redfish.md
