# API 接口契约 V1\.0

> 本文档汇总《产品需求方案 V2\.0》及三份详细设计中的全量接口，作为前后端开发契约。所有接口统一前缀 `/api/v1`，统一响应格式与错误码。

# 1\. 通用约定

## 1\.1 基础

- 接口前缀：`/api/v1`
- 协议：HTTPS
- 字符编码：UTF\-8
- 时间格式：ISO 8601（`2026-06-19T10:30:00Z`）
- 金额单位：元（Decimal，保留 2\-4 位）

## 1\.2 鉴权

|端|鉴权方式|
|---|---|
|Web 后台|JWT（运营登录获取，Header `Authorization: Bearer <token>`）|
|iLink 微信端（店长/老板）|iLink Bot 协议（`Authorization: Bearer <bot_token>`），网关侧按 weixin\_user\_id 路由身份（通过 ilink\_binding 查 role + ref\_id），非 HTTP 直连|

## 1\.3 统一响应格式

成功：

```JSON
{
  "data": { ... } | [ ... ]
}
```

失败：

```JSON
{
  "error": {
    "code": "ERROR_CODE",
    "message": "可读错误信息",
    "reasons": ["可选，详细原因清单"]
  }
}
```

## 1\.4 分页约定

- 列表接口统一 query 参数：`page`（从 1）、`size`（默认 20，最大 100）
- 响应：

```JSON
{
  "data": {
    "items": [...],
    "total": 128,
    "page": 1,
    "size": 20
  }
}
```

## 1\.5 错误码总表

|code|HTTP|说明|所属模块|
|---|---|---|---|
|`UNAUTHORIZED`|401|未登录/ token 失效|通用|
|`FORBIDDEN`|403|无权限/越权|通用|
|`NOT_FOUND`|404|资源不存在|通用|
|`VALIDATION_ERROR`|422|参数校验失败|通用|
|`CONTEXT_INCOMPLETE`|422|门店配置/客群基线未完成，无法诊断|对话|
|`SCHEMA_INVALID`|500|LLM 输出 schema 校验失败已降级|对话|
|`LLM_DEGRADED`|200|LLM 降级（兜底/dry-run）|对话|
|`SAAS_STALE`|200|SaaS 数据过期|SaaS|
|`SAAS_AUTH_FAILED`|502|SaaS 鉴权失败|SaaS|
|`SAAS_RATE_LIMITED`|429|SaaS 限流|SaaS|
|`SAAS_TIMEOUT`|504|SaaS 接口超时|SaaS|
|`SAAS_CACHE_MISS`|200|缓存 miss 且 SaaS 不可用，该维度 missing\_data|SaaS|
|`GATE_BLOCKED`|422|溯源完整性 Gate 未过，禁止分配|任务|
|`CLOSE_GATE_BLOCKED`|422|关闭 Gate 未过，禁止关闭|任务|
|`GATE_OVERRIDE_REQUIRED`|422|需老板拍板或运营手动覆盖 Gate|任务|
|`STAFF_NOT_BOUND`|403|店长未通过 iLink 身份绑定|iLink|
|`STAFF_DISABLED`|403|店长账号已停用|iLink|
|`FILE_TYPE_INVALID`|422|文件类型不合法|iLink 回传|
|`FILE_TOO_LARGE`|422|文件超大小|iLink 回传|
|`FILE_LIMIT_EXCEEDED`|422|文件数量超限|iLink 回传|
|`ILINK_SESSION_EXPIRED`|503|iLink 会话过期，需重新扫码|iLink|
|`ILINK_CONTEXT_INVALID`|422|context\_token 失效，需用户激活|iLink|
|`ILINK_TIMEOUT`|504|iLink 接口超时|iLink|
|`ILINK_NOT_BOUND`|403|微信用户未绑定本系统账号|iLink|
|`BOSS_DECISION_PARSE_FAILED`|422|老板回复解析失败，转人工|对话|

# 2\. 门店管理

## 2\.1 门店列表

```Plaintext
GET /stores?page=1&size=20&keyword=
响应：
{
  "data": {
    "items": [
      { "id": "s001", "name": "门店A", "brand_name": "...", "saas_shop_id": "...",
        "position_tags": ["商务宴请"], "last_saas_cache_at": "...", "can_diagnose": true }
    ],
    "total": 5, "page": 1, "size": 20
  }
}
```

## 2\.2 门店详情

```Plaintext
GET /stores/{id}
响应：
{
  "data": {
    "id": "s001", "name": "门店A", "saas_shop_id": "...", "saas_shop_code": "...",
    "brand_name": "...", "center_id": "...",
    "position_tags": [...], "main_time_slots": [...],
    "price_min": 80, "price_max": 150, "profit_redline": 0.6,
    "baseline": { "age_groups": [...], "main_scenes": [...] },
    "redline": { "signature_dishes": [...], "profit_redline": 0.6 },
"last_saas_cache_at": "..."
  }
}
```

## 2\.3 更新门店配置（本系统维护字段）

```Plaintext
PUT /stores/{id}/config
请求：{ "position_tags": ["商务宴请"], "main_time_slots": ["晚餐"], "price_min": 80, "price_max": 150 }
响应：{ "data": { "id": "s001", "updated_at": "..." } }
```

## 2\.4 更新客群基线

```Plaintext
PUT /stores/{id}/baseline
请求：{ "age_groups": ["26-35","36-45"], "preferences": ["品质感"], "visit_freq": "月1-2次",
        "main_scenes": ["商务宴请"], "avg_spend": 120 }
响应：{ "data": { "id": "s001", "updated_at": "..." } }
```

## 2\.5 更新品牌红线

```Plaintext
PUT /stores/{id}/redline
请求：{ "signature_dish_ids": ["d001"], "untouchables": ["不能出现辣度低于XX"], "profit_redline": 0.6 }
响应：{ "data": { "id": "s001", "updated_at": "..." } }
```

## 2\.6 套餐审核配置

```Plaintext
GET /stores/{id}/combos/config
响应：{ "data": [{ "saas_combo_key": "...", "name": "双人商务套餐", "target_customer": [...],
                   "boss_target": "提毛利", "platform_groupbuy": true, "time_slot": "晚市" }] }

PUT /stores/{id}/combos/{saas_combo_key}/config
请求：{ "target_customer": ["商务宴请"], "boss_target": "提毛利", "platform_groupbuy": true, "time_slot": "晚市" }
响应：{ "data": { "saas_combo_key": "...", "updated_at": "..." } }
```

## 2\.7 数据完整度检查

```Plaintext
GET /stores/{id}/completeness
响应：
{
  "data": {
"store_config": true, "baseline": true, "redline": false, "saas_reachable": true,
        "dishes_count": 45, "dishes_with_cost": 38, "combo_count": 3, "cache_status": "hit",
    "can_diagnose": false, "missing": ["品牌红线未设置"]
  }
}
```

# 3\. SaaS 只读接入

## 3\.1 菜品只读视图（实时，走工具+缓存）

```Plaintext
GET /stores/{id}/saas/dishes?category_big=热菜&page=1&size=20
响应：
{
  "data": {
    "items": [
      { "saas_item_id": "...", "name": "宫保鸡丁", "category_big": "热菜", "category_small": "...",
        "unit": "份", "price": 38.0, "cost": 12.5, "cached_at": "..." }
    ],
    "total": 45, "cache_status": "hit"
  }
}
错误：SAAS_AUTH_FAILED | SAAS_RATE_LIMITED | SAAS_TIMEOUT | SAAS_CACHE_MISS
```

## 3\.2 套餐只读视图（实时）

```Plaintext
GET /stores/{id}/saas/combos
响应：
{
  "data": [
    { "saas_combo_key": "...", "name": "双人商务套餐", "price": 168,
      "cost": 78, "has_signature": true,
      "items": [{ "saas_item_id": "...", "name": "麻婆豆腐" }],
      "cached_at": "..." }
  ]
}
```

## 3\.3 手动刷新缓存

```Plaintext
POST /stores/{id}/saas/refresh
响应：{ "data": { "refreshed": true, "fetched_at": "..." } }
```

## 3\.4 SaaS 可达性与缓存状态

```Plaintext
GET /stores/{id}/saas/health
响应：
{
  "data": {
    "reachable": true, "cache_status": "hit",
    "last_saas_cache_at": "...", "data_freshness_hours": 0.5,
    "dishes_count": 45, "dishes_with_cost": 38, "combo_count": 3
  }
}
```

# 4\. 智能体对话

## 4\.1 新建对话

```Plaintext
POST /conversations
请求：{ "store_id": "s001", "scope": "combo" | "dish" | "all", "scope_ref_id": "cv_01", "title": "双人套餐审核" }
响应：{ "data": { "id": "conv_01", "status": "active", "created_at": "..." } }
错误：CONTEXT_INCOMPLETE
```

## 4．2 对话列表

```Plaintext
GET /conversations?store_id=s001&page=1&size=20
响应：
{
  "data": {
    "items": [
      { "id": "conv_01", "title": "...", "scope": "combo", "status": "active", "updated_at": "..." }
    ],
    "total": 3
  }
}
```

## 4．3 对话详情

```Plaintext
GET /conversations/{id}
响应：
{
  "data": {
    "id": "conv_01", "store_id": "s001", "scope": "combo", "status": "active",
    "messages": [
      { "id": "m_01", "role": "user", "content": "...", "created_at": "..." },
      { "id": "m_02", "role": "assistant", "content": "...", "structured_payload": {...},
        "trace_ids": [...], "delivery_level": "formal", "created_at": "..." }
    ],
    "battle_cards": [{ "id": "bc_01", "suggestion_type": "换菜", "operator_status": "pending" }]
  }
}
```

## 4．4 发送消息（流式）

```Plaintext
POST /conversations/{id}/messages
请求：{ "content": "/审核 双人商务套餐" }
响应：text/event-stream
事件：
  event: thinking   data: {"step":"调用 get_combos","tool":"get_combos"}
  event: trace      data: {"trace_id":"tr_01","source_type":"knowledge_base","source_ref":"R012"}
  event: content_delta data: {"text":"已为你完成..."}
  event: conclusion data: {"type":"combo_audit","payload":{...}}
  event: suggestion data: {"battle_card_id":"bc_01","payload":{...}}
  event: delivery   data: {"delivery_level":"formal","blocked_reasons":[]}
  event: done       data: {"message_id":"m_02"}
错误：CONTEXT_INCOMPLETE | SCHEMA_INVALID（降级后仍返回流，含 LLM_DEGRADED 标记）
```

## 4．5 消息溯源

```Plaintext
GET /conversations/{id}/messages/{msg_id}/traces
响应：
{
  "data": [
    { "trace_id": "tr_01", "source_type": "knowledge_base", "source_ref": "R012",
      "source_snippet": "低毛利+客群不匹配→替换", "query": "...", "confidence": "high", "called_at": "..." }
  ]
}
```

# 5\. 作战卡（优化建议）

## 5．1 从消息生成作战卡

```Plaintext
POST /conversations/{id}/battle-cards
请求：{ "message_id": "m_02", "suggestion_index": 0 }
响应：{ "data": { "id": "bc_01", "operator_status": "pending", "delivery_level": "formal" } }
```

## 5．2 更新作战卡状态

```Plaintext
PUT /battle-cards/{id}/status
请求：{ "operator_status": "confirmed" | "rejected" | "needs_boss_decision" }
响应：{ "data": { "id": "bc_01", "operator_status": "confirmed" } }
```

## 5．3 调整建议（生成新版本）

```Plaintext
POST /battle-cards/{id}/adjust
请求：{ "suggestion_desc": "...", "expected_effect": "..." }
响应：{ "data": { "id": "bc_02", "supersedes": "bc_01", "operator_status": "pending" } }
```

## 5．4 发送老板拍板

```Plaintext
POST /battle-cards/{id}/send-boss
请求：{}
响应：{ "data": { "ilink_msg_id": "...", "status": "sent", "channel": "ilink" } }
错误：ILINK_SESSION_EXPIRED | ILINK_CONTEXT_INVALID | ILINK_NOT_BOUND
```

# 6\. 任务管理

## 6．1 生成任务

```Plaintext
POST /tasks/generate
请求：{ "battle_card_id": "bc_01" }
响应：{ "data": { "id": "t_01", "status": "pending_assign" } }
错误：GATE_BLOCKED { "reasons": ["销量数据缺失"] }
```

## 6．2 任务列表

```Plaintext
GET /tasks?store_id=s001&status=pending&page=1&size=20
响应：
{
  "data": {
    "items": [
      { "id": "t_01", "battle_card_id": "bc_01", "action_type": "换菜", "assignee_staff_id": "staff_01",
        "assignee_name": "张店长", "status": "in_progress", "created_at": "..." }
    ],
    "total": 8
  }
}
```

## 6．3 任务详情

```Plaintext
GET /tasks/{id}
响应：
{
  "data": {
    "id": "t_01", "battle_card_id": "bc_01", "store_id": "s001",
    "action_type": "换菜", "action_desc": "...", "expected_effect": "...",
    "evidence_requirements": [...], "observation_period_days": 14,
    "assignee_staff_id": "staff_01", "assignee_name": "张店长", "status": "in_progress",
    "close_blocked_reasons": [],
    "executions": [{ "version_no": 1, "is_latest": true, "execution_status": "executed", ... }],
    "review": { "effect_label": "positive", ... }
  }
}
```

## 6．4 编辑任务

```Plaintext
PUT /tasks/{id}
请求：{ "assignee_staff_id": "staff_01", "observation_period_days": 14, "evidence_requirements": [...] }
响应：{ "data": { "id": "t_01", "updated_at": "..." } }
```

## 6．5 分配任务

```Plaintext
POST /tasks/{id}/assign
请求：{ "assignee_staff_id": "staff_01", "observation_period_days": 14 }
响应：{ "data": { "id": "t_01", "status": "pending", "assignee_name": "张店长" } }
错误：GATE_BLOCKED
```

## 6．6 撤销分配

```Plaintext
POST /tasks/{id}/unassign
响应：{ "data": { "id": "t_01", "status": "pending_assign", "assignee_staff_id": null } }
```

## 6．7 状态变更记录

```Plaintext
GET /tasks/{id}/status-history
响应：{ "data": [{ "status": "pending_assign", "changed_at": "...", "operator": "..." }] }
```

## 6．8 店长账号管理

```Plaintext
GET    /stores/{id}/staff                    # 门店店长账号列表
POST   /stores/{id}/staff                    # 创建店长账号（姓名/手机号）
PUT    /staff/{id}                           # 编辑店长账号（停用/换绑openid）
DELETE /staff/{id}                           # 删除店长账号

GET /tasks/{id}/access-logs
响应：{ "data": [{ "action": "open", "success": true, "ip": "...", "user_agent": "...", "accessed_at": "..." }] }
```

# 7\. iLink 店长端（任务通知与执行回传）

> V2.0 这一期不做小程序。店长通过微信 iLink Bot 接收任务通知并以对话方式回传执行情况（发图片 \+ 文字）。智能体解析自然语言回传为结构化执行回传，存入 task\_execution。以下为运营后台与 iLink 网关内部接口。

## 7．1 iLink 主动推送任务通知

```Plaintext
POST /ilink/push-task/{task_id}
说明：运营分配任务后触发，智能体网关内部调用 iLink 网关推送到店长微信
响应：{ "data": { "sent": true, "ilink_msg_id": "...", "to_weixin_user_id": "..." } }
错误：ILINK_SESSION_EXPIRED | ILINK_NOT_BOUND | ILINK_CONTEXT_INVALID
```

## 7．2 iLink 消息 Webhook（接收店长回传）

```Plaintext
POST /ilink/webhook
说明：iLink 网关收到店长微信消息（文字 + 图片）后回调智能体网关，智能体将自然语言解析为结构化执行回传
请求：{
  "weixin_user_id": "wx_user_01",
  "msg_type": "text" | "image",
  "content": "已执行，这周31份毛利55%",
  "images": [{ "cdn_url": "https://cdn...", "file_size": 102400 }],
  "context_token": "...",
  "received_at": "2026-06-19T10:30:00Z"
}
响应：{ "data": { "parsed": true, "task_execution_id": "te_02", "execution_status": "executed", "reply_to_user": "已收到回传，已记录" } }
错误：STAFF_NOT_BOUND | STAFF_DISABLED | FILE_TYPE_INVALID | FILE_TOO_LARGE
说明：智能体解析失败时 parsed=false，标记"待人工复核"，运营在复核页手动补录
```

## 7．3 任务执行回传历史（运营端）

```Plaintext
GET /tasks/{id}/executions
响应：{ "data": [{ "task_execution_id": "te_02", "version_no": 2, "is_latest": true, "execution_status": "executed", "executor_name": "张店长", "submitted_at": "...", "raw_text_preview": "已执行，这周31份...", "parsed": true }] }

GET /tasks/{id}/executions/{exec_id}
响应：{ "data": { "task_execution_id": "te_02", "version_no": 2, "execution_status": "executed", "actual_action": "...", "raw_text": "已执行，换了蟹粉豆腐，这周31份毛利55%", "images": [...], "parsed": true } }

POST /tasks/{id}/executions/{exec_id}/manual-review
说明：人工复核智能体解析失败的回传，运营手动补录结构化字段
请求：{ "execution_status": "executed", "executor_name": "张店长", "actual_action": "...", "result_value": 31 }
响应：{ "data": { "task_execution_id": "te_02", "parsed": true, "manually_reviewed": true } }
```

# 8\. 执行复核

## 8．1 获取复核数据

```Plaintext
GET /tasks/{id}/review
响应：
{
  "data": {
    "latest_execution": { "version_no": 2, "execution_status": "executed", "baseline_value": 23, "result_value": 31 },
    "executions": [...],
    "before_after_data": { "销量(周均)": { "before": 23, "after": 31, "change": "+34.8%" } },
    "data_gap": "no_gap",
    "images": [{ "evidence_id": "ee_01", "access_url": "..." }]
  }
}
```

## 8．2 提交复核结论

```Plaintext
PUT /tasks/{id}/review
请求：{ "effect_label": "positive" | "negative" | "uncertain", "data_gap": "no_gap" | "observation_pending" | "data_missing" }
响应：{ "data": { "id": "t_01", "status": "completed", "closed": true } }
错误：CLOSE_GATE_BLOCKED { "reasons": ["请上传执行图片"] }
```

# 9\. 老板反馈

```Plaintext
POST /feedbacks
请求：{
  "ref_type": "battle_card" | "task_execution",
  "ref_id": "bc_01",
  "feedback_type": "agree" | "disagree" | "revision" | "satisfied" | "unsatisfied" | "new_request",
  "content": "...",
  "is_boss_decision": true,
  "boss_decision_content": "保留麻婆豆腐，改为小份"
}
响应：{ "data": { "id": "f_01", "created_at": "..." } }

GET /feedbacks?store_id=s001&type=agree&page=1&size=20
响应：{ "data": { "items": [...], "total": 12 } }

GET /feedbacks/{id}
响应：{ "data": { "id": "f_01", "ref_type": "...", "content": "...", "is_boss_decision": true } }
```

# 10\. 规则 / 案例 / 红线 / 品牌资料

## 10．1 规则卡（支持门店筛选）

```Plaintext
GET /rules?store_id=s001&status=verified&page=1&size=20
说明：store_id 可选，不传返回全部（全局 + 各门店专属）；传则返回全局 + 指定门店
响应：{ "data": { "items": [{ "id":"...","rule_no":"R-012","store_id":null,"content":"...",
  "trigger_condition":"...","default_judgment":"...","risk_type":"profit","required_data":[...],
  "migration_boundary":"...","version":"v1.0","status":"verified","positive_count":5,"source":"expert" }], "total": 30 } }

POST /rules
请求：{ "store_id":"s001","rule_no":"R-013","content":"...","risk_type":"profit","version":"v1.0" }
说明：store_id 可空（null=全局规则），非空=门店专属规则
响应：{ "data": { "id":"...","rule_no":"R-013" } }

PUT /rules/{id}
PUT /rules/{id}/status
```

## 10．2 案例卡

```Plaintext
GET /cases?store_id=s001&page=1&size=20
响应：{ "data": { "items": [...], "total": 8 } }

POST /cases/from-task/{task_id}
请求：{ "applicable_scene":"晚市 商务宴请","reusable_conclusion":"...","non_migration_boundary":"..." }
响应：{ "data": { "id":"...","case_no":"C-009" } }
```

## 10．3 红线（支持门店筛选）

```Plaintext
GET /redlines?store_id=s001&status=active
说明：store_id 可选，不传返回全部；传则返回全局（store_ids=NULL）+ 包含该门店的红线
响应：{ "data": [{ "id":"...","redline_no":"RL-001","store_ids":["s001","s002"],"content":"整体菜单毛利率不得低于45%",
  "forbidden_actions":[...],"scope":"晚市","status":"active" }] }

POST /redlines
请求：{ "store_ids":["s001","s002"],"content":"...","forbidden_actions":[...],"required_actions":[...] }
说明：store_ids 可空（null=全局红线）

PUT /redlines/{id}
```

## 10．4 品牌资料库

```Plaintext
GET /knowledge?store_id=s001&category=menu_guide&page=1&size=20
说明：store_id 可选，不传返回全部（品牌级 + 各门店级）；传则返回品牌级（store_id=NULL）+ 该门店级
响应：{ "data": { "items": [{ "id":"...","store_id":"s001","title":"门店A出品规范","category":"production_spec",
  "content":"...","tags":["出品","摆盘"],"version":"v1.0","status":"active" }], "total": 15 } }

POST /knowledge
请求：{ "store_id":"s001","title":"门店A出品规范","category":"production_spec","content":"...","tags":["出品"] }
说明：store_id 可空（null=品牌级资料），非空=门店级资料

PUT /knowledge/{id}
请求：{ "title":"...","content":"...","status":"archived" }
```

# 11\. 报告导出

```Plaintext
POST /reports/export
请求：{
  "store_id": "s001",
  "report_type": "conversation" | "task" | "full",
  "format": "pdf" | "markdown" | "json",
  "conversation_id": "conv_01",
  "task_ids": ["t_01"]
}
响应：
{
  "data": {
    "delivery_level": "formal_report" | "preview_report",
    "blocked_reasons": [],
    "download_url": "https://...",
    "filename": "conversation_formal_s001_20260619.pdf"
  }
}
```

# 12\. 微信 iLink Bot 接入

> 自研智能体通过微信 iLink Bot API 接入微信个人号，老板/店长在微信直接和 AI 助手对话。详细见《微信 iLink Bot 接入与多端对话详细设计》。iLink 协议由网关层封装，以下为运营后台管理与网关内部接口。

```Plaintext
GET  /ilink/status                  # iLink Bot 登录状态（active/expired）
响应：{ "data": { "status": "active", "bot_account_id": "...", "logged_in_at": "..." } }

POST /ilink/qrcode                  # 生成登录二维码（失效时重新扫码）
响应：{ "data": { "qrcode_url": "...", "expires_at": "..." } }

GET  /ilink/bindings                # 绑定列表（weixin_user_id ↔ 账号）
响应：{ "data": [{ "id":"...","weixin_user_id":"...","role":"boss","ref_id":"...","phone":"...","bound_at":"..." }] }

DELETE /ilink/bindings/{id}         # 解除绑定
响应：{ "data": { "deleted": true } }

# 智能体网关 → iLink 网关（内部）
POST /internal/ilink/push
请求：{ "to_weixin_user_id":"wx_user_01","msg_type":"text"|"boss_decision","content":{ "text":"..." }|{ "battle_card_id":"bc_01","suggestion":"..." } }
响应：{ "data": { "sent": true, "ilink_msg_id": "..." } }
错误：ILINK_SESSION_EXPIRED | ILINK_CONTEXT_INVALID | ILINK_TIMEOUT
```

## 12\.1 店长微信对话（复用智能体网关，通过 iLink Webhook 接入）

```Plaintext
# 店长通过微信 iLink Bot 发消息，iLink 网关回调智能体网关
# 智能体网关内部创建/查找 conversation，走统一 Agent Loop 处理
# message.channel = "ilink"，role = "user" 或 "assistant"
# 无需独立 API 端点，复用第 4 节对话接口的逻辑层
```

# 13\. 接口清单速查

|模块|方法|路径|
|---|---|---|
|门店|GET|/stores|
|门店|GET|/stores/{id}|
|门店|PUT|/stores/{id}/config|
|门店|PUT|/stores/{id}/baseline|
|门店|PUT|/stores/{id}/redline|
|门店|GET|/stores/{id}/combos/config|
|门店|PUT|/stores/{id}/combos/{combo\_view\_id}/config|
|门店|GET|/stores/{id}/completeness|
|SaaS|GET|/stores/{id}/saas/dishes|
|SaaS|GET|/stores/{id}/saas/combos|
|SaaS|POST|/stores/{id}/saas/refresh|
|SaaS|GET|/stores/{id}/saas/health|
|对话|POST|/conversations|
|对话|GET|/conversations|
|对话|GET|/conversations/{id}|
|对话|POST|/conversations/{id}/messages（SSE）|
|对话|GET|/conversations/{id}/messages/{msg\_id}/traces|
|作战卡|POST|/conversations/{id}/battle-cards|
|作战卡|PUT|/battle-cards/{id}/status|
|作战卡|POST|/battle-cards/{id}/adjust|
|作战卡|POST|/battle-cards/{id}/send-boss|
|任务|POST|/tasks/generate|
|任务|GET|/tasks|
|任务|GET|/tasks/{id}|
|任务|PUT|/tasks/{id}|
|任务|POST|/tasks/{id}/assign|
|任务|POST|/tasks/{id}/unassign|
|任务|GET|/tasks/{id}/status-history|
|任务|GET|/tasks/{id}/access-logs|
|店长|GET|/stores/{id}/staff|
|店长|POST|/stores/{id}/staff|
|店长|PUT|/staff/{id}|
|店长|DELETE|/staff/{id}|
|iLink推送|POST|/ilink/push-task/{task\_id}|
|iLink回传|POST|/ilink/webhook|
|回传历史|GET|/tasks/{id}/executions|
|回传详情|GET|/tasks/{id}/executions/{exec\_id}|
|回传人工复核|POST|/tasks/{id}/executions/{exec\_id}/manual-review|
|复核|GET|/tasks/{id}/review|
|复核|PUT|/tasks/{id}/review|
|反馈|POST|/feedbacks|
|反馈|GET|/feedbacks|
|反馈|GET|/feedbacks/{id}|
|规则|GET|/rules|
|规则|POST|/rules|
|规则|PUT|/rules/{id}|
|规则|PUT|/rules/{id}/status|
|案例|GET|/cases|
|案例|POST|/cases/from-task/{task\_id}|
|红线|GET|/redlines|
|红线|POST|/redlines|
|红线|PUT|/redlines/{id}|
|品牌资料|GET|/knowledge|
|品牌资料|POST|/knowledge|
|品牌资料|PUT|/knowledge/{id}|
|报告|POST|/reports/export|
|iLink|GET|/ilink/status|
|iLink|POST|/ilink/qrcode|
|iLink|GET|/ilink/bindings|
|iLink|DELETE|/ilink/bindings/{id}|

# 14\. 开发约定

1. **所有列表接口**：统一 `page`/`size` 分页，返回 `items`/`total`。
2. **写操作**：返回更新后的资源或 `updated_at`，便于前端同步。
3. **iLink 店长端鉴权**：店长通过 iLink 微信 Bot 接入，身份由 `ilink_binding` 表绑定（weixin_user_id ↔ store_staff），网关层按 role + ref_id 路由。任务归属校验：任务回传时校验 store_staff.store_id == task.store_id，越权拒绝。
4. **SSE 接口**：`POST /conversations/{id}/messages` 返回 `text/event-stream`，断线重连由前端用 `last_event_id` 续传。
5. **错误码**：业务 Gate 类错误用 422，鉴权 401，越权 403，限流 429，SaaS 上游错误 502/504。
6. **幂等**：`POST /tasks/generate` 同一 battle\_card 重复调用返回已存在的 task（幂等），不重复创建。
7. **审计**：所有写操作记录操作人/时间，敏感操作（Gate 覆盖/店长账号停用/规则变更）记审计日志。
8. **版本化**：接口前缀 `/api/v1`，破坏性变更升 `/api/v2`。
