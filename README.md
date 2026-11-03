# 微信账号基础服务 —— 公开接口文档（Control Plane API v1）

本服务是一个**多账号常驻后端**:每个已登录的微信账号在服务内维护一条常驻会话(登录态、心跳、收发)。外部系统通过一套 **HTTP REST + SSE** 接口驱动登录、下发命令、实时接收消息与事件。

本文为公开学习版，介绍 HTTP / SSE 接口、字段、调用流程与错误处理。接口调用示例中的地址、账号、标识符、时间、设备资料和内容均为合成数据或占位值；不能用于实际登录或授权。服务地址与接入凭据由服务提供方单独分配。

本版省略服务运行信息、底层协议映射、安全材料结构、生成算法与内部策略。标为不透明上下文的字段应来自实际接口响应并原样回传，不应根据示例构造。涉及媒体处理、额外验证或实验性能力的完整接入说明，应向服务提供方取得。

## 交流学习

欢迎通过 Telegram 交流接口设计、开发实践与技术学习。

Telegram：[@opencode666](https://t.me/opencode666)

也可扫描下方二维码联系：

<img src="telegram-opencode666.png" alt="Telegram @opencode666 联系二维码" width="260">

---

## 目录

1. [快速开始](#1-快速开始)
2. [连接与鉴权](#2-连接与鉴权)
3. [通用约定](#3-通用约定)
4. [错误与状态码](#4-错误与状态码)
5. [端点速查](#5-端点速查)
6. [端点详解](#6-端点详解)
7. [命令集](#7-命令集)
8. [事件类型（SSE）](#8-事件类型sse)
9. [订阅类型](#9-订阅类型)
10. [登录状态机](#10-登录状态机)
11. [代理阶段与租约](#11-代理阶段与租约)
12. [调用语义（务必阅读）](#12-调用语义务必阅读)
13. [端到端对接流程](#13-端到端对接流程)
14. [对接方常见问题（FAQ / 待确认项）](#14-对接方常见问题faq--待确认项)

---

## 1. 快速开始

一个账号从上号到收消息的最小闭环:

```bash
BASE=https://api.example.invalid:8443

# 1) 起登录，拿二维码
curl -sX POST $BASE/v1/accounts/login -H 'Content-Type: application/json' -d '{}'
# → {"login_id":"a1b2c3","qr_png_base64":"<QR_IMAGE_BASE64>","phase":"qr_pending",...}
# 把 qr_png_base64 解码成图片(实际为 JPEG)展示给用户扫码

# 2) 拿到 login_id 后立即订阅本次登录；额外验证材料会实时到达
curl -sN $BASE/v1/accounts/login/a1b2c3/events
# -> event: login_status
# -> data: {"phase":"scanned","verification_required":true,"auth_url":"https://verify.example.invalid/challenge",...}

# 轮询仍可作为断线兜底，直到 phase=online、拿到 uin
curl -s $BASE/v1/accounts/login/a1b2c3
# → {"phase":"online","uin":123456789,...}

# 3) 打开该账号的消息订阅
curl -sX POST $BASE/v1/accounts/123456789/subscription \
  -H 'Content-Type: application/json' -d '{"enabled":true,"type":"messages"}'

# 4) 保持一条 SSE 长连，实时收消息与事件
curl -sN $BASE/v1/accounts/123456789/events?since=0
# → 持续推送 event: message / state / login / command_ready / operation ...
```

---

## 2. 连接与鉴权

| 项 | 说明 |
|---|---|
| 协议 | 生产环境 **HTTPS**(SSE 携带真实聊天内容,必须加密传输)|
| 鉴权 | 默认 **双向 TLS(mTLS)**:客户端需提供服务端 CA 签发的客户端证书,并信任服务端证书。另建议配合 IP allowlist。|
| 路径前缀 | 所有接口以 `/v1` 开头 |
| 端口 | 由部署方约定(示例用 `8443`)|

使用客户端证书的 curl 示例:

```bash
curl --cert client.crt --key client.key --cacert server-ca.crt \
     -s https://api.example.invalid:8443/v1/healthz
```

> 请向服务提供方获取:服务地址与端口、服务端 CA 证书、你的客户端证书与私钥。

---

## 3. 通用约定

- 请求体与响应体均为 **JSON**(`Content-Type: application/json`)。SSE 端点返回 `text/event-stream`。
- **读命令**同步返回结果;**写命令**返回 `{operation_id, status}`(见 §6.6、§12)。
- 服务端异常一律**脱敏**后返回,不泄漏内部细节:错误统一为 `{"error": "..."}`。

**核心标识符**

| 标识符 | 含义 | 生命周期 |
|---|---|---|
| `login_id` | 一次登录尝试的不透明字符串标识| 登录进行期间;终态后可弃 |
| `uin` | 微信账号数字 id | 登录成功后确定,账号在册期间稳定;是所有账号级接口的 key |
| `operation_id` | 一次已接受命令的不透明字符串标识| 可查询命令状态与终态 |
| `control.idempotency_key` | 写命令调用方提供的幂等键 | 同一账号内关联重复写命令与终态 |
| `control.request_id` | 读命令调用方提供的请求 id | 同一账号内查询读命令状态与终态 |
| `generation` | 账号的“代”序号,每次重登录 +1 | SSE 游标 `since` 仅在**同一代内**有效(见 §12.5)|
| `lease_id` | 代理租约 id(可选,由你分配)| 用于关联代理登录事件 |

---

## 4. 错误与状态码

| HTTP 码 | 场景 | 响应体 |
|---|---|---|
| `200` | 读成功 / 同步写终态 / 查询成功;幂等重提可能返回原操作当前状态 | 各端点结果体 |
| `202` | 起登录成功;或异步写被持久接受 | `{login_id,...}` 或 `{operation_id,"status":"ACCEPTED"}` |
| `400` | 请求体缺字段 / 枚举值非法 | `{"error":"..."}` |
| `404` | 未知 `uin` / `login_id` / `operation_id` / 路径 | `{"error":"..."}` |
| `409` | 起登录时该账号**已在线**(幂等冲突,见 §6.2)| `{"error":"...","code":"ALREADY_ONLINE","phase":"online","uin":...,"generation":...}` |
| `409` | 账号尚未就绪接收命令 | `{"error":"account is not ready for commands","code":"ACCOUNT_NOT_READY"}`;命令未创建、未入队、未执行 |
| `503` | 服务未启用登录 / 未启用代理事件 / 未配置账号事件流，或命令容量已满 | 过载时为 `{"error":"command capacity is full","code":"COMMAND_OVERLOADED","retry_after_ms":1000,"operation_id"?:"...","request_id"?:"..."}` |
| `504` | 本次 HTTP 等待超过 `deadline_ms`;不代表持久 operation 已终态 | `{"error":"command deadline exceeded","status":"DEADLINE_EXCEEDED","operation_id"?:"...","request_id"?:"...","failure":{...}}` |
| `500` | 其他内部错误(已脱敏)| `{"error":"..."}` |

**`deadline_ms` 与 `504` 的语义**：请求等待时限默认 `30000` 毫秒，范围 `1`–`120000`。HTTP `504` 只表示本次等待超时，operation 可能仍为 `ACCEPTED` / `RUNNING`。按读命令的 `request_id` 或写命令的 `operation_id` / `idempotency_key` 查询，确认终态后再释放账号队列（§12.7）。operation 查询里的 `DEADLINE_EXCEEDED` 是不可再变更的终态。写命令的结果核对规则为：

- 明确未发送的命令可以通过 `failure.request_sent=false` 识别；是否重提仍须遵守该命令的调用规则。
- 已发送且结果未知的写命令以 `OUTCOME_UNKNOWN` 表示；使用操作标识核对，不应自动重发。若已取得明确结果，则返回对应成功或失败状态。
- **写命令永不自动重试。**

**`failure` 结构**：失败和超时结果可包含 `code`、`stage`、`request_sent`、`retryable`、`retry_after_ms`。`request_sent=false` 表示确定未发送，`true` 表示请求已写出但不保证上游收到或处理，`null` 表示无法确认。重试提示不能代替写操作的幂等和结果核对规则。

`stage` 是错误分类标识，如 `queue`、`proxy_connect`、`connect`、`response_parse`、`local`；调用方不得只凭阶段猜测请求是否发送，应读取 `request_sent`。私信同步额外的 `value.sync_failure` 见 §7.1.3。

代理相关错误包括 `PROXY_CONNECT_TIMEOUT`、`PROXY_AUTH_FAILED`、`PROXY_CONNECT_FAILED`。错误码用于提示连接、认证或超时问题，不提供服务端网络实现细节。

需要更长等待时，可在范围内提高 `deadline_ms`。异步 `202` 表示命令已被接受，仍须等待终态。容量拒绝返回 `503 COMMAND_OVERLOADED` 及 `retry_after_ms`；若附 `operation_id`，可查询对应 `REJECTED` 终态。已接受后才被拒绝的同步写命令可能返回 `200` / `status=REJECTED`。每条写命令建议提供唯一 `control.idempotency_key`，以便响应丢失后核对。

---

## 5. 端点速查

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/v1/healthz` | 存活探针 |
| POST | `/v1/accounts/login` | 起登录,返回二维码 |
| GET | `/v1/accounts/login/{login_id}` | 查登录状态(含 `expired`)|
| GET | `/v1/accounts/login/{login_id}/events` | **SSE** 单次登录实时状态流；推送额外验证 URL 与完整 WebView 材料(§6.3.2)|
| DELETE | `/v1/accounts/login/{login_id}` | 取消进行中的登录(刷新二维码用,§6.3.1)|
| GET | `/v1/accounts` | 列账号 + 状态 + **当前会话来源**(login_id/上线时间/代理/订阅/写能力) + **登录设备**(`device`,§6.4)|
| GET | `/v1/accounts/events` | **SSE** 节点级账号事件流(所有账号的登录阶段 + 上下线,§6.12)|
| GET | `/v1/accounts/{uin}/subscription` | 只读:当前订阅快照(§6.9)|
| GET | `/v1/accounts/{uin}/capabilities` | 只读:当前写能力快照(§6.10)|
| DELETE | `/v1/accounts/{uin}` | 关闭账号会话(**保留身份可续登**,§6.5)|
| DELETE | `/v1/accounts/{uin}/data` | ★ 彻底删除账号 + 清所有本地数据(管理员;删后不再 resume,§6.5.1)|
| POST | `/v1/accounts/{uin}/commands` | 下发命令(读/写)—— **视频号推荐流/搜索/关注列表/主页/详情/评论/关注/私信、朋友圈、通讯录 全部经此端点的 `method` 提供,见 §7.0 功能对照总览** |
| GET | `/v1/accounts/{uin}/operations/{operation_id}` | 查写操作终态 |
| GET | `/v1/accounts/{uin}/operations?idempotency_key={key}` | 按写命令幂等键查状态/终态 |
| GET | `/v1/accounts/{uin}/operations?request_id={id}` | 按读命令请求 id 查状态/终态 |
| GET | `/v1/accounts/{uin}/events` | **SSE** 事件流(收消息)|
| POST | `/v1/accounts/{uin}/subscription` | 开/关订阅 |
| POST | `/v1/accounts/{uin}/capabilities` | 开/关写能力 |
| GET | `/v1/proxy/events` | **SSE** 代理租约事件流 |

---

## 6. 端点详解

### 6.1 `GET /v1/healthz` — 存活探针

```bash
curl -s $BASE/v1/healthz
```
**200**
```json
{ "status": "ok", "accounts": 3 }
```
`accounts` = 当前在册账号数。

### 6.1.1 `GET /v1/readyz` 与 `GET /v1/recovery` — 服务就绪与恢复进度

`healthz` 表示服务存活，不能单独证明账号可以执行命令。`readyz` 在服务尚未完成恢复或未达到就绪条件时返回 **503**；正常就绪时返回 **200**。

`recovery` 用于查看账号恢复进度，包含 `schema_version`、`instance_id`、`recovery` 和 `inventory`。`instance_id` 为不透明实例标识；公开版不列出身份校验摘要和运维清单。调用方可根据 `recovery.accounts` / `counts` 中的 `queued`、`preparing`、`connecting`、`retry_wait`、`ready`、`reauth_required`、`failed`、`cancelled` 状态展示进度。

`settled` 表示本轮均已取得结果，`complete` 还要求没有失败或取消。需要恢复但尚不可执行命令的账号可能显示 `state=restore_*`、`command_ready=false`；此时不得提交业务命令。临时网络问题不等同于需要重新扫码，认证失效与网络恢复失败应分别处理。

恢复的详细运维字段与验收规则不在本公开版范围内。业务命令是否可用，以账号的 `command_ready` 为准。

---

### 6.2 `POST /v1/accounts/login` — 起登录

**服务重启后的会话恢复**：启用会话恢复时，服务启动后可尝试恢复已保存的会话，不调用本接口、不产生二维码。调用方应先查询账号恢复状态并等待 `command_ready=true`，避免同时发起扫码登录。恢复失败不会自动开始扫码；需要用户重新登录的状态及处理方式见 §10.1。

发起一次扫码登录。服务端自动判定新设备登录或老设备续登,并为该登录起独立后台推进流程,期间通过 SSE 发 `login` 事件。

**请求体**(全部可选)

| 字段 | 类型 | 说明 |
|---|---|---|
| `account_id` | string | 可选。指定使用某已缓存身份(别名或 uin)|
| `proxy` | object | 可选。走代理登录;省略 = 直连 |
| `proxy.scheme` | string | `socks5` / `socks5h` / `http`;`socks5h` 与 `socks5` 使用相同 SOCKS5 协议,目标域名以 ATYP=0x03 发给代理端解析 |
| `proxy.host` | string | 代理主机 |
| `proxy.port` | int | 代理端口 |
| `proxy.username` / `proxy.password` | string | 可选。凭据不会出现在任何事件或日志中 |
| `proxy.label` | string | 可选。仅用于日志/上报标记,不参与选择;最多 64 字节,仅允许 ASCII 字母/数字及 `. _ : -`(正则 `[A-Za-z0-9._:-]*`) |
| `proxy.egress_check_url` | string | 可选。启用代理出口 IP 自动观测的 HTTPS 回显端点;示例 `https://egress.example.invalid/`;省略时不探测 |
| `proxy.reconnect_budget_sec` | int | 可选。长连接瞬时断线后的自动重连时长预算,范围 1–3600 秒,默认 300 秒;超时未恢复后转为 `recovery_exhausted`，保留检查点且 `reauth_required=false`；只有明确认证失效才要求重登 |
| `lease_id` | string | 可选。你为该登录分配的代理租约 id,会在代理事件中回显 |

```bash
curl -sX POST $BASE/v1/accounts/login -H 'Content-Type: application/json' -d '{
  "proxy": {"scheme":"socks5h","host":"192.0.2.10","port":1080,"username":"u","password":"p","label":"proxy-example-001","egress_check_url":"https://egress.example.invalid/","reconnect_budget_sec":300},
  "lease_id": "lease-001"
}'
```

**202**
```json
{
  "login_id": "a1b2c3",
  "qr_png_base64": "<QR_IMAGE_BASE64>",
  "expires": 1700000000.0,
  "phase": "qr_pending",
  "error": null,
  "lease_id": "lease-001",
  "proxy_phase": "login_bootstrap",
  "verification_required": false,
  "auth_url": null,
  "webview_user_agent": null,
  "c_utility_guid": null,
  "security_info": null,
  "ccdata": null,
  "verification_material_ready": false,
  "verification_material_error": null
}
```
- `qr_png_base64`:base64 解码后为**二维码图片字节**,展示给用户扫码。⚠️ **字段名带 `png` 属历史命名,实际图片格式为 JPEG**(由微信服务端返回)。请按**不透明图片字节**处理(用 magic bytes 判定:`FF D8 FF`=JPEG、`89 50 4E 47`=PNG),不要假定为 PNG。
- `expires`:二维码过期时刻,**绝对 Unix 秒时间戳**(= 服务器当前 Unix 秒 + 二维码有效秒数,默认有效期约 300 秒)。直接按 Unix 秒解析即可得到真实过期时间。
- `phase`:登录阶段(§10)。**即使代理早期失败**也返回 202,此时 `phase="failed"` 且 `error` 有值,仍带 `login_id`。
- `proxy_phase`:代理阶段(§11);直连时可能为 `null`。
- `verification_required` / `auth_url`:扫码后若微信要求额外安全验证，前者为 `true`，后者为本次登录的一次性 HTTPS 验证 URL。URL 查询参数可能含敏感 `ticket`；调用方只应在当前登录界面使用，不得写日志、持久化、缓存或跨登录复用。
- `webview_user_agent` / `c_utility_guid` / `security_info` / `ccdata`：一次额外验证所需的不透明上下文。只有 `verification_material_ready=true` 时四项才同时可用，必须与同一次登录的 `auth_url` 配套使用。页面加载时使用返回的 `webview_user_agent`；其余字段按授权接入说明传递。不公开其生成方式、内部结构或字段之间的关联，不得自行拼接、解码、缓存或跨登录复用。
- `verification_material_error`:若 URL 已收到但生成失败，服务保留 URL 用于诊断、四项材料均返回 `null`，并给出稳定的脱敏错误码；此时不得用空值继续验证。登录确认、取消、失败、超时或登录执行进程退出后，URL 和整组材料都会清除。登录响应均带 `Cache-Control: no-store, private` 与 `Pragma: no-cache`。
- `socks5h` 与 `socks5` 的目标域名均由代理端解析。
- `proxy.egress_check_url` 仅接受 HTTPS、443 端口及符合要求的 DNS 主机名，不接受 IP 字面量、userinfo、fragment 或本地域名；调用方应使用自己获准使用的回显服务。省略时不检测。检测失败只更新观测状态，不代表账号掉线。
- `proxy.reconnect_budget_sec` 控制临时断线后的自动恢复等待范围。恢复期间 `command_ready=false`；预算内 `state` 可保持 `online`，耗尽后转为 `recovery_exhausted`，`reauth_required=false`。明确认证失效时转为重新登录状态；内部重试算法和时序不在公开版中披露。
- 配置出口检测后，登录及连接恢复可产生新的观测结果；不应把它作为固定出口保证或命令就绪条件。
- 探测结果通过 `proxy_egress` SSE 事件上报,也合并在 `GET /v1/accounts/login/{login_id}` 的 `proxy` 快照中。失败会报告 `check_status:"failed"` 和有限的 `error_code`,并清空该次检查对应的出口 IP;失败不会让账号掉线。快照的 `egress_epoch` 应与同一 `proxy` 对象的当前 `epoch` 比较;两者不同时,出口值是上一个连接 epoch 的最近结果,新一轮探测尚未完成。成功只表示**该次新建回显连接**观察到的代理出口;代理若按连接轮换 IP,不能据此证明微信长连接使用同一个出口。要强绑定,代理必须提供固定出口或会话粘滞能力。

**503**:服务未启用登录。

**扫码账号准入**：指定 `account_id` 时须由对应账号扫码，扫错号返回 `scan_wrong_number`。账号已在线、正在登录、或应从已有账号续登入口操作时，可能分别返回 `account_already_online`、`account_login_in_progress`、`existing_account_requires_resume`。调用方按错误提示使用正确账号或入口，不应展示错误中的账号身份细节。

**幂等语义(重复对同一账号起登录)**:当请求带 `account_id`(uin 串或已缓存别名)时,服务保证不重复起登录、不重复出二维码:

- **已有进行中的登录**(该 `account_id` 的上一次登录处于 `starting`/`qr_pending`/`scanned`/`confirmed`/`authenticating`):**返回原来那次登录**——同一个 `login_id`、它当前的 `phase` 和它已持有的二维码(仍以 `202` 返回)。不会开第二个登录。⚠️ 因此想**换新码**不能只靠再 POST(会拿回同一个旧码)——要先 `DELETE …/login/{login_id}` 取消旧的再 POST(见 §6.3.1)。
- **账号已在线**(该 `account_id` 已有在线会话):**返回 `409`**,体形如 `{"code":"ALREADY_ONLINE","phase":"online","uin":<uin>,"generation":<代>}`。不会开第二个登录、不出二维码。要下号请先 `DELETE /v1/accounts/{uin}`。
- ⚠️ **不带 `account_id` 的全新扫码登录无法去重**:服务在认证出 `uin` 之前没有可用的去重键,故请对同一账号的并发"全新登录"自行加锁(不要并发发起)。

---

### 6.3 `GET /v1/accounts/login/{login_id}` — 查登录状态

轮询登录进展(作为 SSE `login` 事件的可靠兜底)。

```bash
curl -s $BASE/v1/accounts/login/a1b2c3
```
**200**
```json
{
  "login_id": "a1b2c3",
  "phase": "online",
  "uin": 123456789,
  "command_ready": false,
  "command_ready_at": null,
  "error": null,
  "expired": false,
  "lease_id": "lease-001",
  "proxy_phase": "longlink",
  "verification_required": false,
  "auth_url": null,
  "webview_user_agent": null,
  "c_utility_guid": null,
  "security_info": null,
  "ccdata": null,
  "verification_material_ready": false,
  "verification_material_error": null,
  "proxy": {
    "lease_id": "lease-001", "login_id": "a1b2c3", "uin": 123456789,
    "generation": 0, "lease_state": "active", "proxy_phase": "longlink",
    "ok": true, "terminal": false, "release_reason": null, "release_event_id": null,
    "last_heartbeat_seq": 42, "observed_egress_ip": "198.51.100.10", "region": "US",
    "egress_check_status": "success", "egress_error_code": null,
    "egress_ts": 1700000002.0, "egress_epoch": 1
  }
}
```
- `uin`:在 `phase` 达到 `confirmed` 之后有值;`online` 时账号已上线。
- `command_ready`:该账号此刻是否已完成启动同步并能接收命令;`command_ready_at` 为最近一次进入就绪态的 Unix 秒时间戳,未就绪为 `null`。这是独立于 `phase=online` 的状态,判定与变化事件见 §6.6/§6.12。
- `proxy`:仅代理登录时出现,合并了代理租约权威快照(见 §11)。
- `verification_required` / `auth_url`：需要额外验证时返回 `true` 和一次性 HTTPS URL。验证材料只用于当前登录，不能记录、持久化或跨尝试复用。
- `webview_user_agent` / `c_utility_guid` / `security_info` / `ccdata` / `verification_material_ready`：含义同 §6.2；必须将同一响应中的材料作为整体使用。`verification_material_error` 非空时停止处理，不使用旧材料补齐。
- `expired`：仅在 `phase="qr_pending"` 时用于判断二维码是否已过期。为 `true` 时按 §6.3.1 取消后重新申请，不必等待登录自行失败；其它阶段应按当前 `phase` 处理。
- **404**:未知 `login_id`。
- **保留期**:登录进入终态(`online`/`failed`/`aborted`)后,该记录**至少保留 24 小时**可查（服务重启后可能不可用）。轮询中断后恢复,只要在 24h 内仍可查到那次登录的最终结果。24h 后被回收(一个仍在线账号的 `online` 记录会保留到它下线为止,始终可经 §6.4 从账号反查)。

---

### 6.3.1 `DELETE /v1/accounts/login/{login_id}` — 取消进行中的登录(刷新二维码)

取消一次**还停在扫码阶段**(`starting`/`qr_pending`/`scanned`)的登录。用于用户点「刷新二维码」:**先 `DELETE` 取消旧登录,再 `POST /v1/accounts/login` 起新登录拿新码**(因幂等 §6.2,不先取消的话新 POST 会返回同一个旧码)。

```bash
curl -sX DELETE $BASE/v1/accounts/login/a1b2c3
```
**200**
```json
{ "login_id": "a1b2c3", "aborted": true, "phase": "aborted" }
```
| 字段 | 说明 |
|---|---|
| `aborted` | `true` = 本次调用把它取消了(`phase` 变 `aborted`,租约释放);`false` = 未取消(见下)|
| `phase` | 取消后的当前 phase |

- **已过扫码阶段不取消**(`confirmed`/`authenticating`/`online`):此时人已扫码、认证在途,取消会与绑定竞争,故**拒绝**——返回 `aborted:false` + 当前 `phase`,登录照常推进。要给一个**已在线**账号下号请用 `DELETE /v1/accounts/{uin}`(§6.5),不是这个。
- **幂等**:对已 `aborted` 的登录重复 `DELETE` 仍返回 `aborted:true`。
- 取消后,该 `account_id` 的下一次 `POST /login` 不再算"进行中",会开新登录、给**新二维码**。
- **404**:未知 `login_id`(或已被 24h 回收)。

> **刷新二维码标准流程**:轮询 `GET …/login/{id}` 见 `expired:true`(或用户主动点刷新)→ `DELETE …/login/{id}` → `POST /v1/accounts/login {account_id}` 拿新码。两次快调用即得新码。

---

### 6.3.2 `GET /v1/accounts/login/{login_id}/events` — 单次登录实时状态流(SSE)

取得 `login_id` 后立即订阅本流。首帧是当前登录快照，之后在阶段或验证材料变化时发送 `login_status`。调用方可即时更新扫码与额外验证界面。

```bash
curl -sN "$BASE/v1/accounts/login/a1b2c3/events"
```

需要额外验证时的示意帧：

```text
event: login_status
data: {"login_id":"a1b2c3","phase":"scanned","uin":null,"error":null,"lease_id":"lease-001","proxy_phase":"login_cgi","verification_required":true,"auth_url":"https://verify.example.invalid/challenge?ticket=EXAMPLE_ONLY","webview_user_agent":"<opaque-user-agent>","c_utility_guid":"<opaque-guid>","security_info":"<opaque-value>","ccdata":"<opaque-value>","verification_material_ready":true,"verification_material_error":null,"command_ready":false,"command_ready_at":null,"expired":false}
```

- `data` 与同一时刻 `GET /v1/accounts/login/{login_id}` 的 `LoginStatus` 结构一致；`auth_url`、`webview_user_agent`、`c_utility_guid`、`security_info`、`ccdata` 来自同一完整快照。
- `verification_required=true` 但 `verification_material_ready=false` 时，`auth_url` 仍返回，四项 WebView bridge 材料全部为 `null`，`verification_material_error` 给出稳定错误码。调用方不得拼用上一帧或另一次登录的材料。
- 登录进入 `online` / `failed` / `aborted` 时，服务发送一帧已清除上述一次性字段的终态快照，然后正常结束该流。
- 此流不提供历史敏感材料重放，也不接受 `Last-Event-ID` 追赶。网络重连后重新订阅同一路径，首帧就是当前权威快照；若登录记录已过保留期则返回 `404`。
- 响应带 `Cache-Control: no-store, private`、`Pragma: no-cache` 与 `X-Accel-Buffering: no`。
- 慢消费者的有界队列溢出时连接会关闭；调用方重新建立连接即可获得当前快照。空闲期间服务发送 SSE keepalive 注释。
- **404**：未知 `login_id`。**503**：此登录事件能力不可用。

该流只服务一个 `login_id`。全节点 `/v1/accounts/events` 仍用于非敏感的阶段和上下线提示，不包含 `auth_url`、UA、GUID、`security_info` 或 `ccdata`。

---

### 6.4 `GET /v1/accounts` — 列账号

列出所有在册账号及其状态。即使 SSE 中断,关键状态(如需要重登录)也能在此查到。
`device` 为登录设备信息对象或 `null`，用于展示该账号登录时使用的设备。

已保存的 `reauth_required` 状态可在重启后恢复为在册终态。该恢复不递增 `generation`，不代表在线或重新认证，`command_ready=false`，业务命令返回 `409 ACCOUNT_NOT_READY`。调用方须引导用户显式重新登录；服务不会自动重试登录。

```bash
curl -s $BASE/v1/accounts
```
**200**
```json
{
  "accounts": [
    {
      "uin": 123456789,
      "generation": 0,
      "state": "online",
      "reauth_required": false,
      "last_fault": null,
      "command_ready": true,
      "command_ready_at": 1700000001.0,
      "device": {
        "template_id": "EXAMPLE_DEVICE_TEMPLATE",
        "device_model": "EXAMPLE_DEVICE_MODEL",
        "model_name": "示例 iPad",
        "device_name": "iPad",
        "device_type": "iPad",
        "os_type": "EXAMPLE_OS_TYPE",
        "os_version": "EXAMPLE_OS_VERSION",
        "os_build": "EXAMPLE_OS_BUILD"
      },

      "login_id": "a1b2c3",
      "online_since": 1700000000.0,
      "lease_id": "lease-001",
      "proxy": {
        "lease_id": "lease-001", "login_id": "a1b2c3", "uin": 123456789,
        "generation": 0, "lease_state": "active", "proxy_phase": "longlink",
        "ok": true, "terminal": false, "release_reason": null, "release_event_id": null,
        "last_heartbeat_seq": 42, "observed_egress_ip": "198.51.100.10", "region": "US",
        "egress_check_status": "success", "egress_error_code": null,
        "egress_ts": 1700000002.0, "egress_epoch": 1
      },

      "subscribe_messages": true,
      "data_subscriptions": ["contacts", "profile"],
      "capabilities": ["finder.like", "message.mark_read"]
    }
  ]
}
```
| 字段 | 说明 |
|---|---|
| `state` | 会话态:`online` / `reauth_required` / `closed` / `offline`(语义见 §10)|
| `reauth_required` | `true` = 账号已掉线需重新登录(重新调 `POST /v1/accounts/login`)|
| `last_fault` | 最后一次故障详情,可为 `null` |
| `command_ready` | `true` 表示启动同步已完成,可接收业务命令;不由 `phase=online` 单独推断 |
| `command_ready_at` | 最近一次变为就绪的绝对 Unix 秒时间戳;未就绪时为 `null` |
| `device` | 该账号当前 `generation` 对应的持久化登录设备展示对象；不可确认时为 `null`。对象字段见下表 |
| `login_id` | **让当前会话上线的那次登录**的 id;可据此 `GET …/login/{login_id}` 反查其最终结果,或安全地自动接上这个会话。会话非本服务上线(未知来源)时为 `null` |
| `online_since` | 上线时刻(**绝对 Unix 秒**),`null` 表示未知 |
| `lease_id` | 让它上线的那次登录使用的代理租约 id(直连登录为 `null`)|
| `proxy` | 该登录的代理快照,字段与 `GET …/login/{login_id}` 里的 `proxy` 完全一致(直连或无快照时为 `null`)|
| `subscribe_messages` | 当前是否开启消息订阅 |
| `data_subscriptions` | 当前开启的业务数据订阅类型列表(见 §9)|
| `capabilities` | 当前已开启的写能力列表(见 §6.10)|

> `login_id`/`online_since`/`lease_id`/`proxy` 按 `(uin, generation)` 关联到那次上线的登录;跨重登录(generation 会 +1)不会串。若账号是以本服务之外的方式恢复上线的,这四项为 `null`(此时仍可用 `state` 判断在线)。
> `subscribe_messages`/`data_subscriptions`/`capabilities` 也可分别用只读的 `GET …/subscription`、`GET …/capabilities` 单独查(§6.9/§6.10),核对时无需发修改请求。

`device` 非空时固定包含以下字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `template_id` | string / null | 登录设备的模板 ID，如 `EXAMPLE_DEVICE_TEMPLATE`；旧身份或非设备池身份为 `null` |
| `device_model` | string | 登录设备的硬件型号标识，如 `EXAMPLE_DEVICE_MODEL` |
| `model_name` | string / null | 设备型号的展示名称，如 `示例 iPad`；未知型号为 `null` |
| `device_name` | string | 登录身份中的设备名，如 `iPad` |
| `device_type` | string | 登录身份中的设备类型，如 `iPad` |
| `os_type` | string | 登录身份中的系统类型串，如 `EXAMPLE_OS_TYPE` |
| `os_version` | string / null | 系统版本展示值，如 `EXAMPLE_OS_VERSION`；精度为 major.minor，不含补丁版本；旧身份系统类型不含可解析版本时为 `null` |
| `os_build` | string / null | 系统 build，如 `EXAMPLE_OS_BUILD`；目录未知时为 `null` |

设备信息对应当前账号代次；未知或不可用时 `device=null`，不影响账号状态的读取。未知的型号名称、系统版本或 build 可以为 `null`，客户端应兼容。所有设备标识均应按服务返回值展示，不应据公开示例推断支持目录或创建设备。

客户端可将 `model_name`（空时用 `device_model`）与 `os_version` 用于展示。设备对象只用于显示，不作为认证凭据或自行构造登录材料的依据。

---

### 6.5 `DELETE /v1/accounts/{uin}` — 关闭账号(**只关会话,保留身份可续登**)

停止该账号会话，保留账号身份与同步状态。之后可以从原账号入口重新扫码登录；保留身份不表示免扫码。显式关闭后，不会在下次服务启动时自动恢复该会话。彻底清除数据使用 §6.5.1。

```bash
curl -sX DELETE $BASE/v1/accounts/123456789
```
**200**
```json
{ "uin": 123456789, "closed": true }
```
**404**:未知账号(无活动会话)。

---

### 6.5.1 `DELETE /v1/accounts/{uin}/data` — ★ 彻底删除账号 + 清除服务所有账号数据(管理员)

删除当前服务管理的该账号数据，包括会话、身份、同步状态、订阅、操作记录及登录尝试。删除后不再恢复原会话，下一次登录需要重新扫码。该操作幂等，在线和离线账号均可调用；外部备份不在接口清理范围内。

这是不可逆的管理员操作，必须在接入层限制调用权限。服务返回 `200` 才代表删除完整完成；失败或部分完成应按返回的错误码处理。

```bash
curl -sX DELETE $BASE/v1/accounts/123456789/data
```
**200**
```jsonc
{
  "uin": 123456789,
  "purged": true,
  "was_live": true,                       // 删除时是否有在线会话(已一并驱逐)
  "identity_purged": ["123456789","wxid_xxx"],  // 实际清掉的身份键(uin + 已知别名 wxid/profile_id)
  "journal_purged": true,                 // 是否已清理账号操作记录
  "session_purged": true,                 // 是否已清理会话状态
  "host_cache_purged": true,              // 是否清掉该账号的路由缓存
  "acceptance_records_purged": 2,         // 已清理的接受记录数
  "login_attempts_purged": 1              // 清掉的内存登录 attempt 数
}
```
**404**：当前服务没有该账号的可删除数据。
**409**:`ACCOUNT_PURGE_QUIESCE_PENDING` 或 `ACCOUNT_PURGE_RACED_LOGIN`：旧会话尚未确认停止，或最终删除前已有新登录绑定。此时不会开始持久层删除，调用方应稍后重试同一请求。
**500**：`ACCOUNT_PURGE_PREPARE_FAILED`、`ACCOUNT_PURGE_QUIESCE_FAILED` 或 `ACCOUNT_PURGE_INCOMPLETE` 表示删除未完成。响应可带 `retryable`、`failed_components` 和 `partial`；`partial.purged=false` 不能视作成功，应使用同一 UIN 重试或联系服务提供方。
**503**:服务未接入身份存储(未启用登录的部署),无法删除身份 → 账号删除功能不可用。

> 删除权限必须由服务提供方与接入方明确授权。删除失败不会回滚已经完成的部分；按同一 UIN 重试，直到确认完成。公开版不披露内部数据布局、清理步骤或防护元数据。

---

### 6.6 `POST /v1/accounts/{uin}/commands` — 下发命令

对某账号下发一条读或写命令。可用命令见 §7。

**请求体**

| 字段 | 类型 | 说明 |
|---|---|---|
| `method` | string | 命令方法名(§7)|
| `params` | object | 该命令的参数(必须可 JSON 序列化)|
| `control.deadline_ms` | int | 端到端预算,默认 `30000`,范围 `1`–`120000`;超时返回 `504`；仍须查询命令终态 |
| `control.authorize` | bool | **写命令**逐次授权位,默认 `false`;`true` 才允许改变账号状态 |
| `control.idempotency_key` | string | **写命令**幂等去重键 |
| `control.request_id` | string | **读命令**可选请求 id;未传时服务自动生成。响应、`504`、终态 SSE 与按键查询使用同一 id |
| `account_generation` | int/null | 可选。钉住一个代;与当前代不符的迟到结果判 `STALE` 丢弃 |

**读命令示例**
```bash
curl -sX POST $BASE/v1/accounts/123456789/commands \
  -H 'Content-Type: application/json' -d '{
  "method": "finder.search",
  "params": { "query": "关键词", "limit": 10 },
  "control": { "request_id": "finder-search-example-001", "deadline_ms": 30000 }
}'
```
**读命令响应 200**(同步返回结果)
```json
{
  "account_id": "123456789",
  "generation": 0,
  "operation_id": "…",
  "status": "SUCCEEDED",
  "request_id": "finder-search-example-001",
  "value": { "...": "读结果，结构随命令而定（见 §7）" },
  "error": null,
  "outcome": null
}
```

**写命令示例**
```bash
curl -sX POST $BASE/v1/accounts/123456789/commands \
  -H 'Content-Type: application/json' -d '{
  "method": "message.mark_read",
  "params": { "to_user_name": "wxid_abc", "read_seq": 100 },
  "control": { "authorize": true, "idempotency_key": "markread-example-001" }
}'
```
写命令响应形状恒为 `{operation_id, status, ...}`,分两种模式(由服务端配置决定):

- **同步模式(默认)**:阻塞到终态,`200` 返回带终态的完整结果:
  ```json
  { "account_id":"123456789","generation":0,"operation_id":"deadbeef",
    "status":"SUCCEEDED","value":{"ret":0,"ok":true},"error":null,"outcome":"SUCCEEDED" }
  ```
- **异步模式**:写被持久接受后立即 `202` 返回,终态随后经 SSE `operation` 事件(§8.5)投递,并可用 `GET …/operations/{id}` 查询:
  ```json
  { "operation_id": "deadbeef", "status": "ACCEPTED" }
  ```

**`status` 取值**

| 值 | 含义 |
|---|---|
| `SUCCEEDED` | 成功 |
| `FAILED` | 命令失败;请求是否已发出看 `failure.request_sent`,不要从 `error` 推断 |
| `REJECTED` | 门禁或容量拒绝(能力未开 / 未授权 / 身份或代不符 / 执行队列已满)|
| `ACCEPTED` | 已持久接受、尚未终结(异步写或并发幂等重提)|
| `RUNNING` | 执行中 |
| `STALE` | 迟到结果被丢弃(代已变)|
| `DEADLINE_EXCEEDED` | operation 终态超时；与 HTTP `504` 响应体中的同名状态区分 |

**`outcome` 取值**(仅写命令;读为 `null`)

| 值 | 含义 |
|---|---|
| `SUCCEEDED` | 写成功 |
| `FAILED` | 写结果已确定为失败(未发出,或微信返回明确拒绝) |
| `OUTCOME_UNKNOWN` | 已发往微信但结果未知(断线/超时),**不自动重试** |

**404**:未知账号。 **503**:容量过载,`code=COMMAND_OVERLOADED`、`retry_after_ms` 提示该命令未执行;如果返回了 `operation_id`,可查询其 `REJECTED` 终态。账号暂时无法执行命令或仍在等待旧代次退出时,同步写命令返回 `200` 且 `status=REJECTED`;已建立接受记录,并发布终态事件。提供了 `idempotency_key` 时,即使该 HTTP 响应丢失也可按键查询。**504**:本次 HTTP 等待超过 `deadline_ms`,不等于 operation 已终态(见 §4)。HTTP 层会为未提供 `request_id` 的读命令生成 id,并在成功响应、`504` 与终态 `operation` 事件中返回。504 后按 `request_id` 查询；仍为 `ACCEPTED` / `RUNNING` 时继续等待，查询到终态（包括 `DEADLINE_EXCEEDED`）后才释放账号队列。`504` 响应体中的同名 `status` 不是 operation 查询结果。

**命令就绪门禁**

`phase=online` 与命令就绪是两个不同状态；只有 `command_ready=true` 才提交业务命令。登录或恢复可能需要等待，首个代理在线事件不能替代命令就绪判断。

就绪后若会话离开 `ONLINE`（重连/长连接重建），服务推送 `command_ready=false`；恢复在线且必需同步完成后再推送 `true`。重新登录和会话接管使用新 `generation`，在新一代就绪前保持 false。状态可从 `GET /v1/accounts` 与 `GET /v1/accounts/login/{login_id}` 读取，也通过节点级和账号 SSE 推送（§6.8、§6.12）。命令在被接受前发现账号未就绪时，返回 `409 ACCOUNT_NOT_READY`，不分配 operation id、不入队、不执行、不发 operation 终态事件。若命令已被持久接受、执行前账号才变为未就绪，则不执行命令，将记录结算为 `REJECTED` 并发布关联事件；此时可能已返回异步 `202`，可按 `request_id` / `idempotency_key` 查询确认未执行。订阅/能力开关及 operation 查询属于控制面，未就绪时仍可调用。

命令控制字段:

| 字段 | 适用范围 | 说明 |
|---|---|---|
| `control.request_id` | 读命令 | 可由调用方生成,也可省略让服务自动生成;每账号应唯一。命令被持久接受后，即可按 `GET …/operations?request_id=…` 查询，即使它尚未开始执行；执行结束后可查终态，执行进程丢失时结算为 `FAILED`。HTTP `504` 后 operation 可能暂时仍为 `ACCEPTED` / `RUNNING`;按此 id 继续查询。查询到 `DEADLINE_EXCEEDED` 等终态后，状态不会再变化，可释放该账号的调用队列。 |
| `control.idempotency_key` | 写命令 | 写操作的账号内幂等键;建议每条写命令必传,以便 HTTP 响应丢失后对账。`operation` 终态事件、`GET …/operations/{operation_id}` 和按键查询均返回此字段。按键查询返回最近一次尝试,包括允许安全重试的执行前 `REJECTED` 终态。 |

按关联键查询示例:
```bash
curl -s "$BASE/v1/accounts/123456789/operations?request_id=profile-read-example-001"
curl -s "$BASE/v1/accounts/123456789/operations?idempotency_key=comment-123-once"
```
两种查询各自必须且只能提供一个参数。`200` 返回对应 operation 当前记录;未找到返回 `404`。关联键在同一账号内应保持唯一,不得并发复用。

---

### 6.7 `GET /v1/accounts/{uin}/operations/{operation_id}` — 查写操作终态

查询某写操作的**持久化终态**。用于:SSE 漏收 `operation` 事件时补齐,或服务重启后仍可查。

```bash
curl -s $BASE/v1/accounts/123456789/operations/deadbeef
```
**200**
```json
{
  "operation_id": "deadbeef",
  "uin": 123456789,
  "method": "finder.private_send",
  "status": "SUCCEEDED",
  "outcome": "SUCCEEDED",
  "error": null,
  "idempotency_key": "comment-123-once",
  "value": { "ret": 0, "ok": true,
              "client_msg_id": "<opaque-client-message-id>",
              "server_message_id": 10002 }
}
```
- `value`：已确定结果的摘要，与同步响应及 SSE `operation` 的同名字段一致。私信发送成功且已分配消息 ID 时含 `client_msg_id` / `server_message_id`，后者用于 send 与 sync 的关联。失败不能视作送达；`OUTCOME_UNKNOWN` 可能没有 `value`，不能据此推算消息 ID。
- `failure`:失败时的通用请求边界对象;与同步 `CommandResult.failure`、SSE `operation.failure` 逐字段一致。
- 只能查到属于该 `uin` 的操作(账号隔离)。
- **404**:未知操作。

**存储不可用时的查询规则**：以上 operation_id 查询及 §6.6 的 request_id / 幂等键
查询，只有确认不存在才返回 404。账号存储正在恢复或不可读时返回 **503**，不表示
原命令未接收，也不能据此重新执行写操作：

```json
{"error":"account storage is temporarily unavailable","code":"STORE_RECOVERING","retryable":true,"retry_after_ms":1000}
```

`STORE_RECOVERING` / `STORE_UNAVAILABLE` 可按提示重试查询，并带 `Retry-After: 1`。
`STORE_CORRUPT` 为 `retryable=false`、`retry_after_ms=null`，需要修复存储，响应不含
账号正文、路径或底层异常。账号会话不在线时，查询同样遵循此规则。

**超时后的结果核对**：HTTP `504` 或上游网关超时不能证明命令未执行。读命令按 `request_id` 查询，写命令按 `operation_id` / `idempotency_key` 查询，或通过关联的 SSE `operation` 事件核对；未带命令关联标识的 `profile` 数据推送不能作为该命令的执行结果。

`200` 记录包含该命令的 `idempotency_key`(若调用方有提供)。该查询在账号尚未就绪时也可调用。读命令按 `request_id` 查询,见 §6.6。

---

### 6.8 `GET /v1/accounts/{uin}/events` — 事件流(SSE)

打开该账号的 Server-Sent Events 长连,实时接收消息与事件。**这是收「普通单聊/群聊」消息的唯一通道**(本服务不存储消息)。⚠️ **视频号私信不走 SSE**,须主动调 `finder.private.sync` 拉取(见 §7.2.2①)。

**查询参数**

| 参数 | 默认 | 说明 |
|---|---|---|
| `since` | `0` | 从该 `seq` 之后开始(断线重连时用于追赶)|
| `generation` | — | 你上次见到的代。与当前代不符时,服务端会重置为全量回放新代(见 §12.5)|

```bash
curl -sN "$BASE/v1/accounts/123456789/events?since=42&generation=0"
```

**SSE 帧格式**
```
id: 43
event: message
data: {"seq":43,"kind":"message","generation":0,"payload":{"conversation":"wxid_abc","new_msg_id":10001,"msg_type":1,"from_user":"wxid_abc","to_user":"wxid_self","is_chatroom":false,"chatroom_sender":"","create_time":1700000000,"preview":"你好"}}

```
- `id` = 事件 `seq`(可作为重连游标 `since`)。
- `event` = 事件类型:`message` / `state` / `login` / `command_ready` / `data` / `operation`(§8)。
- `data` = JSON:`{seq, kind, generation, payload}`。
- 空闲时服务端周期性发送 `: keepalive` 心跳注释行。

**投递与对账**

- 消息可能重复投递，调用方按 `new_msg_id` 去重。
- 网络断开后按 `since` / `generation` 续接，保持稳定 SSE 连接，并由调用方保存已经消费的数据。
- 本服务不提供普通聊天历史消息查询接口；公开版不披露上游确认和同步游标实现。

**404**:未知账号。

---

### 6.9 `POST /v1/accounts/{uin}/subscription` — 订阅开关

开/关该账号的推送订阅(`profile` **默认 ON**,`messages`/`contacts`/`sns` **默认 OFF**;均跨重登录保持,见 §9)。订阅是**推送门控**:打开某类型只决定“该类数据是否推给你”,**不会**触发额外的主动拉取。订阅类型见 §9。

**请求体**

| 字段 | 类型 | 说明 |
|---|---|---|
| `enabled` | bool | 必填。开/关 |
| `type` | string | 可选,默认 `"messages"`。取值:`messages` / `contacts` / `profile` / `sns` |

```bash
curl -sX POST $BASE/v1/accounts/123456789/subscription \
  -H 'Content-Type: application/json' -d '{"enabled":true,"type":"messages"}'
```
**200**(返回当前全量订阅快照)
```json
{
  "uin": 123456789,
  "type": "messages",
  "enabled": true,
  "subscribe_messages": true,
  "data_subscriptions": ["contacts", "profile"]
}
```
**400**:缺 `enabled` 或未知 `type`。 **404**:未知账号。

**只读查询** `GET /v1/accounts/{uin}/subscription`(核对时无需发修改请求):
```bash
curl -s $BASE/v1/accounts/123456789/subscription
```
**200**
```json
{ "uin": 123456789, "subscribe_messages": true, "data_subscriptions": ["contacts", "profile"] }
```
**404**:未知账号。

---

### 6.10 `POST /v1/accounts/{uin}/capabilities` — 写能力开关

开/关该账号的写能力。**所有写能力默认全关**;写命令必须先开对应能力,再逐次 `authorize`(§7.2、§12.2)。

**请求体**

| 字段 | 类型 | 说明 |
|---|---|---|
| `enable` | string[] | 要开启的能力名 |
| `disable` | string[] | 要关闭的能力名 |

```bash
curl -sX POST $BASE/v1/accounts/123456789/capabilities \
  -H 'Content-Type: application/json' -d '{"enable":["message.mark_read","finder.like"]}'
```
**200**
```json
{ "uin": 123456789, "capabilities": ["finder.like", "message.mark_read"] }
```
**404**:未知账号。

**只读查询** `GET /v1/accounts/{uin}/capabilities`(核对时无需发修改请求):
```bash
curl -s $BASE/v1/accounts/123456789/capabilities
```
**200**
```json
{ "uin": 123456789, "capabilities": ["finder.like", "message.mark_read"] }
```
**404**:未知账号。

---

### 6.11 `GET /v1/proxy/events` — 代理租约事件流(SSE)

代理登录的租约生命周期事件流(阶段迁移、出口 IP 观测、心跳)。仅在使用代理登录时需要。

**查询参数 / 头**

| 参数 | 说明 |
|---|---|
| `login_id` | 可选,按登录过滤 |
| `lease_id` | 可选,按租约过滤 |
| `last_event_id`(或 HTTP 头 `Last-Event-ID`)| SSE 重连游标,原样回传服务端返回的 `id` |

```bash
curl -sN "$BASE/v1/proxy/events?lease_id=lease-001"
```
**SSE 帧**(`proxy_status`)
```
id: cursor-example:15
event: proxy_status
data: {"lease_id":"lease-001","login_id":"a1b2c3","account_id":null,"uin":123456789,"generation":0,"epoch":1,"proxy_phase":"online","event_seq":15,"heartbeat_seq":1,"label":"","ok":true,"terminal":false,"release_reason":null}
```
**SSE 帧**(`proxy_egress`)
```
id: cursor-example:16
event: proxy_egress
data: {"lease_id":"lease-001","login_id":"a1b2c3","account_id":null,"uin":123456789,"generation":0,"epoch":1,"event_seq":16,"label":"","observed_egress_ip":"198.51.100.10","region":"US","check_status":"success","error_code":null,"ts":1700000002.0}
```
- SSE 帧的 `id` 是不透明重连游标，必须完整原样回传；不要解析它，也不要将其与 JSON `data.epoch`（代理连接代次）比较。
- `event` 取值:`proxy_status`(阶段/状态)/ `proxy_egress`(出口 IP 观测)/ `resync`(游标失效,提示按 `GET …/login/{login_id}` 和解)。
- ⚠️ **注意区分两处代理数据结构不同**:
  - **本 SSE 事件**的 `data` 是**原始事件载荷**,心跳字段名为 `heartbeat_seq`,并带 `event_seq`/`epoch`;**不含** `lease_state`/`observed_egress_ip`/`region`(这些在 egress 事件或快照里)。
  - **`GET …/login/{login_id}` 的 `proxy` 快照**是**聚合后的状态快照**,字段名为 `last_heartbeat_seq`,并含 `lease_state`/`observed_egress_ip`/`region` 及最近一次出口检查状态(见 §11 表)。
- 字段含义见 §11。
- **503**:服务未启用代理事件。

---

### 6.12 `GET /v1/accounts/events` — 节点级账号事件流(SSE)

一条事件流推送当前实例账号的登录阶段、命令就绪和会话状态，帮助客户端维护账号列表。普通聊天消息使用 §6.8；额外验证的一次性材料仅通过当前登录的 §6.3.2 获取，不会出现在本流中。

**查询参数 / 头**

| 参数 | 说明 |
|---|---|
| `last_event_id`(或 HTTP 头 `Last-Event-ID`)| SSE 重连游标,原样回传服务端返回的 `id`（不透明字符串）|

```bash
curl -sN "$BASE/v1/accounts/events"
```

**SSE 帧**(`event: login` —— 登录阶段迁移)
```
id: cursor-example:12
event: login
data: {"type":"login","login_id":"a1b2c3","phase":"online","uin":123456789,"generation":0,"error":null,"ts":1700000000.0}
```
`phase` 取值同 §10(`starting`…`online`/`failed`/`aborted`);`phase="online"` 即"该账号上线",带 `uin`+`generation`。

**SSE 帧**(`event: account_state` —— 上线后的会话态迁移)
```
id: cursor-example:13
event: account_state
data: {"type":"account_state","uin":123456789,"state":"offline","reauth_required":false,"last_fault":null,"ts":1700000100.0}
```
`state` 取值同 §10(`online`/`reauth_required`/`closed`/`offline`)。上线本身由 `login`(`phase=online`)帧携带,故此帧只报**上线之后**的迁移(掉线/需重登/关闭),二者不重复。

**SSE 帧**(`event: command_ready` —— 命令就绪变化)
```
id: cursor-example:14
event: command_ready
data: {"type":"command_ready","uin":123456789,"generation":0,"command_ready":true,"command_ready_at":1700000003.25,"reason":"startup_complete","ts":1700000003.25}
```
`command_ready` 独立于 `phase`；`online` 可先到，客户端仍须等待就绪。重连时可能先变为 false，恢复后再变为 true；重新登录的新 `generation` 需要重新确认就绪。详情见 §6.6。

**SSE 帧**(`event: resync`)
```
event: resync
data: {"reconcile":"/v1/accounts"}
```
重连游标无法续接时，服务发出 `resync`；此时先用 `GET /v1/accounts` 对账，再继续消费事件。

- `id` 为不透明游标，不能用于推断进程、部署或内部序列结构。
- 空闲时周期性发送 `: keepalive` 心跳注释行。
- 慢消费者(积压)会被断开,请带 `Last-Event-ID` 重连补齐。
- 事件是**至少一次 + 尽力而为**的提示:请按完整 SSE `id` 去重,并把每条事件当作"去读 §6.4 权威列表"的触发信号,而非唯一真相。
- **503**：账号事件流不可用。

---

## 7. 命令集

视频号、朋友圈和通讯录能力均通过 `POST /v1/accounts/{uin}/commands` 的 `method` 选择。读命令同步返回结果；写命令默认关闭，须开启对应能力并逐次授权。所有命令均须遵守 §12 的账号串行、幂等和终态核对规则。

本章公开方法、主要字段和调用边界。可选功能、支持范围以及未公开的上下文或媒体处理说明，须向服务提供方取得授权对接资料。示例全部为合成值或明显占位值，不能直接用于调用。凡标注为不透明（opaque）的值，必须从对应接口取得并原样传递，不得解析、猜测、拼接或跨账号复用。

### 7.0 功能 → 接口(命令)对照总览

| 功能 | method | 类型 | 说明 |
|---|---|---|---|
| 推荐信息流 | `finder.feed` | 读 | §7.1.3；`cursor_hex` 翻页 |
| 进入推荐页 / 离开视频号 | `finder.navigation` | 本地导航 | §7.1.0；须绑定当前 `generation` |
| 搜索 | `finder.search` | 读 | §7.1.1；分类、排序及 `page_token` 翻页 |
| 关注列表 | `finder.follow_list` | 读 | §7.1.3；`contacts[]`、`cursor_hex` |
| 创作者主页与作品 | `finder.creator` | 读 | §7.1.3；`contact`、`objects[]` |
| 查询关注态 | `finder.follow_status` | 读 | §7.1.3；也可读作品的 `author_follow_flag` |
| 作品详情 | `finder.object` | 读 | §7.1.3；`object`、`comments[]` |
| 搜索作品详情 | `finder.open_search_object` | 读 | §7.1.1；使用同一搜索项的返回字段 |
| 解析搜索作品引用 | `finder.resolve_object` | 读 | §7.1；返回作品标识 |
| 评论翻页 | `finder.comments` | 读 | §7.1.3 |
| 展开评论回复 | `finder.expand_replies` | 读 | §7.1.3.2 |
| 关注 / 取消关注 | `finder.follow` / `finder.unfollow` | 写 | §7.2.6 |
| 点赞作品 / 取消 | `finder.heart` / `finder.unheart` | 写 | §7.2.4 |
| 喜欢作品 / 取消 | `finder.like` / `finder.unlike` | 写 | §7.2.4；与作品点赞不同 |
| 发评论 / 回复 | `finder.comment` / `finder.reply` | 写 | §7.2.5 |
| 点赞评论 / 取消 | `finder.comment_heart` / `finder.comment_unheart` | 写 | §7.2.7 |
| 网页扫码授权预览 / 确认 | `finder.scan_login_preview` / `finder.scan_login_confirm` | 写 | §7.2.8；两者均需写能力与授权 |
| 建立私信会话 | `finder.open_session` | 写 | §7.2.1；可能创建外部会话 |
| 发私信 | `finder.private_send` | 写 | §7.2.1 |
| 私信会话与消息同步 | `finder.private.sync` | 读 | §7.1.3、§7.2.2；不通过 SSE 推送私信消息 |
| 朋友圈时间线 | `sns.timeline` | 读 | §7.1.3 |
| 通讯录 | `contact.list` | 读 | §7.1.5；增量使用 `contacts` 订阅 |
| 本账号资料 | `account.profile` | 读 | §7.1.3；另有登录时一次性 `profile` 订阅 |
| 普通聊天消息 | `messages` 订阅及账号 SSE | 订阅 | §6.8、§8.1 |
| 会话已读上报 | `message.mark_read` | 写 | §7.2 |

```bash
curl -sX POST "$BASE/v1/accounts/$UIN/commands" -H 'Content-Type: application/json' \
  -d '{"method":"finder.feed","params":{}}'
curl -sX POST "$BASE/v1/accounts/$UIN/commands" -H 'Content-Type: application/json' \
  -d '{"method":"finder.follow_list","params":{}}'
```

### 7.1 读命令

| method | 主要 params |
|---|---|
| `finder.search` | `query` 必填；`category` 为 `all/video/account/goods/live`，默认 `all`；`sort` 可选 `latest/hot`；`page_token` 续页 |
| `finder.feed` | `cursor_hex`：上页返回值，原样续页 |
| `finder.navigation` | `model_id`、`event`、`event_id`；顶层 `account_generation` 必填，见 §7.1.0 |
| `finder.follow_list` | `finder_username` 可省略或为空，表示本账号；否则为目标创作者；`cursor_hex`、`scene` 可选 |
| `finder.creator` | `finder_username` 必填 |
| `finder.object` | `object_id` 必填；`finder_username` 按读结果回填 |
| `finder.open_search_object` | 同一 `finder.search` item 的 `export_id`、`object_id`、`from_session_id`；建议同时回传 `search_query/document_id/hash_doc_id` |
| `finder.resolve_object` | 同一搜索项的 `export_id`、`object_id` 必填；返回 `{ret,ok,object_id,object_id_str,v2_name,nickname,objects[]}` |
| `finder.comments` | `object_id` 必填；`finder_username`、`max_comment_id`、`cursor_hex` 按需回填 |
| `finder.expand_replies` | `comment_id`、`object_id` 必填；其余上下文见 §7.1.3.2 |
| `finder.private.sync` | `self_chat_username` 必填，使用本账号 wxid |
| `contact.list` | 无必填业务参数 |
| `account.profile` | 无必填业务参数 |
| `sns.timeline` | `limit`、`cursor`、`first_page_md5` 按需传入 |
| `finder.follow_status` | `finder_usernames` 数组（最多 20 个）或单个 `finder_username` |

通用分页及上下文字段包括 `limit`、`cursor`、`offset`、`scene`、`max_id`、`first_page_md5`、`finder_username`、`object_id`、`export_id`、`search_query`、`document_id`、`hash_doc_id`、`from_session_id`、`max_comment_id`、`cursor_hex`、`flag`、`self_chat_username`。仅按相应命令使用；不透明值从实际响应原样回填。

搜索详情须使用对应搜索项的完整引用，缺少必要引用时会失败。搜索读取不要求先进入推荐页，也不会授予任何写操作权限。上下文无效或与当前账号、代次不匹配时拒绝。遇 `request_sent=false,retryable=false` 应修正前置条件，不以相同参数反复重试。具体支持范围由服务提供方确认。

#### 7.1.0 `finder.navigation`：显式导航与会话边界

用户实际进入可见推荐页时提交 `enter_recommend_feed`，真正离开视频号交互链路时提交 `leave`。空列表不影响进入事件。后台预加载、隐藏组件挂载和其他查询不能代替真实页面进入。

`model_id` 为提供方分配的不透明标识；支持值须从授权对接资料取得。

```json
{
  "method": "finder.navigation",
  "account_generation": 1,
  "params": {
    "model_id": "<PROVIDER_MODEL_ID>",
    "event": "enter_recommend_feed",
    "event_id": "example-enter-001"
  }
}
```

- `account_generation` 必须为当前实际观测到的正整数代次，不能将旧页面事件改标成新代次。
- `event` 仅支持 `enter_recommend_feed`、`leave`。`event_id` 长度为 1–128，仅含 ASCII 字母、数字及 `._:-`；重传同一事件必须复用 ID。
- 命令不拉取业务内容，不改变关注关系，也不打开写能力。依赖导航的后续交互须等待成功。

成功 `value` 示例：

```json
{
  "model_id": "<PROVIDER_MODEL_ID>",
  "event": "enter_recommend_feed",
  "event_id": "example-enter-001",
  "generation": 1,
  "active_entry": "recommend_feed",
  "navigation_revision": 1,
  "request_sent": false
}
```

`leave` 的 `active_entry` 为 `null`。同代、同 ID、同参数返回首次结果；同 ID 参数不同则失败。前端须按 UI 生命周期与已确认 `navigation_revision` 丢弃迟到响应，不能用旧进入响应覆盖新离开状态。会话恢复或重新登录会清除导航；新代不得沿用旧页面就绪标记。

同账号、同代次的导航事件必须串行，`leave` 排在已提交的进入事件完成之后。离页后不得重传未确认的进入事件。无游标的 `finder.feed` 按显式打开推荐流处理；翻页保持既有导航。其他读取不能冒充进入事件。直接入口支持范围须由提供方确认；不能暗中拉取推荐流或伪造页面事件。

失败码包括 `FINDER_NAVIGATION_INPUT_INVALID`、`FINDER_NAVIGATION_MODEL_UNSUPPORTED`、`FINDER_NAVIGATION_EVENT_CONFLICT`、`FINDER_NAVIGATION_REPLAY_CAPACITY`、`FINDER_NAVIGATION_STATE_UNAVAILABLE`、`FINDER_NAVIGATION_GENERATION_MISMATCH`，均有 `failure.request_sent=false`。容量错误不得通过自动重登或写请求处理；离开事件仍可能已清除活动导航，不能凭错误重放旧进入事件。

#### 7.1.1 `finder.search` 返回结构(含分类字段)

`items[]` 按 `item_type` 分组展示。`document_id`、`hash_doc_id`、`export_id`、`from_session_id`、`page_token` 均是不透明引用，禁止自行构造。

以下为结构示意；带尖括号的标识或链接均须替换为实际返回值。

```json
{
  "ret": 0,
  "ok": true,
  "query": "示例关键词",
  "has_more": true,
  "page_token": "<PAGE_TOKEN>",
  "items": [{
    "title": "示例作品",
    "description": "示例说明",
    "username": "",
    "source_title": "示例作者",
    "document_id": "<DOCUMENT_ID>",
    "hash_doc_id": "<HASH_DOC_ID>",
    "export_id": "<EXPORT_ID>",
    "object_id": "1001",
    "from_session_id": "<FROM_SESSION_ID>",
    "image_url": "<IMAGE_URL>",
    "duration": 10,
    "publish_time": 1700000000,
    "box_type": null,
    "box_real_type": null,
    "box_subtype": null,
    "box_title": "示例分类",
    "item_type": "object",
    "stats": {
      "like_cnt": 1, "thumb_cnt": 2, "forward_cnt": 3,
      "comment_cnt": 4, "duration": 10, "upload_time": 1700000000, "friend_like": 0
    }
  }]
}
```

| `item_type` | 含义 |
|---|---|
| `suggest` | 分类、热点或相关搜索提示 |
| `account` | 账号项，带可定位的 `username` |
| `object` | 视频或直播作品项 |
| `goods` | 公众号或商品项 |
| `unknown` | 尚未识别的类型 |

`box_type`、`box_real_type`、`box_subtype` 为整数或 `null`，`box_title` 为字符串；它们是可选原始分类元数据，保留供诊断；其取值映射未在公开版列出。优先使用 `item_type`，未知分类应保留，不自行猜测类型。

作品项的 `username` 为空；`source_title` 只是显示昵称，不能作为身份主键。使用同一项的 `export_id`、`object_id`、`from_session_id` 调用 `finder.open_search_object`，再从返回作品的 `username/author_username` 取得作者身份。不要将搜索项的 `object_id` 直接传给 `finder.object`。

账号项带 `username`、`document_id`、`image_url` 和 `description`，作品专用字段为空：

```json
{
  "title": "示例账号", "description": "示例简介",
  "username": "v2_example_creator@finder", "source_title": "",
  "document_id": "<ACCOUNT_DOCUMENT_ID>", "hash_doc_id": "", "export_id": "", "object_id": "",
  "image_url": "<AVATAR_URL>", "duration": "", "publish_time": 0,
  "box_type": null, "box_real_type": null, "box_subtype": null, "box_title": "",
  "item_type": "account", "stats": {}
}
```

账号项可直接用于 `finder.creator` 或 `finder.follow_list`；关注和私信仍须满足对应写操作前置。

分页与详情规则：

- 首次传 `query`、`category`，`sort` 可为 `latest` 或 `hot`；省略使用默认序。续页仍传相同 `query`，原样回传 `page_token`，分类和排序由续页结果保持。更换排序应重新搜索首页。
- `has_more=false` 时停止；没有更多结果时不再返回 `page_token`。`page_token_expired=true` 表示续页未被采用，响应已回到首页，须按首页重建消费状态。
- 不应假定上游游标永久有效。续页失败仅在 `failure.request_sent=false` 时按重试建议处理；`true/null` 时不要以原参数自动重发。
- `finder.open_search_object` 返回详情对象，不内联首页评论。评论须使用详情返回的 `object.id/id_str` 另调 `finder.comments`；不要沿用搜索项的对象引用替代它。

##### 7.1.2 视频号对象结构 FinderObject(feed / object / creator / comments 共用)

`finder.feed.objects[]`、`finder.object/finder.comments.object`、`finder.creator.objects[]` 使用下列共用结构。不透明上下文只供原样回填。

```json
{
  "id": 1001, "id_str": "1001",
  "username": "v2_example_creator@finder", "nickname": "示例作者",
  "description": "示例作品", "create_time": 1700000000,
  "like_count": 1, "comment_count": 2, "forward_count": 3,
  "read_count": null, "fav_count": 4, "like_flag": 0, "fav_flag": 0,
  "object_type": 0,
  "object_nonce_id": "<OBJECT_NONCE_ID>", "session_buffer": "<SESSION_BUFFER>",
  "media": [{
    "url": "<MEDIA_URL>", "thumb_url": "<THUMB_URL>", "cover_url": "<COVER_URL>",
    "media_type": 4, "width": 720, "height": 1280, "video_play_len": 10,
    "decode_key": "<OPAQUE_MEDIA_VALUE>", "encrypted_prefix_bytes": "<RETURNED_LENGTH>",
    "file_size": 1000000, "md5_sum": "<CHECKSUM>",
    "spec": [{"coding_format":"h264","bit_rate":1000000,"first_load_bytes":"<RETURNED_LENGTH>","width":720,"height":1280}]
  }],
  "author_follow_flag": 0, "author_username": "v2_example_creator@finder",
  "author_nickname": "示例作者", "author_head_url": "<AVATAR_URL>", "readable": true
}
```

长度占位值仅用于省略具体示例值，实际字段类型见 §7.1.4.1。作品、评论和消息标识可能超过 JavaScript 安全整数范围，优先使用接口返回的字符串字段或无损十进制字符串，不能经浮点数转换。

`like_count/comment_count/forward_count/fav_count` 用于相应统计展示；搜索项另有 `stats`。`read_count` 仅本账号自己的作品可能返回真实播放数；其他作品为 `null`，不能展示成零播放。`author_follow_flag` 为 `1/0/null`，`null` 表示没有提供，不能推断关注态。

关注、点赞、评论等操作须使用本账号刚读到的对应作品对象。`object_nonce_id`、`session_buffer` 等不透明值不能跨作品、账号或代次替换。首次发私信须先获得 `finder.open_session` 的返回值。

#### 7.1.3 各视频号读命令返回结构

以下为主要公开结构；可选扩展字段及未公开的编码说明须向提供方取得。

**FinderContact**（用于主页 `contact` 与关注列表 `contacts[]`）：

```json
{"username":"v2_example_creator@finder","nickname":"示例作者","signature":"","head_url":"<AVATAR_URL>","follow_flag":0,"fans_count":0,"feed_count":0}
```

**FinderComment**（评论及 `replies[]` 共用）：

```json
{
  "comment_id": 2001, "comment_id_str": "2001",
  "username": "<COMMENTER_ID>", "nickname": "示例评论者", "content": "示例评论", "create_time": 1700000000,
  "like_count": 0, "head_url": "<AVATAR_URL>", "region": "",
  "finder_username": "", "has_finder_account": false, "is_author": false,
  "reply_comment_id": 0, "reply_count": 1, "reply_has_more": true, "reply_cursor_hex": "<REPLY_CURSOR>",
  "replies": []
}
```

`is_author` 直接使用返回值，不按昵称或 username 推断。`region` 未提供时为空串。首页只内联部分回复，`reply_count` 为总数，`reply_has_more=true` 时可继续展开。

**`finder.creator`**：

```json
{
  "ret":0,"ok":true,
  "requested_username":"v2_example_creator@finder","canonical_username":"v1_example_creator@finder",
  "contact":{},"feeds_count":0,"fans_count":0,"is_private":false,
  "has_more":true,"next_max_id":1001,"next_max_id_str":"1001","cursor_hex":"<CURSOR>","objects":[]
}
```

`contact` 为 FinderContact，作品为 FinderObject。续页原样回传 `cursor_hex` 和 `max_id=next_max_id_str`；不要将 64 位 ID 转成浮点数。`requested_username` 与 `canonical_username` 的用法见 §7.1.3.1。

**`finder.object` / `finder.comments`**：

```json
{
  "ret":0,"ok":true,"object":{},"comment_count":0,
  "has_more":false,"next_max_comment_id":0,"next_max_comment_id_str":"0","cursor_hex":"","comments":[]
}
```

`object` 为 FinderObject 或 `null`。评论续页同时回传 `cursor_hex` 和 `max_comment_id=next_max_comment_id_str`。详情可读不表示评论页已读，评论请求须单独检查结果。

某些可选详情能力可能附带 `route`、`success_policy`，这两个字段是不透明标识，其取值、适用方法和成功判断要求见授权对接资料。公开版不列出匹配规则；不能仅按 `ret` 是否为零判断这些可选结果，也不能将未知策略当作成功。旧操作终态不会自动改写，应重新读取取得新结果。可选能力尚未完成端到端验证时，须由提供方明确告知后再启用。

##### 7.1.3.2 `finder.expand_replies` 展开某条评论的回复(子评论)分页

用于展开顶层评论的二级回复。作品标识取自已读取的作品；搜索结果应先通过 `finder.resolve_object` 或 `finder.open_search_object` 获得可用作品标识。

| 参数 | 要求 |
|---|---|
| `comment_id` | 必填，顶层评论 ID；可用别名 `root_comment_id`，优先使用 `comment_id_str` |
| `object_id` | 必填，作品的 `id/id_str` |
| `reply_cursor_hex` | 首屏可省略或为空；续页回传上次的 `cursor_hex` |
| `object_nonce_id` | 从当前作品读结果原样回填的不透明上下文 |
| `session_buffer` | 从同一作品读结果原样回填的不透明上下文 |
| `finder_username` | 按读结果回填作者标识；兼容路径建议提供 |
| `comment_scene` | 可选；支持值与兼容要求见授权对接资料 |
| `ref_comment_id` | 可选，定位回复锚点；默认 `0` 表示线程顶部 |

提供完整作品上下文时优先使用对应能力；缺少部分上下文时是否支持兼容读取，由提供方确认。`route` 仅作为不透明响应字段，不得据其猜测未授权能力。

```json
{
  "method":"finder.expand_replies",
  "params":{
    "comment_id":"2001","object_id":"1001","reply_cursor_hex":"<REPLY_CURSOR>",
    "object_nonce_id":"<OBJECT_NONCE_ID>","session_buffer":"<SESSION_BUFFER>","finder_username":"v2_example_creator@finder"
  }
}
```

返回：

```json
{"ret":0,"ok":true,"route":"<OPAQUE_ROUTE>","has_more":true,"cursor_hex":"<NEXT_CURSOR>","request_id":0,"replies":[]}
```

`replies[]` 为 FinderComment，每条回复的 `reply_comment_id` 指向被回复评论。续页将响应 `cursor_hex` 回传到 `reply_cursor_hex`，其余定位字段保持一致；`has_more=false` 时停止。空首屏游标不表示没有回复。`comment_id/object_id` 缺任一即拒绝，且必须按无损大整数处理。旧字段 `expand_type/expanded_count/sub_comment_total/recommend_context` 虽可能被接受，但已忽略，不能用于分页。

**`finder.follow_list`**：

```json
{"ret":0,"ok":true,"follow_count":0,"has_more":false,"cursor_hex":"","generation":1,"contacts":[]}
```

`contacts[]` 为 FinderContact。省略 `finder_username` 查询本账号；成功且为空表示结果确实为空，不能当作身份未就绪。续页原样回传 `cursor_hex`。

##### 7.1.3.1 视频号身份归一化

`finder.creator` 的 `requested_username` 回显请求，`canonical_username` 和 `contact.username` 为返回的规范身份；二者可能不同。应保存返回的映射，不按昵称合并或自行转换标识。

| 使用场景 | 标识来源 |
|---|---|
| 关注 / 取关 | 本次读到的作品 `username` 与对应 `contact`；连同作品上下文一致回填 |
| 查询关注态、建立私信会话 | 优先使用 `canonical_username` |
| 资料主键与别名 | 使用规范身份，保存 `requested_username` 到规范身份的映射 |

`finder.open_session` 可接受服务支持的 `v1_…@finder` 或 `v2_…@finder` 别名；同一账号和目标的会话标识可复用，但不得跨账号复用。`session_id` 为不透明值，不推算其生成方式。建立会话属于写操作，不能为确认身份反复调用。

**`finder.private.sync`**：

```json
{
  "ret":0,"ok":true,"rounds":1,"completed":true,"store_epoch":"<STORE_EPOCH>",
  "conversations":[{
    "session_id":"<SESSION_ID>","finder_username":"v2_example_creator@finder","message_count":1,
    "latest_create_time":1700000000,"latest_content":"示例消息","latest_is_sender":false,
    "messages":[{
      "server_message_id":3001,"client_msg_id":"","create_time":1700000000,"is_sender":false,
      "from_user":"v2_example_creator@finder","to_user":"<RECIPIENT_ID>","msg_type":1,
      "content":"示例消息","status":0,"local_message_id":1
    }]
  }]
}
```

`messages[]` 按 `(create_time,local_message_id)` 升序。方向以 `is_sender` 为准。`server_message_id` 用于去重与发送对账；`client_msg_id` 可能为空，不能自行补算。`store_epoch` 为不透明的累计结果边界，见 §7.2.3。时间为 Unix 秒。

失败时检查 `value.sync_failure`，不要解析错误文本：

```json
{"status":"FAILED","value":{"ok":false,"sync_failure":{"code":"PROXY_CONNECT_FAILED","stage":"proxy_connect","messages_pulled":false,"guaranteed_no_messages_pulled":true,"gap":false,"send_blocked":false,"received_items":0,"retryable":true}}}
```

- `messages_pulled` 为 `true/false/null`；仅 `guaranteed_no_messages_pulled=true` 代表本次确定未取回消息。
- `gap=false` 仅描述本次调用，不证明历史完整。`received_items` 只统计本次，零值不表示没有历史消息。
- 只有 `send_blocked=false` 才能解除本地发送阻挡。一次明确未取消息的失败不会消除旧的未完成或不确定状态。
- `SYNC_SESSION_REJECTED` 表示认证被拒绝，即使没有取回消息也不能解锁发送。`SYNC_UNRESOLVED_ITEMS/SYNC_INCOMPLETE` 表示尚未完成，`SYNC_SNAPSHOT_FAILED` 表示结果读取失败。遵循实际 `sync_failure` 和认证状态处理，不能用单个错误码代替全部字段。

`from_user/to_user` 分别是发送方和接收方，可能为空或为不透明会话身份。入站消息的 `from_user` 可用于识别对端创作者；出站消息的 `from_user` 对应本账号。优先使用返回的 `conversation.finder_username`，缺失时按 §7.2.3 处理。响应不含未读计数；媒体消息只有类型化占位或 XML，不提供下载。

**`finder.follow_status`**：

```json
{"statuses":[{"finder_username":"v2_example_creator@finder","follow_flag":0,"nickname":"示例作者","ok":true}]}
```

单项失败时该项为 `follow_flag:null,ok:false,error:"…"`，其他项仍可使用。

**`finder.open_session`**：

```json
{"ret":0,"ok":true,"session_id":"<SESSION_ID>","finder_username":"v2_example_creator@finder","real_chat_username":"wxid_example_self","self_chat_username":"wxid_example_self","my_account_type":2}
```

先开启对应 capability，每次请求带 `control.authorize=true` 和唯一 `idempotency_key`。返回值可原样作为 `finder.private_send.session`，多出的 `ret/ok` 被忽略。`real_chat_username/self_chat_username` 使用本账号身份，不自行替换。上下文、账号或代次不匹配时在发送前拒绝；它不授予其他写权限。可选直接入口的支持与验证状态须向提供方确认，不能将权限拒绝提示为对端身份未知。

**`account.profile`**：

```json
{"ret":0,"ok":true,"profile":{"user_info":{"userName":"wxid_example_self","nickName":"示例昵称","alias":"example_alias","sex":0,"signature":"","city":"","province":"","country":""},"user_info_ext":{"bigHeadImgUrl":"<AVATAR_URL>","smallHeadImgUrl":"<AVATAR_URL>"}}}
```

`userName/nickName/alias` 分别为 wxid、昵称、微信号；头像位于 `user_info_ext`。未知扩展可能保留为 `field_<n>`。每次调用实时读取，不缓存，避免高频轮询。无需依赖登录时一次性订阅来按需取得资料。

**`finder.feed`**：`{ret,ok,object_count,has_more,cursor_hex,ret_flag,need_clear,objects:[FinderObject],object_load:{emitted,objects}}`。翻页回传 `cursor_hex`。

**`sns.timeline`**：`{ret,ok,object_count,objects_seen,first_page_md5,objects_b64:[]}`。`objects_b64` 是不透明编码对象；其授权解码说明须向提供方取得。

## 7.2 写命令

写命令须开启对应 capability、逐次带 `control.authorize=true`，且会话可用。建议每条写命令使用唯一 `control.idempotency_key`；必须等持久 operation 终态后再提交下一条。写操作不自动重试，`OUTCOME_UNKNOWN` 不等同于失败或未发送。

| method | capability | 主要 params / 限制 |
|---|---|---|
| `message.mark_read` | `message.mark_read` | `to_user_name`、整数 `read_seq` 必填；`unread_chats` 可选 |
| `oplog.send` | `oplog` | `ops:[{cmd_id,cmd_buf_hex}]`；具体操作与不透明内容仅在取得授权说明后使用 |
| `finder.follow` | `finder.follow` | `contact`、`ref_object`，见 §7.2.6 |
| `finder.unfollow` | `finder.unfollow` | 同上，目标须已关注 |
| `finder.heart` | `finder.heart` | 当前作品 `object`；`scene` 默认 `0` |
| `finder.unheart` | `finder.unheart` | 同上，`object.like_flag=1` |
| `finder.like` | `finder.like` | 当前作品 `object`、`finder_username`；`scene` 默认 `20` |
| `finder.unlike` | `finder.unlike` | 同上，`object.fav_flag=1` |
| `finder.comment` | `finder.comment` | `object`、`content`；`client_comment_id` 可选，见 §7.2.5 |
| `finder.reply` | `finder.reply` | 同上，加 `parent_comment`；`root_comment_id` 可选 |
| `finder.comment_heart` | `finder.comment_heart` | `object`、`comment_id`，见 §7.2.7 |
| `finder.comment_unheart` | `finder.comment_unheart` | 同上 |
| `finder.open_session` | `finder.open_session` | `finder_username` 必填；可能创建外部会话 |
| `finder.private_send` | `finder.private_send` | `content`、原样回填的 `session`；`continuation_mode` 可选，见 §7.2.1 |
| `finder.private_send.clear_limit` | `finder.private_send.clear_limit` | 可选管理功能，`blocked_day`、`reason`，见 §7.2.1 |
| `finder.scan_login_preview` | `finder.scan_login_preview` | 新鲜 `connect_url` 必填；虽为预览也需要写能力与授权 |
| `finder.scan_login_confirm` | `finder.scan_login_confirm` | 新鲜 `connect_url`；`scope` 仅支持 `snsapi_login` 或省略 |
| `message.send` | `message.send` | 当前不可用，调用被拒绝 |

对象类参数必须取自本账号实际读结果，不能跨作品、账号、代次复用或自行补造。不透明上下文的结构不属于公开契约。

##### 7.2.4 视频号作品点赞与喜欢

| 效果 | method | 返回对象计数 / 状态 |
|---|---|---|
| 作品点赞 / 取消 | `finder.heart` / `finder.unheart` | `like_count/like_flag` |
| 喜欢作品 / 取消 | `finder.like` / `finder.unlike` | `fav_count/fav_flag` |

`object` 须包含本次读结果的 `id/username/object_nonce_id/session_buffer`。取消操作还须携带已确认的 `like_flag=1` 或 `fav_flag=1`；状态不满足会失败。

```json
{
  "method":"finder.heart",
  "params":{"object":{"id":"1001","username":"v2_example_creator@finder","object_nonce_id":"<OBJECT_NONCE_ID>","session_buffer":"<SESSION_BUFFER>","object_type":0},"scene":0},
  "control":{"authorize":true,"idempotency_key":"example-heart-001"}
}
```

可选字段为 `client_report_buff`、`play_json`、`session_enter_ms`、`delivery_ts_ms`、`exposure_ts_ms`、`exposure_seq`、`feed_id`。只按授权对接说明透传与本次交互匹配的实际返回值，不解析不透明内容，也不自行生成上下文；不需要时省略。公开版不披露这些值的内部生成规则。

返回 `value` 以实际命令结果为准；成功后对应状态和计数可从后续读取核对。两类能力分别开启，不能混用。

##### 7.2.5 视频号发评论 / 回复评论(`finder.comment` / `finder.reply`)

`finder.comment` 参数：

- `object` 必填，原样回填本账号刚读到的目标作品，至少含 `id/session_buffer`；`object_nonce_id` 按返回值携带。
- `content` 必填，评论文本。
- `client_comment_id` 可选，不传由服务处理；它只用于关联，真实评论 ID 以返回结果为准。
- 搜索作品可回填本次 `search_query`，不得伪造来源。

作品来自推荐、主页或直接详情时，使用对应读结果；来自搜索时，先使用 `finder.open_search_object`。不要借用另一作品的 `session_buffer`。对旧作品操作前应重新读取，不能长期保存后回放上下文。

`finder.reply` 使用相同作品与文本参数，并加 `parent_comment:{comment_id,finder_username?,username?}`；目标评论至少提供 `finder_username` 或 `username` 中一个，均取自实际评论结果。回复顶层评论时 `root_comment_id` 默认使用该评论 ID；回复楼中楼时须回填所属根评论 ID。所有 ID 按无损大整数处理。

成功 `value`：`{ret:0,ok:true,comment_id,client_comment_id}`。评论和回复对外可见，默认关闭，须分别开启能力、逐次授权并等待终态。

```json
{
  "method":"finder.reply",
  "params":{"object":{"id":"1001","session_buffer":"<SESSION_BUFFER>"},"content":"示例回复","parent_comment":{"comment_id":"2001","username":"<COMMENTER_ID>"}},
  "control":{"authorize":true,"idempotency_key":"example-reply-001"}
}
```

##### 7.2.6 视频号关注 / 取消关注(`finder.follow` / `finder.unfollow`)

- `contact:{username,nickname?,follow_flag}` 取自当前账号读到的规范身份及已确认关注态。`follow_flag` 必须为整数 `0/1`；关注传当前 `0`，取关传当前 `1`。旧关注调用省略时兼容为 `0`；取关不能省略已知状态。
- `ref_object:{id,username,object_nonce_id,session_buffer,object_type}` 原样回填本次读到的对应作品。
- `ref_object.delivery_buffer/ref_object.base_provenance_hex/session_enter_ms` 为可选不透明补充上下文，只能来自实际返回值或提供方授权资料要求的输入；不能借此绕过校验。
- 不要求为关注操作强制先打开推荐页。直接入口的可用范围由提供方确认；上下文、账号或代次不匹配时仍拒绝，且不授予其它写权限。
- 不满足前置条件时为 `FAILED`，`failure.code=FINDER_FOLLOW_BASE_FAILED`、`stage=local`、`request_sent=false`；不能提示成账号掉线或自动重试。
- 返回 `value:{ret,ok,contact,live_status_flag}`；成功要求 `ret=0` 且 `contact.follow_flag` 为目标状态。

相关可选兼容能力尚未完成端到端验证时，应由提供方明确确认后使用。

##### 7.2.7 视频号点赞评论 / 取消(`finder.comment_heart` / `finder.comment_unheart`)

必填 `object` 为评论所在作品的本次读结果，含 `id/object_nonce_id/session_buffer`；`comment_id` 为目标评论 ID，优先使用 `comment_id_str`。`scene` 可选，默认 `0`。只回填真实不透明上下文，不自行补造。

返回 `value:{ret,ok}`；`ret=0` 且 `ok=true` 表示操作成功，以 operation 终态及其 `value` 为准。取消使用 `finder.comment_unheart`，参数相同，须单独开启对应能力。操作对评论作者可见，必须逐次授权。旧作品应重新读取后再操作。

```json
{"method":"finder.comment_heart","params":{"object":{"id":"1001","object_nonce_id":"<OBJECT_NONCE_ID>","session_buffer":"<SESSION_BUFFER>"},"comment_id":"2001"},"control":{"authorize":true,"idempotency_key":"example-comment-heart-001"}}
```

##### 7.2.8 扫码登录授权(`finder.scan_login_preview` / `finder.scan_login_confirm`)

为微信开放平台 `snsapi_login` 网页登录提供预览和确认。调用方须从目标网页二维码取得有效 `connect_url`；本接口不解码二维码图片。

1. 可先调用 `finder.scan_login_preview`，向用户展示请求应用和 scope；它不确认网页登录，但仍须写能力与逐次授权。
2. 由用户确认后调用 `finder.scan_login_confirm`，这会授权目标网页登录本账号。

`connect_url` 必须为 `https://open.weixin.qq.com/connect/confirm?uuid=<QR_UUID>`，uuid 非空、单次有效且有时效。过期时从目标网页重新取得，不能缓存回放。确认成功后再次确认会被拒绝。`scope` 只能为 `snsapi_login` 或省略。

| method | `value` 字段 |
|---|---|
| `finder.scan_login_preview` | `ret,ok,scan_ret,authorize_ret,appid,app_name,scope,user_nickname` |
| `finder.scan_login_confirm` | `ret,ok,confirmed,scan_ret,authorize_ret,appid,app_name,consent_scope,scope_sent` |

预览成功要求相关前置返回成功；重复授权时部分应用信息可能为空。确认的 `ok=true` 要求整次流程成功；前置失败时中止，`confirmed=false`。确认未发送时 `scope_sent=null`，对应未发送的结果可为 `ret=null`。本地 URL、scope 或已消费状态校验失败按未发送失败处理，不作结果未知。

同一 `connect_url` 可按预览后确认的流程使用。用户必须看到并确认目标应用与 scope；禁止把确认操作当作后台读取。未开能力或未逐次授权时返回 `REJECTED`。不保证支持 gamecenter、头像选择、人脸核身、H5、异常确认等额外授权流程，支持范围须向提供方确认。

```json
{
  "method":"finder.scan_login_preview",
  "params":{"connect_url":"https://open.weixin.qq.com/connect/confirm?uuid=<QR_UUID>"},
  "control":{"authorize":true,"idempotency_key":"example-preview-001"}
}
```

确认时将 method 换为 `finder.scan_login_confirm`，沿用当次 URL，并使用该确认操作自己的唯一幂等键。示意结果：

```json
{"ret":0,"ok":true,"confirmed":true,"appid":"<APP_ID>","app_name":"示例应用","consent_scope":"snsapi_login","scope_sent":"snsapi_login"}
```

#### 7.1.4 图片 / 视频 URL 使用说明

`media.url/thumb_url/cover_url` 和搜索 `image_url` 可能包含限时访问票据，应使用完整返回值，避免写入公开日志或公开仓库。不要长期缓存带票据 URL；过期后重新读取获取。头像相对稳定，也可能随用户修改而变化。

图片可按返回 URL 加载；加密视频不能直接当作普通视频文件播放。媒体 URL 的有效性、下载权限以及授权处理方式由服务提供方说明。本服务不提供通用取图或取流代理端点。

#### 7.1.4.1 加密媒体处理

公开版仅列出可见字段，不披露加密媒体处理算法、参数生成方式或示例密钥。需要播放或下载加密媒体时，须向服务提供方取得授权处理说明；仅凭本文不能完成此能力的对接。

| `media[]` 字段 | 公开说明 |
|---|---|
| `decode_key` | 不透明媒体处理值，以原始字符串保存，禁止转为浮点数或自行推算；不应公开记录 |
| `encrypted_prefix_bytes` | 返回的整数长度参数，按授权说明使用，不在公开版列出固定值 |
| `file_size` | 文件总字节数，`0` 表示未知 |
| `md5_sum` | 可选完整性校验值，按授权说明校验 |
| `spec[]` | 规格列表，可含 `coding_format/bit_rate/width/height/first_load_bytes`；各字段用途以授权说明为准 |

字段为空或缺失时不自行补值。分段下载、拖动播放和处理后校验均须遵循授权资料，不能根据字段名猜测处理规则。

#### 7.1.5 `contact.list` 通讯录返回结构(完整列表 + 增量契约)

```json
{
  "full":true,"rounds":1,"records":2,"count":2,
  "contacts":[{"user_name":"wxid_example_contact","nick_name":"示例联系人","remark":"","alias":"","sex":0,"signature":"","province":"","city":"","contact_type":0}],
  "chatrooms":[{"user_name":"example_room@chatroom","nick_name":"示例群","remark":"","alias":"","sex":0,"signature":"","province":"","city":"","contact_type":0}],
  "strangers":[{"type":"stranger","user_name":"<STRANGER_ID>","remark":""}],
  "generation":1,"other_records":{}
}
```

`records` 保留为原始记录条数，`count` 为联系人与群条数；`rounds` 可作为辅助值。`contacts/chatrooms/strangers` 分别为联系人、群和陌生记录；未提供的可选文本字段为空串。群成员列表不在此结构中。

`full=true` 表示完整快照，可用于重建名单。不要仅凭 `records/count=0` 丢弃此前资料，应结合完整性和快照状态。`contacts` 订阅的 SSE `data` 使用同一结构，新增和修改按 `user_name` 幂等更新。当前删除提示只有计数，不能定位被删除的 `user_name`；`other_records` 为未分类记录计数，键的详细解释须见授权资料。精确删除应通过新的完整快照核对。

##### 版本 / 顺序对账(全量快照 vs SSE 增量的覆盖边界)

HTTP 全量快照不携带可与 SSE `seq` 直接比较的覆盖点。`seq` 只对同一 `generation` 内的 SSE 事件排序，不能自行拼出跨通道版本。

- **SSE 内全量加增量**：在登录前或登录时开启 `contacts` 订阅，按流内 `seq` 升序处理。收到 `full:true` 整体重置名单，其后应用更高 seq 的增量；跨代丢弃旧游标并等待新全量。订阅默认关闭，登录后才开启会错过登录时一次性全量。
- **HTTP 全量加 SSE 增量**：`generation` 变化时以新全量重建，旧代增量全部丢弃。同代无法精确判断滞后增量是否已被 HTTP 快照覆盖，应按 `user_name` 保守幂等合并；删除以新完整快照中的缺失为准，删除计数仅作刷新提示。不确定顺序时标记需刷新，不宣称精确恢复删除顺序。

### 7.2.1 视频号私信 `finder.private_send`

先取得 `finder.open_session` 返回值，再将其原样作为 `session` 传给发送命令。`session` 至少包含 `session_id/finder_username/real_chat_username/self_chat_username`，`my_account_type` 按返回值携带。禁止跨账号或目标替换会话；同一账号和目标的会话标识可跨重新登录复用。

发送参数为 `content` 与 `session`。不要传 `client_msg_id/msg_source`，由服务处理；自行覆盖可能导致响应受理但消息不能正常投递。正常发送结果应保存返回的 `client_msg_id/server_message_id`。

```json
{
  "method":"finder.private_send",
  "params":{"content":"示例私信","session":{"session_id":"<SESSION_ID>","finder_username":"v2_example_creator@finder","real_chat_username":"wxid_example_self","self_chat_username":"wxid_example_self","my_account_type":2}},
  "control":{"authorize":true,"idempotency_key":"example-private-send-001"}
}
```

`finder.open_session` 与 `finder.private_send` 均须开启各自能力并逐次授权，使用唯一幂等键。建会话只需目标创作者标识，不要求先关注或互动；但可能创建外部状态，不能当作可反复执行的普通查询。能否发送仍受平台规则限制。

陌生会话在收到对端回复前只能发送首条消息，调用方须维护会话发送状态，并通过 `finder.private.sync` 确认对端回复后再继续；不能凭一次 `ok=true` 认定对端已读或可继续多发。服务不提供已读回执，受理和消息 ID 也不保证对端实际可见。

**拒绝与停发**：以 `value.error_code` 及对应字段处理，不只看 `ret`。

| `value.error_code` | 可见行为 |
|---|---|
| `FINDER_PRIVATE_SEND_TARGET_PRIVACY` | `account_send_blocked=false`；不设置账号级停发，不返回 `retry_at/blocked_day` |
| `FINDER_PRIVATE_SEND_DAILY_LIMIT` | `account_send_blocked=true`；已排队的发送也受阻挡，`retry_at` 表示本地策略的下次可检查时间 |

拒绝为 `FAILED`，不返回顶层 `value.server_message_id`。结果可能带 `classification_source` 和 `rejection_sysmsg:{session_id,msg_type,sysmsg_type,plain,server_message_id,create_time}`；其中嵌套消息 ID 是拒绝提示的标识，不能作为发送成功或送达凭据。不能确定分类时按返回的阻挡状态处理，不推测内部判断过程。同步响应、SSE operation 与 operation 查询的 `value` 一致。

**可选人工续聊**：`params.continuation_mode="manual_reply"` 仅表达用户主动续聊意图，不授予发送权限，也不能用于营销或自动发送。该功能需要提供方确认支持及验证状态；当前为尚未完成端到端验证的可选能力。服务独立验证后决定是否允许，失败时遵循 `request_sent`、错误码和阻挡状态；不得伪造对端回复或重用历史证据。具体授权条件须从提供方取得。

**可选本地误判解除**：`finder.private_send.clear_limit` 使用独立 capability，默认关闭，须逐次 `authorize=true`，提供与当前限制一致的 `blocked_day` 及非空 `reason`。它仅清除本服务确认可解除的本地限制，不解除平台真实限制、不重试历史消息，也不改写旧 operation/幂等终态；失败时保留阻挡，新发送使用新幂等键。

```json
{"method":"finder.private_send.clear_limit","params":{"blocked_day":"2000-01-01","reason":"示例：经授权核实本地分类错误"},"control":{"authorize":true,"idempotency_key":"example-clear-limit-001"}}
```

返回 `value:{ok:true,cleared:true,blocked_day:"2000-01-01"}`；没有本地限制时 `cleared=false`。日期仅为合成示例，实际必须匹配当前限制；原因与时间留存审计。策略过期或解除不保证平台同期解除限制。

### 7.2.2 视频号私信的接收与回执

| 项目 | 普通聊天 | 视频号私信 |
|---|---|---|
| 消息交付 | SSE `message` | 调用 `finder.private.sync` |
| 订阅 | `messages` 订阅 | 无消息订阅和消息 SSE |
| 方向 | SSE 的收发字段 | `messages[].is_sender` |
| 去重与对账 | `new_msg_id` | `server_message_id` |

按数据来源区分两类消息，不按昵称或 wxid 猜测。SSE `operation` 只提供命令结果，不能当作私信消息流。

私信同步返回会话摘要 `session_id/message_count/latest_create_time/latest_content/latest_is_sender`，并提供逐条 `messages[]`，字段见 §7.1.3。正文为文本或类型化媒体占位/XML，无媒体下载。`conversation.finder_username` 可能为空，不包含对端昵称和头像；可使用非空创作者标识调用 `finder.creator` 获取 `contact.nickname/head_url`，调用方可按账号身份映射保存并按需刷新。

发送结果应同时保存 `client_msg_id/server_message_id`，以 `server_message_id` 与同步消息关联。同步的 `client_msg_id` 可能为空，不能依赖它，也不能从 `local_message_id` 推算。`store_epoch` 变化后不保证补回历史 `client_msg_id`；旧发送记录若只保存一个不同 ID，应如实保留对账缺口。

同步 `200`、SSE `operation`、`GET …/operations/{id}` 的 `value` 一致。正常结算且已分配消息 ID 时返回两个 ID；响应丢失时按 operation 或幂等键补查，不通过重新发送补结果。`OUTCOME_UNKNOWN` 可能无 `value`，不能推算上游消息 ID，应以 operation 保留不确定状态。

普通聊天断线后用同代 `seq` 续传，按 `new_msg_id` 去重；跨 `generation` 重建游标。私信同步可能重复返回累计消息，按消息 ID 幂等消费。当前没有视频号私信撤回/删除事件，也没有私信已读回执；普通聊天撤回可能以系统消息出现，v1 不单独建模。

### 7.2.3 视频号私信异常恢复与版本边界

**响应丢失与补取**：客户端未收到响应或尚未可靠保存时，不推进自己的消费点。同一 `store_epoch` 下再次同步可取得累计记录，按消息 ID 幂等补取；`completed=true` 时可确认该累计边界已完成，不能据此证明上游无限期历史完整。

`store_epoch` 变化表示累计结果边界变化。保留旧缺口，将新结果独立落库并按消息 ID 合并；不要用 `generation` 替代 `store_epoch` 判断同一累计范围。服务不保证上游历史保留时间或条数，调用方应保存自己已收到的数据。

`completed=false` 表示本轮未完成，可在同账号上一条命令到达终态后，使用相同 `self_chat_username` 继续同步；遵循 `retryable` 和限流建议，避免高频空轮询。同步不自动重试，失败时用 `value.sync_failure` 的 `messages_pulled/guaranteed_no_messages_pulled/gap/send_blocked` 判断取回范围和发送阻挡，不能因一次未取消息就抹除旧缺口。

**ID 范围**：正常 `server_message_id` 跨同账号会话和重新登录稳定；异常记录可能只有会话范围标识或为零。稳妥去重键为 `(账号,session_id,server_message_id)`。遇到非正 ID 保留原始记录，不把它作为去重或归属凭据。

**未知会话身份**：优先使用 `conversation.finder_username`；为空时不要按昵称或会话 ID 推算创作者。调用方应保存建会话时的目标 `finder_username` 与返回 `session_id` 的绑定。缺少可信身份的记录保留为未知，不擅自合并。当前没有独立的会话 ID 反查创作者接口。

---

## 8. 事件类型（SSE）

`GET …/events` 使用 SSE 的 `event` 区分事件类型，事件数据为 JSON。以下标识和时间均为合成示例；实际值由当前请求或事件返回，调用方不得自行推导标识的构成。

### 8.1 `message` — 收到的消息

普通单聊、群聊消息通过此事件推送：

```json
{
  "conversation": "wxid_example_sender",
  "new_msg_id": 10001,
  "msg_type": 1,
  "from_user": "wxid_example_sender",
  "to_user": "wxid_example_self",
  "is_chatroom": false,
  "chatroom_sender": "",
  "create_time": 1700000000,
  "preview": "示例文本"
}
```

| 字段 | 说明 |
|---|---|
| `conversation` | 会话标识，单聊和群聊均使用此字段 |
| `new_msg_id` | 消息标识，用于去重 |
| `msg_type` | 消息类型 |
| `from_user` / `to_user` | 发送方 / 接收方标识 |
| `is_chatroom` | 是否为群聊 |
| `chatroom_sender` | 群消息发送者标识；非群消息为空 |
| `create_time` | Unix 秒时间戳 |
| `preview` | 文本正文或媒体类型占位描述 |

文本消息的 `preview` 是折叠为单行的完整文本，群聊发送者前缀已移除。媒体和富消息仅返回类型化占位描述，不含媒体二进制或下载链接。v1 不提供普通聊天媒体文件或原始消息体获取接口。

常见 `msg_type`：`1` 文本、`3` 图片、`34` 语音、`43` 视频、`47` 表情、`49` 应用消息、`10000` 系统消息、`10002` 撤回相关系统消息。调用方应按 `new_msg_id` 去重，并自行保存需要保留的消息。

### 8.2 `state` — 会话状态迁移

```json
{"state":"online","last_fault":null}
```

状态包括 `online`、`reauth_required`、`recovery_exhausted`、`offline`、`closed`。`reauth_required` 表示需要用户重新登录；网络恢复失败应与认证失效区分，见 §10.1。

### 8.3 `login` — 登录流程里程碑

携带当前登录阶段。阶段字段、二维码刷新与额外验证的处理方式见 §6.3 和 §10。

### 8.3.1 `command_ready` — 命令接收状态

```json
{"command_ready":true,"command_ready_at":1700000000.25,"generation":1,"reason":"startup_complete"}
```

账号级和节点级账号 SSE 均使用 `event: command_ready`；节点事件还携带 `uin`、`type` 与 `ts`。调用方应按 `generation` 维护就绪状态，收到 `resync` 后重新读取账号列表。只有 `command_ready=true` 才可提交业务命令。

### 8.4 `data` — 已订阅的业务数据

```json
{"type":"contacts","data":{"...":"该类型的数据结构"}}
```

`type` 对应 §9 的订阅类型。此类推送为尽力交付；调用方应结合按需查询进行核对。

### 8.5 `operation` — 命令终态

```json
{
  "operation_id":"operation-example-001",
  "method":"finder.private_send",
  "status":"SUCCEEDED",
  "outcome":"SUCCEEDED",
  "error":null,
  "idempotency_key":"private-send-example-001",
  "value":{
    "ret":0,
    "ok":true,
    "client_msg_id":"opaque-client-message-id",
    "server_message_id":10002
  }
}
```

已接受的命令会产生终态事件，但断线等情况可能导致调用方漏收。结果核对应以 operation 查询为依据：

- 写命令通过 `operation_id` 或请求提供的 `idempotency_key` 查询；读命令通过 `request_id` 查询。
- 每条写命令应提供唯一幂等键。未提供幂等键且 HTTP 响应丢失时，无法只凭请求内容找回服务生成的操作标识。
- 同账号、同一幂等键的重复提交复用原 `operation_id`；原操作未结束时可返回原操作当前状态。
- 同步响应、SSE `operation` 与 operation 查询中的 `value` 字段一致。成功发送且已分配消息标识时，结果包含 `client_msg_id` 和 `server_message_id`。
- `OUTCOME_UNKNOWN` 表示发送结果无法确定，可能不含 `value`。不得由此推算消息标识或自动重发。
- 因账号未就绪而在接受前返回 `409` 的请求不创建 operation，也不产生 operation 终态事件。

读命令事件示例：

```json
{"operation_id":"operation-example-002","request_id":"profile-example-001","method":"account.profile","status":"SUCCEEDED","outcome":null,"error":null,"value":{"...":"响应数据"}}
```

失败事件中的 `failure` 与同步响应、状态查询一致。私信拒绝分类见 §7.2.1；拒绝结果中的系统提示标识只能用于定位提示，不能作为消息送达证明。

---

## 9. 订阅类型

| `type` | 含义 | 事件 | 默认值 |
|---|---|---|---|
| `messages` | 普通单聊、群聊消息 | `message` | 关闭 |
| `contacts` | 通讯录全量及增量 | `data` | 关闭 |
| `profile` | 本账号资料 | `data` | 开启 |
| `sns` | 朋友圈互动通知 | `data` | 关闭 |

通过 `POST …/subscription` 设置 `{"enabled":true,"type":"..."}`。数据订阅在重新登录后保留；`profile` 每次登录默认开启。

`profile` 和 `contacts` 的登录推送发生在账号上线时。需要登录时通讯录推送的调用方，应在该账号登录前或登录时开启 `contacts`；登录后才开启会错过这一轮推送，可改用 `contact.list` 查询。按需调用 `account.profile` 可直接取得本账号资料，不依赖登录推送时序。

视频号搜索、群成员详情、朋友圈时间线等内容通过命令按需读取，不属于上述订阅推送。

---

## 10. 登录状态机

`GET …/login/{login_id}` 及登录事件中的 `phase`：

```text
starting → qr_pending → scanned → confirmed → authenticating → online
                              ↘ failed / aborted
```

| `phase` | 含义 |
|---|---|
| `starting` | 已受理，二维码准备中 |
| `qr_pending` | 等待扫码 |
| `scanned` | 已扫码，等待手机端确认；若要求额外验证，应使用当前 `auth_url` 完成验证 |
| `confirmed` | 用户已确认，服务开始认证 |
| `authenticating` | 认证处理中 |
| `online` | 账号已上线；仍须等待 `command_ready=true` 后提交命令 |
| `failed` / `aborted` | 登录失败 / 已中止，均为终态 |

指定 `account_id` 后使用其他账号扫码会被拒绝。恢复期间不应同时发起新的扫码尝试；是否需要重新登录，应依据账号状态而非等待时间判断。可选自动恢复能力及其适用范围由服务提供方确认。

### 10.1 会话态(`state`)语义 —— `offline` 到底是什么

| `state` | 含义 | 调用方处理 |
|---|---|---|
| `online` | 账号会话在线；恢复期间可能暂时不可接收命令 | 同时检查 `command_ready` |
| `reauth_required` | 明确需要用户重新认证 | 引导用户显式重新登录 |
| `recovery_exhausted` | 网络恢复未在允许时间内完成 | 检查网络、代理与恢复状态，不自动要求扫码 |
| `offline` | 会话已停止 | 根据业务需要显式重新登录 |
| `closed` | 会话已关闭 | 根据业务需要显式重新登录 |

瞬时网络恢复期间，账号可能仍显示 `online`，但 `command_ready=false`。`offline`、`closed`、`reauth_required` 不应被当作仍在自动恢复的临时状态。重新登录成功会产生新的 `generation`。

`reauth_required` 账号仍可查询状态；账号已被移除时，其账号级 SSE 可能返回 `404`。建议使用节点级 `GET /v1/accounts/events` 统一感知账号变化，并通过账号列表核对当前状态。

---

## 11. 代理阶段与租约

代理阶段 `proxy_phase` 与登录阶段 `phase` 相互独立，其公开枚举为：

| `proxy_phase` | 含义 |
|---|---|
| `login_bootstrap` | 登录准备 |
| `login_cgi` | 登录请求处理中 |
| `read_lane` | 读取准备完成 |
| `longlink` | 连接建立 |
| `reconnect` | 正在恢复连接 |
| `online` | 当前连接可用 |

这些值用于展示和事件关联；业务命令是否可调用仍以 `command_ready` 为准。

租约状态为 `reserved` → `active` → 释放终态。登录成功激活时返回 `confirmed=true, terminal=false`；`terminal=true` 仅表示租约已释放，须同时携带 `release_reason` 和 `release_event_id`。已释放租约不应再次使用。

**登录状态查询中的 `proxy` 快照字段：**

| 字段 | 含义 |
|---|---|
| `lease_id` / `login_id` / `account_id` / `uin` / `generation` | 关联标识 |
| `epoch` | 当前代理连接代次，建立前可为 `null`；连接变化后可能递增 |
| `label` | 调用方提供的代理标记 |
| `lease_state` / `proxy_phase` | 租约状态 / 代理阶段 |
| `ok` | 当前是否健康 |
| `terminal` | 租约是否已释放 |
| `release_reason` / `release_event_id` | 释放原因 / 稳定释放事件标识，未释放时为 `null` |
| `last_heartbeat_seq` | 最近心跳序号 |
| `observed_egress_ip` / `region` | 最近观测到的出口 IP / 可选地域 |
| `egress_check_status` | `success` / `failed`，尚未检查时为 `null` |
| `egress_error_code` | 失败分类；成功或尚未检查时为 `null` |
| `egress_ts` / `egress_epoch` | 检查完成的 Unix 秒时间戳 / 对应连接代次 |

`GET /v1/proxy/events` 的 `proxy_status` 事件包含 `lease_id`、`login_id`、`account_id`、`uin`、`generation`、`epoch`、`proxy_phase`、`event_seq`、`label`、`ok`、`terminal`、`confirmed`、`release_reason`、`release_event_id` 和 `error`。心跳事件使用 `heartbeat_seq`，不要与快照的 `last_heartbeat_seq` 混用。

`proxy_egress` 事件包含 `lease_id`、`login_id`、`account_id`、`uin`、`generation`、`epoch`、`event_seq`、`label`、`observed_egress_ip`、`region`、`check_status`、`error_code` 和 `ts`。成功时 IP 非空；失败时 IP 为 `null`。错误码包括 `endpoint_error`、`proxy_connect`、`timeout`、`tls_error`、`http_status`、`invalid_response`、`network_error`、`scheduler_error` 和 `internal_error`。

出口检查结果只代表对应 `epoch` 的一次观测，不能证明所有业务连接始终使用同一出口；需要固定出口时，应由代理提供方提供相应能力。

---

## 12. 调用语义（务必阅读）

### 12.1 读 vs 写

读命令同步返回业务结果。写命令返回操作标识和状态；同步模式可能直接返回终态，异步模式需要继续查询或接收事件。

### 12.2 写命令三重门禁

每次写操作必须同时满足：

1. 账号已开启对应 capability；写能力默认关闭。
2. 当前请求带 `control.authorize=true`。
3. 账号会话有效且允许执行该操作。

前置条件不满足时返回 `REJECTED` 或 `FAILED`，不执行写操作。能力开关不能代替用户对具体操作的授权。

### 12.3 幂等与不重试

同账号同一幂等键的重复写请求复用原操作。原操作尚未结束时，可返回它的当前状态；不能将尚未终态的操作当作已经完成。

`OUTCOME_UNKNOWN` 不表示未发送，不应自动重试。读请求是否可重试应结合 `failure.request_sent`、`retryable` 及该方法的限制判断。

### 12.4 消息零存储 + 至少一次投递

普通聊天 SSE 不提供持久消息历史查询；调用方应按至少一次投递处理重复事件，并自行完成去重、存储和展示。去重标识使用 `new_msg_id`。视频号私信的拉取与历史边界按 §7.2.2、§7.2.3 处理。

### 12.5 “代”(generation)与 SSE 游标

- 每次重新登录后，账号进入新的 `generation`，原 `seq` 游标不再作为新代的消费位置。
- 断线重连应同时携带最近处理的 `since` 和 `generation`。
- 事件返回当前 `generation`；检测到变化时，调用方须按新代重新核对账号和订阅状态。
- 收到 `resync` 时应重新查询权威快照，不能仅依靠本地旧事件恢复状态。

### 12.6 账号隔离

账号级接口按路径中的 `uin` 确定目标账号。调用方须校验自己业务中的账号归属，避免将不同账号的操作、游标或上下文混用。一个服务实例只供一套后台或同一授权域使用；API 不提供独立的租户隔离维度，见 §14 Q2。

### 12.7 ⚠️ 单账号串行:一个账号同一时刻只做一个操作(务必遵守)

同一账号必须串行提交命令：发起一条 → 等到持久 operation 终态 → 再发下一条。不同账号可以并行。

`202` 表示已接受，`504` 表示本次 HTTP 等待超时，二者都不能作为释放账号队列的依据。应通过 operation 查询或关联的终态事件确认结束。翻页同样逐页执行，不并发请求同一账号的多页。

### 12.8 有些"读"需要配套调用"写"(已读上报等)

读取数据不等于改变业务状态。需要标记已读或执行其他有副作用的配套动作时，应调用相应写命令，开启对应能力并逐次授权。例如普通聊天的 `message.mark_read`。

视频号私信的已读和其他状态上报仅在明确支持、明确授权时执行；调用方不能把读取成功当作这些状态已经更新。

---

## 13. 端到端对接流程

本章展示调用形状，使用虚构地址和样例标识。实际调用须取得授权，替换地址、证书、账号及当前读结果。省略的鉴权配置按 §2 补齐；下列 shell 示例依赖 `curl` 和 `jq`。每一步均须检查状态，等待前一步终态后再继续。

### 13.1 上号(登录)

```bash
BASE=https://api.example.invalid:8443
LOGIN_RESPONSE=$(curl -sX POST "$BASE/v1/accounts/login" \
  -H 'Content-Type: application/json' -d '{}')
LOGIN=$(printf '%s' "$LOGIN_RESPONSE" | jq -r .login_id)
# 展示 LOGIN_RESPONSE 中的二维码，并订阅本次登录的状态事件。
curl -sN "$BASE/v1/accounts/login/$LOGIN/events"
# 断线时用查询兜底；上线后取得 uin，并继续等待 command_ready=true。
curl -s "$BASE/v1/accounts/login/$LOGIN"
```

### 13.2 收消息

```bash
curl -sX POST "$BASE/v1/accounts/$UIN/subscription" \
  -H 'Content-Type: application/json' -d '{"enabled":true,"type":"messages"}'
curl -sN "$BASE/v1/accounts/$UIN/events?since=0"
# 重连携带上次处理的 seq 和 generation；按 new_msg_id 去重后保存。
```

### 13.3 执行一次写(以点赞为例)

```bash
curl -sX POST "$BASE/v1/accounts/$UIN/capabilities" \
  -H 'Content-Type: application/json' -d '{"enable":["finder.like"]}'
# 先读取当前账号的作品详情，将完整 object 保存在 OBJECT_JSON，作者标识保存在 FINDER_USERNAME。
# 不自行构造 object 内的上下文字段。
REQUEST=$(jq -n --argjson object "$OBJECT_JSON" --arg finder "$FINDER_USERNAME" \
  '{method:"finder.like",params:{object:$object,finder_username:$finder},
    control:{authorize:true,idempotency_key:"like-example-001"}}')
curl -sX POST "$BASE/v1/accounts/$UIN/commands" \
  -H 'Content-Type: application/json' -d "$REQUEST"
# 返回 202 时继续等待 operation 终态；失败或结果未知时不自动重发。
```

### 13.4 下号

```bash
curl -sX DELETE "$BASE/v1/accounts/$UIN"
```

停止会话与彻底删除账号数据是不同操作，见 §6.5、§6.5.1。

### 13.5 视频号完整链路对接示例(端到端)

所有方法均经 `POST /v1/accounts/{uin}/commands` 提交。以下片段分别展示请求结构；分页和作品上下文须使用当前响应返回值，不得跨账号或跨代复用。私信会话不得跨账号或目标混用，重新登录后的复用边界按 §7.2.1 处理。

**推荐和搜索：**

```json
{"method":"finder.feed","params":{}}
```

```json
{"method":"finder.search","params":{"query":"示例关键词","category":"video","sort":"hot"},"control":{"request_id":"search-example-001"}}
```

推荐续页回传 `cursor_hex`；搜索续页回传 `page_token` 并保留相同查询条件。令牌视为不透明值，不解析或生成它们。

**打开作品及读取评论：**

推荐或主页作品使用返回的作品标识读取详情。搜索项按 §7.1 的流程提交 `finder.open_search_object`，所需字段来自同一搜索项。取得详情后，用详情中的作品标识调用 `finder.comments`。不得混用搜索项标识和作品详情标识。

```json
{"method":"finder.creator","params":{"finder_username":"v2_example_creator@finder"}}
```

```json
{"method":"finder.comments","params":{"object_id":"100001"}}
```

作品统计字段包括 `like_count`、`comment_count`、`forward_count`、`read_count`、`fav_count`；`null`、零值和不可见的处理按 §7.1。

**关注和取关：**

先取得已确认的当前关注态及当前作品读结果，分别开启 `finder.follow` 或 `finder.unfollow`，将 `contact` 和 `ref_object` 原样回传。每次带 `control.authorize=true` 和唯一幂等键；不能凭目标操作猜测当前关注态。失败时停止，不自动重试。

**建会话及发送私信：**

```json
{"method":"finder.open_session","params":{"finder_username":"v2_example_creator@finder"},"control":{"authorize":true,"idempotency_key":"open-session-example-001"}}
```

该方法本身是写操作。确认成功后，将响应中的完整会话对象作为 `session` 原样传给 `finder.private_send`，同时开启发送能力、逐次授权并提供唯一幂等键。不要自行生成 `client_msg_id` 或 `msg_source`。应保存发送结果中的两个消息标识，并以 `server_message_id` 对账；成功响应不等于对方已读。

**接收私信及核对结果：**

```json
{"method":"finder.private.sync","params":{"self_chat_username":"wxid_example_self"}}
```

普通聊天使用 SSE `message`，视频号私信使用 `finder.private.sync` 拉取。按 `server_message_id` 去重，按 `is_sender` 区分方向；会话对端资料可通过 `finder.creator` 按需取得。发送响应漏收时，按 operation 标识或幂等键查询，不通过盲目重发进行核对。

### 13.6 通讯录:从 SSE 流内取全量 + 增量(口径 A,精确对账)

需要登录时全量推送时，应在该账号登录前或登录时开启 `contacts`。对同一 `generation` 的通讯录事件，按 `seq` 升序处理：`full:true` 表示以该批次重置名单，其后的增量再依次应用。重复事件应幂等处理。

```bash
curl -sX POST "$BASE/v1/accounts/$UIN/subscription" \
  -H 'Content-Type: application/json' -d '{"enabled":true,"type":"contacts"}'
curl -sN "$BASE/v1/accounts/$UIN/events?since=0"
```

重连携带 `since` 和 `generation`。代次变化时丢弃旧代消费位置，并等待新代全量或主动查询快照。当前删除提示不足以定位具体联系人时，以下次完整名单核对，见 §7.1.5。HTTP 全量查询与 SSE 增量的边界按 §7.1.5 的保守合并规则处理，不能假定两个通道共享可比较的游标。

---

## 14. 对接方常见问题（FAQ / 待确认项）

### Q1. 写命令是同步还是异步模式?

两种模式均支持，默认同步，具体配置由服务提供方确认。客户端须同时兼容直接返回终态的 `200` 和待继续跟踪的 `202`。

### Q2. 一个服务实例是否只服务一套后台?有无租户隔离?

一个实例只应服务一套后台或同一授权域。本 API 不提供租户隔离；不同业务方、测试与生产环境应使用独立实例及各自的授权凭据。接入方应按 §2 获取服务地址和客户端证书。管理接口须由服务提供方另行限制权限。

### Q3. `message` 事件只有 `preview`,能否拿到完整消息体/媒体?

文本的 `preview` 为单行完整文本；媒体消息仅为类型化占位。v1 不提供普通聊天的媒体文件或原始消息体获取接口。

### Q4. `account.profile` 结果字段(昵称/头像/wxid)?

按需调用 `account.profile` 返回 `profile.user_info` 和 `profile.user_info_ext`，包含账号标识、昵称、微信号及头像 URL 等。`profile` 订阅默认开启，但只在登录时推送一次；需要随时取得资料时优先使用按需查询。账号列表用于状态展示，不包含昵称、头像等资料。

### Q5. 代理租约是否需要保留粘性绑定(续费后复用)?

代理与账号的业务绑定由调用方管理。登录时须显式提供有效代理；`lease_id` 和 `label` 用于关联，不保证后续登录自动复用代理。需要固定出口或粘性会话时，应向代理提供方确认支持范围。

### Q6. 账号容量与消息保留期限?

实例容量和调用额度向服务提供方确认。普通聊天 SSE 不提供持久消息历史，保留期限由调用方自己的保存策略决定；视频号私信的可读历史与恢复边界按 §7.2.3 处理。

### Q7. `DELETE /v1/accounts/{uin}` 后是否清除缓存身份?两种删除怎么选?

`DELETE /v1/accounts/{uin}` 停止会话并保留身份，之后可以使用原 `account_id` 重新扫码登录。`DELETE /v1/accounts/{uin}/data` 删除当前服务管理的该账号数据，下次登录按全新设备处理。彻底删除为管理员操作，外部备份不属于该接口范围，见 §6.5.1。

### Q8. 重复登录、会话来源、登录态保留、`offline` 语义、节点事件流

- 带 `account_id` 的重复登录按 §6.2 去重；全新登录须由调用方避免并发重复提交。
- 通过账号列表核对 `login_id`、`online_since`、`generation`、订阅与能力；`device` 可为 `null`，也应兼容旧版本缺字段。
- 登录终态至少保留 24 小时，但服务重启后登录记录可能失效；当前账号状态应通过账号列表重新核对。
- `offline`、`closed`、`reauth_required` 的处理按 §10.1，不把它们当作仍会自动恢复的状态。
- 节点级账号事件流用于统一感知所有账号的登录、状态与命令就绪变化。

### Q9. 二维码过期/用户点「刷新二维码」怎么拿新码?

二维码过期不一定立即令登录失败。查询返回 `expired=true` 时，可取消旧登录后再发起一次登录；不先取消会因幂等语义拿回原登录。已经进入不允许取消的阶段时，取消接口返回 `aborted:false`，见 §6.3.1。

### Q10. HTTP `504` 和 operation 的 `DEADLINE_EXCEEDED` 是同一个终态吗?

HTTP `504` 只表示该次 HTTP 等待超时；operation 查询仍可能为 `ACCEPTED` 或 `RUNNING`。只有查询到持久终态后，才能释放该账号队列。operation 一旦进入 `DEADLINE_EXCEEDED` 终态，不会再变为成功或失败。

### 对接配置与行为速查

| 项目 | 处理方式 |
|---|---|
| 地址、证书、额度、可选功能 | 向服务提供方取得 |
| 实例归属 | 一套后台或同一授权域一个实例 |
| 普通聊天媒体获取 | v1 不提供 |
| 账号资料 | `account.profile` 按需读取 |
| 代理绑定 | 调用方维护 |
| 两种删除 | 区分停止会话与管理员彻底删除 |
| 幂等、超时与重试 | 按 §6.6、§6.7、§12 处理 |
| 非公开上下文 | 使用当前响应原样回填，具体接入细节按授权说明取得 |
