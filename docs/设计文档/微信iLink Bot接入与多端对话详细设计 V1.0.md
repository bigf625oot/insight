# 微信 iLink Bot 接入与多端对话详细设计 V1\.0

> 本文档定义自研智能体通过腾讯官方**微信 iLink Bot API**（ClawBot/OpenClaw，2026\.03 开放）接入微信的详细设计，实现「微信即智能体」多端统一对话架构。取代原企业微信 Bot 方案，零客户安装成本。

# 1\. 设计目标

1. **自研智能体接入微信**：把本系统智能体（SaaS数据 \+ 规则库 \+ LLM）通过 iLink Bot API 接入微信个人号，用户在微信直接和 AI 助手对话。
2. **多端统一对话**：一个智能体后端，三端触达（运营 Web / 店长微信 / 老板微信，店长和老板均通过 iLink Bot），对话跨端可关联。
3. **零安装成本**：老板/店长用日常微信加 AI 助手号为好友即可，无需装企微或额外 app。
4. **对话式拍板**：拍板请求主动推送到微信，老板用自然语言回复（"同意"/"换小份"），智能体解析为决策。
5. **iLink 作消息总线**：任务分配、回传提醒、拍板请求全走 iLink 推送，替代企微 Bot 全部场景。

# 2\. iLink Bot API 协议规约

## 2\.1 基础

- 基座地址：`https://ilinkai.weixin.qq.com`
- CDN：`https://novac2c.cdn.weixin.qq.com/c2c`
- 协议：HTTP/JSON（非 WebSocket，长轮询）
- npm 包：`@tencent-weixin/openclaw-weixin`（核心协议）、`@tencent-weixin/openclaw-weixin-cli`（安装器）
- 一个 iLink Bot 账号 = 一个微信个人号（品牌 AI 助手号）

## 2\.2 通用请求头

每个业务 POST 请求：

```Plaintext
Content-Type: application/json
AuthorizationType: ilink_bot_token
Authorization: Bearer <bot_token>
X-WECHAT-UIN: <base64(String(random_uint32))>   # 每次请求重新生成
```

请求体统一含：`{ "base_info": { "channel_version": "2.0.0" } }`

## 2\.3 接口列表

|接口|方法|用途|
|---|---|---|
|`/get_bot_qrcode?bot_type=3`|GET|获取登录二维码|
|`/get_qrcode_status?qrcode=...`|GET|轮询扫码状态|
|`/getupdates`|POST|长轮询接收消息（35s 挂起）|
|`/sendmessage`|POST|发送文本/媒体消息|
|`/getconfig`|POST|获取 typing\_ticket 等配置|
|`/sendtyping`|POST|显示/隐藏"正在输入"|
|`/getuploadurl`|POST|获取 CDN 上传参数|
|CDN `/upload`|POST|上传 AES 加密媒体|
|CDN `/download`|GET|下载 AES 加密媒体|

## 2\.4 登录流程

二维码扫码确认模式，状态机：`wait` → `scaned` → `confirmed`（或 `expired` → 重新获取，最多 3 次）。

- `confirmed` 返回 `bot_token`、`ilink_bot_id`、`baseurl`、`ilink_user_id`
- `bot_token` 是 Bearer token，后续所有调用携带
- `baseurl` 可能与默认值不同，始终用返回值
- 凭证持久化到 `data/credentials.json`（权限 0600），下次启动复用

## 2\.5 消息收发

**长轮询收消息**（`getupdates`）：

- 首次 `get_updates_buf: ""`，每次响应返回新 buf 作为不透明游标
- 服务端 hold 约 35 秒
- `ret: 0` 成功；`ret: -14` 会话过期
- 长轮询超时（AbortError）是正常现象，返回空响应继续下一轮

**发消息**（`sendmessage`）：

```JSON
{
  "msg": {
    "from_user_id": "",
    "to_user_id": "<目标用户>",
    "client_id": "<客户端ID>",
    "message_type": "BOT",
    "message_state": "FINISH",
    "item_list": [{ "type": "TEXT", "text_item": { "text": "回复内容" } }],
    "context_token": "<必须原样回传入站消息的token>"
  }
}
```

`item_list` 支持 TEXT / IMAGE / VOICE / VIDEO / FILE。

## 2\.6 context\_token（核心）

- 每条入站消息含 `context_token`
- 每条出站 `sendmessage` **必须**原样回传，否则无法路由到正确微信会话
- 按 `userId` 缓存最新 token
- 跨重启持久化
- 会话过期（-14）或重新登录时清除

## 2\.7 媒体加密

CDN 媒体用 AES\-128\-ECB \+ PKCS7 加密。三种密钥格式（base64原始字节 / base64十六进制串 / 直接十六进制）。加密后大小 = `ceil((rawSize + 1) / 16) * 16`。

## 2\.8 错误码

|错误码|含义|处理|
|---|---|---|
|`ret: 0`|成功|\-|
|`errcode: -14`|会话过期|清状态 → 重新扫码登录|
|`ret: -2`|参数错误|检查请求体|
|HTTP 4xx|请求/认证失败|检查 token|
|HTTP 5xx|服务端错误|指数退避重试|

# 3\. 整体架构

```Plaintext
┌──────────┐   ┌──────────────┐   ┌──────────────┐
│ 运营Web  │   │ 店长微信     │   │ 老板微信     │
│ 对话工作台│   │ (iLink Bot) │   │ (iLink Bot) │
└─────┬────┘   └──────┬───────┘   └──────┬───────┘
      │ Web API       │ iLink协议        │ iLink协议
      │               │ (getupdates/     │
      │               │  sendmessage)    │
      ↓               ↓                  ↓
┌─────────────────────────────────────────────────────┐
│              智能体网关（Agent Gateway）             │
│   身份路由 / session管理 / 消息分发 / 推送队列       │
└───────────────────────┬─────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────┐
│              智能体后端（Agent Core）                │
│   Agent编排 / SaaS数据 / 规则库 / LLM / 溯源         │
└─────────────────────────────────────────────────────┘
```

- **iLink 网关**：封装 iLink 协议，收发微信消息，管理 context\_token / 会话，路由到智能体
- **智能体网关**：多端统一入口，身份路由、session 管理、主动推送队列
- **智能体后端**：复用已有 Agent Core（诊断/审核/建议/溯源），不因通道而异

# 4\. iLink 网关层设计

## 4\.1 职责

- 维护一个 iLink Bot 账号（品牌 AI 助手微信号）的登录态
- `getupdates` 长轮询接收微信消息
- 调用智能体后端获取回复
- `sendmessage` 发回微信（带 context\_token）
- 主动推送（任务/拍板）通过 `sendmessage` 发起
- context\_token 持久化管理
- 会话过期自动告警/重连

## 4\.2 模块结构

```Python
class ILinkGateway:
    def __init__(self, config, agent_gateway):
        self.base_url = "https://ilinkai.weixin.qq.com"
        self.credential_store = ILinkCredentialStore()   # bot_token 持久化
        self.context_store = ContextTokenStore()          # context_token 持久化
        self.agent_gateway = agent_gateway                # 智能体网关

    async def start(self):
        """主循环：登录 → getupdates 长轮询 → 处理消息"""

    async def login(self):
        """扫码登录，获取 bot_token，持久化"""

    async def send(self, to_user_id: str, text: str, context_token: str):
        """主动发送消息（需 context_token）"""

    async def send_typing(self, user_id: str):
        """显示'正在输入'"""
```

## 4．3 消息处理流程

```Plaintext
1. getupdates 长轮询拿到 msgs[]
2. 对每条 msg：
   a. 过滤：仅处理 message_type == USER
   b. 提取并缓存 context_token（按 userId）
   c. 提取文本内容
   d. 按 userId 识别身份（老板/店长/未绑定）→ 见第5节
   e. 调智能体网关处理（带身份+门店上下文）→ 获取回复
   f. sendmessage 发回微信（回传 context_token）
3. 更新 get_updates_buf 游标
4. 会话过期(-14) → 清状态 → 告警 → 等待重新扫码
```

## 4．4 context\_token 管理

- `ContextTokenStore` 按 `userId` 存最新 token，持久化（Redis/DB）
- 主动推送时取对应用户的最新 context\_token
- 若用户长时间未发消息（token 失效），主动推送可能失败 → 降级为"请先发任意消息激活"或运营电话/微信代录

# 5\. 身份识别与绑定

iLink Bot 的微信 userId 需绑定到本系统角色：

## 5．1 绑定流程

```Plaintext
1. 老板/店长首次在微信加 AI 助手号，发任意消息
2. iLink 网关收到消息，userId 未绑定 → 回复引导：
   "你好，我是XX品牌AI助手。请回复你的手机号完成绑定"
3. 用户回复手机号 → 系统按手机号匹配 store_staff / boss 账号
4. 匹配成功 → 绑定 userId ↔ 账号，回复"绑定成功"
5. 匹配失败 → 回复"未找到账号，请联系运营开通"
```

## 5．2 绑定关系存储

新增 `ilink_binding` 表：

|字段|类型|说明|
|---|---|---|
|id|VARCHAR(36)|PK|
|weixin_user_id|VARCHAR(64)|NOT NULL, UNIQUE|iLink userId|
|role|VARCHAR(20)|NOT NULL|boss / staff|
|ref_id|VARCHAR(36)|关联 boss\_id / store\_staff\_id|
|phone|VARCHAR(20)|绑定手机号|
|bound_at|TIMESTAMP||

## 5．3 身份路由

收到微信消息 → 查 `ilink_binding` → 得到 role \+ ref\_id → 智能体网关按身份组装上下文（老板看拍板/汇总，店长看任务，未绑定引导绑定）。

# 6\. 多端对话 session 模型

## 6．1 session 维度

session 按 **角色 \+ 门店（/任务/作战卡）** 维度，而非按端：

- 老板在微信问"门店A最近怎么样" → session(boss, store\_A)
- 运营在 Web 诊断门店A → session(operator, store\_A)
- 店长在微信问"任务1怎么执行" → session(staff, task\_1)

不同角色的 session 各自独立，但都关联到门店/任务，**可互相引用与查看**（运营能看到老板在微信的拍板对话，老板能看到运营的诊断结论摘要）。

## 6．2 跨端对话可见性

- 老板微信对话内容 → 持久化到 `message` 表（channel=ilink），运营 Web 可查看
- 运营 Web 诊断结论 → 可通过 iLink 主动推送给老板微信
- 店长微信对话 → 持久化（channel=ilink），运营可查看

## 6．3 message 扩展

`message` 表 `channel` 字段：`web` / `ilink`，标识消息来源端（Web 后台 / iLink 微信）。

# 7\. 老板微信拍板交互

## 7．1 主动推送拍板请求

运营在 Web 标记"发给老板拍板" → 智能体网关 → iLink 网关 `sendmessage` 推送到老板微信：

```Plaintext
【拍板请求 · 门店A · 双人商务套餐】
建议：将麻婆豆腐替换为蟹粉豆腐
预期：毛利率 28%→55%
可信度：high  依据：规则卡 R012
请回复：同意 / 不同意 / 你的意见
```

## 7．2 老板自然语言回复解析

老板在微信直接回复，智能体解析为决策：

|老板回复|解析|结果|
|---|---|---|
|"同意"/"可以"/"行"|agree|battle\_card.operator\_status=confirmed|
|"不同意"/"不行"/"否"|disagree|operator\_status=rejected|
|"换小份"/"保留麻婆豆腐改小份"|revision|生成 feedback(revision) \+ 运营据此调整|
|其他文字|revision|记为 feedback，运营跟进|

- 解析由智能体（LLM）完成，输出结构化 `{decision, content}`
- 解析结果写 `feedback`（is\_boss\_decision=true）
- 同步更新 battle\_card 状态
- 回复老板"已收到您的决策：…"

## 7．3 拍板覆盖 Gate

老板"同意"可覆盖溯源完整性 Gate（见执行闭环设计 4\.3），task 标 `gate_overridden=true`。

# 8\. 主动推送场景（替代企微 Bot）

|场景|推送对象|触发|iLink 推送内容|
|---|---|---|---|
|优化建议通知|老板/运营|智能体生成建议|门店 \+ 风险数 \+ 建议摘要|
|任务分配通知|店长|运营分配任务|动作 \+ 目标菜品 \+ 观察周期 \+ 回复引导（"执行后回复图片+文字"）|
|执行回传提醒|店长|距截止 N 天|任务 X 距截止 N 天，请回复图片+文字完成回传|
|老板拍板请求|老板|运营标记拍板|建议内容 \+ 回复引导|
|反馈收集|老板|老板微信回复|自动解析为 feedback|

> 店长端推送：仅走 iLink 推送到店长微信（店长已绑定 iLink 为前提）。iLink 推送失败时降级为运营电话/微信沟通后代录。

# 9\. 会话保持与恢复

- `bot_token` 持久化到 `data/credentials.json`（权限 0600），重启复用
- 会话过期（errcode \-14）：清状态 → 告警通知运营 → 运营在后台触发重新扫码（二维码推送到运营 Web 或老板微信）
- 凭证失效后，入站消息无法接收，主动推送失败 → 监控告警
- 建议：一主一备两个 iLink Bot 账号，主号失效时切备号（可选）

# 10\. 媒体处理

- 店长/老板在微信发图片（如门店现场照、新菜单照片）→ iLink CDN 下载 \+ AES 解密 → magic bytes 校验 → 存本系统 → 关联到 task\_execution / feedback
- 执行回传图片通过 iLink 发送（店长在微信直接发图），智能体将图片关联到对应任务的执行回传
- 图片安全校验同 magic bytes 规则（见执行闭环设计），校验通过才入库
- 后续可扩展 iLink 图片多模态（智能体识别图片内容辅助判断）

# 11\. 异常与降级

|异常|处理|
|---|---|
|iLink 会话过期(-14)|清状态 \+ 告警，等待重新扫码；期间推送失败降级为运营代录|
|iLink 接口超时/5xx|指数退避重试（1s/2s/4s），3 次失败告警|
|getupdates 长轮询超时|正常，继续下一轮|
|context\_token 失效|主动推送失败，回复用户"请先发任意消息激活"|
|身份未绑定|引导绑定流程|
|智能体回复超时|回复"AI 思考中，请稍后"，异步处理完后主动推送结果|
|消息超长|分段 sendmessage（微信单条长度限制）|

# 12．接口契约

## 12．1 iLink 网关内部接口（与智能体网关）

```Plaintext
# 智能体网关 → iLink 网关：主动推送
POST /internal/ilink/push
请求：{
  "to_weixin_user_id": "wx_user_01",
  "msg_type": "text" | "boss_decision",
  "content": { "text": "..." } | { "battle_card_id": "bc_01", "suggestion": "..." }
}
响应：{ "data": { "sent": true, "message_id": "..." } }
错误：{ "error": { "code": "ILINK_SESSION_EXPIRED" | "ILINK_CONTEXT_INVALID" | "ILINK_TIMEOUT" } }
```

## 12．2 运营后台 iLink 管理

```Plaintext
GET  /api/v1/ilink/status                  # iLink Bot 登录状态（active/expired）
POST /api/v1/ilink/qrcode                  # 生成登录二维码（失效时重新扫码）
GET  /api/v1/ilink/bindings                # 绑定列表（userId ↔ 账号）
DELETE /api/v1/ilink/bindings/{id}         # 解除绑定
```

## 12．3 错误码

|code|说明|
|---|---|
|`ILINK_SESSION_EXPIRED`|iLink 会话过期，需重新扫码|
|`ILINK_CONTEXT_INVALID`|context\_token 失效，需用户激活|
|`ILINK_TIMEOUT`|iLink 接口超时|
|`ILINK_NOT_BOUND`|微信用户未绑定本系统账号|
|`BOSS_DECISION_PARSE_FAILED`|老板回复解析失败，转人工|

# 13．配置管理

|配置项|说明|
|---|---|
|`ilink.base_url`|`https://ilinkai.weixin.qq.com`|
|`ilink.cdn_url`|`https://novac2c.cdn.weixin.qq.com/c2c`|
|`ilink.bot_type`|3|
|`ilink.credential_path`|`data/credentials.json`|
|`ilink.long_poll_timeout_ms`|35000|
|`ilink.max_retry`|3|
|`ilink.bot_name`|品牌 AI 助手昵称|
|`ilink.backup_bot_enabled`|是否启用备号（默认 false）|

# 14．开发注意事项

1. **context\_token 必须持久化**：跨重启不丢，否则无法回复历史会话用户。
2. **bot\_token 安全存储**：credentials.json 权限 0600，不入库明文，不记日志。
3. **X\-WECHAT\-UIN 每次随机**：随机 uint32 → 十进制字符串 → base64，不可复用。
4. **长轮询超时是正常的**：AbortError 直接返回空响应继续，不报错。
5. **会话过期要告警**：\-14 是需人工介入的事件（重新扫码），不能静默。
6. **主动推送依赖 context\_token**：用户长期未互动 token 可能失效，推送失败要有降级（运营电话/微信沟通后代录）。
7. **老板回复解析用 LLM**：但需 schema 校验，解析失败转人工（运营跟进），不误判决策。
8. **消息分段**：微信单条文本有长度限制，超长分段发送。
9. **一个 iLink Bot 一个号**：多门店共用一个品牌 AI 助手号，靠身份路由区分，不每门店一号。
10. **iLink 是通道不是大脑**：所有业务判断在智能体后端，iLink 网关只管收发与路由，不写业务逻辑。
