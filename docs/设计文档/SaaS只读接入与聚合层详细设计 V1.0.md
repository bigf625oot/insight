# SaaS 只读接入详细设计 V2\.0（工具 \+ 缓存架构）

> 本文档是《产品需求方案 V2\.0》中"SaaS 数据只读接入"模块的开发前详细设计。**V2\.0 架构变更：不再持久化主数据，不再做全量/增量同步任务**。改为把 SaaS 接口包装为智能体工具（skill），智能体对话时按需调用，聚合结果走 Redis 短期缓存。所有主数据从五乡 cysms SaaS 系统只读获取，本系统不回写。

# 1\. 设计目标与边界

1. **只读不回写**：本系统对 SaaS 数据只拉取不修改。
2. **不持久化主数据**：菜品/成本/销量/套餐构成不落库，按需从 SaaS 拉取 \+ 实时聚合 \+ 短期缓存。
3. **包装为智能体工具（skill）**：SaaS 接口封装为 `get_dishes`/`get_sales`/`get_combos` 等工具，智能体对话时按需调用，数据即工具调用结果。
4. **短期缓存减压**：Redis 缓存聚合结果，避免重复拉取同一门店流水，降低 SaaS 压力与对话延迟。
5. **可降级**：SaaS 不可用时缓存命中仍可服务；缓存也 miss 则该维度标 `missing_data`，结论降级 preview（溯源完整性 Gate 兜底）。
6. **可扩展**：预留字典接口扩展位（SaaS 若提供菜品/门店字典接口，工具优先调字典）。

# 2\. SaaS 接口规约

## 2\.1 接口清单

|接口|方法|URL|用途|
|---|---|---|---|
|流水查询|POST|`https://cysms.wuuxiang.com/api/datatransfer/getserialdata`|获取门店交易流水（含菜品明细/成本/销量/套餐/支付）|

> 【待确认】SaaS 是否提供独立菜品/门店/成本字典接口。当前以流水接口为主，第 9 节预留字典扩展位。

## 2\.2 鉴权

请求 Header：`access_token` / `accessid` / `granttype`(固定 `client`)。`access_token` 过期由 `SaaSTokenManager` 刷新（见 8\.2）。鉴权失败（401）刷新 token 重试 1 次。

## 2\.3 请求参数

|参数|必填|说明|
|---|---|---|
|`centerId`|是|集团/品牌 ID|
|`shopId`|是|门店 ID|
|`settleDate` / `beginDate`+`endDate`|是|结算日期或区间 `yyyy-MM-dd`|
|`pageNo` / `pageSize`|是|分页，pageSize 上限 500|
|`dateType`|否|1 营业日 / 2 自然日等|
|`needPkgDetail`|否|1 返回套餐明细（拉套餐必传）|

## 2\.4 限流与分页

- `pageSize` 上限 500，适配器自动分页至 `pageInfo.pageTotal` 结束。
- 限流：单门店同时 1 个拉取（按 store\_id 加锁），门店间并行全局上限 5；429 指数退避重试。

## 2\.5 返回结构

```Plaintext
{ "code": "0", "msg": "success",
  "data": { "billList": [ Bill ], "pageInfo": {...} } }
```

### Bill 关键字段

- 账单级：`bs_id` / `shop_id` / `shop_name` / `brand_name` / `open_time` / `settle_time` / `settle_biz_date` / `people_qty` / `order_type` / `dinner_type_name` / 会员
- `item[]`：`item_id` / `item_name` / `item_code` / `cost_price`(成本) / `last_price`(售价) / `last_qty`(数量) / `last_total` / `big_class_name` / `small_class_name` / `unit_name` / `pkg_flg`(0非套餐/1套餐主菜/2套餐明细) / `pkg_sc_id`(套餐明细父主菜) / `sc_id`

# 3\. 架构

```Plaintext
┌─────────────────────────────────────────────────┐
│                   智能体（Agent）                │
└───────────────────────┬─────────────────────────┘
                        │ 调用工具（skill）
                        ↓
┌─────────────────────────────────────────────────┐
│        智能体工具层（SaaS Skills）               │
│  get_dishes / get_sales / get_combos            │
└───────┬───────────────────────────────┬─────────┘
        │ 缓存命中？                     │ miss
        ↓                               ↓
┌──────────────────┐            ┌──────────────────┐
│   Redis 缓存层    │            │  SaaSAggregator  │
│ (聚合结果短期缓存)│←─写入──────│ (实时从流水聚合)  │
└──────────────────┘            └────────┬─────────┘
                                         │ 拉流水
                                         ↓
                                ┌──────────────────┐
                                │   SaaSAdapter    │
                                │ (鉴权/分页/重试)  │
                                └────────┬─────────┘
                                         │ HTTPS
                                         ↓
                                ┌──────────────────┐
                                │  五乡 cysms SaaS  │
                                └──────────────────┘
```

- **SaaSAdapter**：封装 SaaS 接口，鉴权/分页/重试/限流，返回原始 Bill 列表。
- **SaaSAggregator**：实时从 Bill 聚合出菜品/销量/套餐业务视图（不落库，返回给工具）。
- **Redis 缓存层**：缓存聚合结果，工具优先读缓存。
- **智能体工具层（SaaS Skills）**：包装为智能体可调用的工具，记 trace。

> 本系统**不存** dish\_view / dish\_sales\_view / combo\_view 等主数据表，**无全量/增量同步任务**。

# 4\. 字段映射与聚合算法

聚合由 `SaaSAggregator` 在内存中完成，结果返回工具（并写缓存），不落库。

## 4\.1 菜品视图聚合

|输出字段|SaaS 来源|聚合规则|
|---|---|---|
|saas\_item\_id|`item.item_id`|distinct|
|name|`item.item_name`|最近一条|
|category\_big / category\_small|`big_class_name` / `small_class_name`|distinct|
|unit|`unit_name`|distinct|
|price|`last_price`|近 30 天出现频次最高（众数，避免单次改价干扰）|
|cost|`cost_price`|最近一条含成本价的流水；空则"成本缺失"|

## 4\.2 销量聚合

- `quantity` = sum(`last_qty`) by `item_id` by 周期
- **仅统计 `pkg_flg=0`(非套餐) 与 `pkg_flg=1`(套餐主菜)**；`pkg_flg=2`(套餐明细) 不计入单品销量，避免重复

## 4\.3 套餐识别算法

```Plaintext
1. item[] 中 pkg_flg=1 → 套餐主菜（代表一个套餐），其 sc_id 为套餐标识 saas_combo_key
2. 同 bills 内 pkg_flg=2 且 pkg_sc_id == 主菜.sc_id → 该套餐明细
3. 同一 saas_combo_key 跨多条流水：
   - name = 主菜 item_name（distinct）
   - price = 主菜 last_price 众数
   - 构成 = distinct(明细 item_id → item_name)
   - cost = sum(明细 cost_price)
   - 销量 = sum(主菜 last_qty)
4. 构成跨流水不一致时，以最近一次为准
```

> 拉套餐数据工具必须传 `needPkgDetail=1`，否则套餐明细缺失。

# 5\. 智能体工具（SaaS Skills）定义

工具供智能体 Agent Loop 调用，每次调用记 `message_trace`(source\_type=saas\_api)。

|工具|入参|出参|缓存 TTL|
|---|---|---|---|
|`get_dishes`|store\_id, \[category\]|菜品视图列表|24h|
|`get_sales`|store\_id, item\_id, period|销量序列|4h|
|`get_combos`|store\_id|套餐构成 \+ 价格|24h|
|`get_store_config`|store\_id|门店配置 \+ 客群基线 \+ 红线（本系统表，不走 SaaS）|\-|

```Python
class SaaSSkills:
    def __init__(self, adapter: SaaSAdapter, aggregator: SaaSAggregator, cache: RedisCache):
        self.adapter = adapter
        self.aggregator = aggregator
        self.cache = cache

    async def get_dishes(self, store_id: str, category: str = None) -> list[DishView]:
        key = f"saas:dishes:{store_id}:{category or 'all'}"
        cached = await self.cache.get(key)
        if cached:
            return cached
        bills = await self.adapter.fetch_serial_data(store_id, days=90, need_pkg_detail=True)
        dishes = self.aggregator.aggregate_dishes(store_id, bills)
        if category:
            dishes = [d for d in dishes if d.category_big == category]
        await self.cache.set(key, dishes, ttl=86400)
        return dishes

    async def get_sales(self, store_id, item_id, period) -> list[SalesView]: ...
    async def get_combos(self, store_id) -> list[ComboView]: ...
```

> 工具是智能体的"数据 skill"：智能体决定何时取数、取什么，SaaSAdapter/Aggregator 负责拉取聚合，缓存负责加速。

# 6\. 缓存策略

## 6\.1 缓存设计

- 存储：Redis
- Key：`saas:{type}:{store_id}:{params_hash}`
- Value：聚合结果（JSON）
- TTL：菜品/套餐 24h，销量 4h

## 6\.2 失效策略

|场景|失效处理|
|---|---|
|TTL 到期|自然过期，下次调用重新拉取|
|运营手动刷新|门店详情页"刷新数据"按钮 → 删除该门店所有 `saas:*` 缓存|
|SaaS 数据变更|最迟缓存 TTL 后生效（菜品/套餐 24h，销量 4h）|

## 6\.3 缓存击穿保护

- 单门店同时只 1 个拉取（按 store\_id 加锁），避免并发对话重复打 SaaS。
- 拉取进行中，其他请求等锁复用结果。

# 7\. 数据时效与降级

|缓存状态|delivery 影响|
|---|---|
|缓存命中（未过期）|formal 可达，溯源标 saas\_api（带 synced\_at=缓存写入时间）|
|缓存 miss，SaaS 拉取成功|formal 可达，刷新缓存|
|缓存 miss，SaaS 失败|该维度标 missing\_data，结论降级 preview，溯源标 SAAS\_STALE|
|cost\_price 缺失|该菜品"成本缺失"，毛利维度 missing\_data|

> 运营在门店详情看菜品/销量只读视图：实时调工具（带缓存），不查本地表。

# 8\. 异常处理与降级

|异常|检测|降级|
|---|---|---|
|鉴权失败(401)|HTTP|刷新 token 重试 1 次；仍失败→工具返回错误，维度 missing\_data|
|限流(429)|HTTP|指数退避(1s/2s/4s)重试 3 次；仍失败→缓存兜底或 missing\_data|
|接口超时|30s|重试 1 次；仍失败→缓存兜底或 missing\_data|
|接口 code=-1|响应体|工具返回错误，维度 missing\_data|
|cost 缺失|cost=0/空|留空，毛利 missing\_data|
|套餐明细缺失|needPkgDetail 未传|套餐审核降级 preview|
|数据质量异常(销量负等)|聚合校验|跳过该条，记录告警|

# 9\. 字典接口扩展位

若 SaaS 提供字典接口，工具优先调字典，无则回退流水聚合。配置项 `use_dict_api`(默认 false)。

```Python
class SaaSAdapter:
    async def fetch_dish_dict(self, center_id, shop_id) -> list[DishDict] | None:
        """字典接口，不存在返回 None，回退流水聚合"""
    async def fetch_shop_dict(self, center_id) -> list[ShopDict] | None: ...
```

# 10\. 配置管理

|配置项|说明|
|---|---|
|`saas.base_url`|`https://cysms.wuuxiang.com/api/datatransfer/getserialdata`|
|`saas.accessid` / `access_token` / `token_refresh_url`|鉴权（token 刷新【待确认】）|
|`saas.default_center_id`|默认集团 ID|
|`saas.fetch_concurrency`|全局拉取并发上限，默认 5|
|`saas.fetch_timeout_s`|单次拉取超时，默认 30|
|`cache.ttl_dishes` / `cache.ttl_combos`|菜品/套餐缓存 TTL，默认 86400|
|`cache.ttl_sales`|销量缓存 TTL，默认 14400|
|`saas.use_dict_api`|是否启用字典接口，默认 false|

## 10\.1 Token 管理

- `SaaSTokenManager` 维护 token 生命周期，即将过期(<1h)自动刷新。
- 刷新失败告警；【待确认】SaaS token 刷新接口，若无则人工配长效 token。

## 10\.2 门店与 centerId 映射

- `store.saas_shop_id` 映射 SaaS shopId，`store.center_id` 映射 centerId。
- 运营在门店配置绑定 SaaS shopId（从 `fetch_shops` 拉取列表选择）。

# 11\. 接口契约

> 去掉了原同步/视图接口（无本地主数据表）。门店只读视图实时调工具（带缓存）。

```Plaintext
# 门店菜品只读视图（实时，走工具+缓存）
GET /stores/{id}/saas/dishes?category=热菜
响应：{ "data": [{ "saas_item_id":"...","name":"宫保鸡丁","price":38.0,"cost":12.5,"synced_at":"缓存时间" }] }

# 门店套餐只读视图（实时）
GET /stores/{id}/saas/combos
响应：{ "data": [{ "saas_combo_key":"...","name":"双人商务套餐","price":168,"cost":78,"items":[...] }] }

# 手动刷新缓存（运营）
POST /stores/{id}/saas/refresh
响应：{ "data": { "refreshed": true, "fetched_at":"..." } }

# SaaS 可达性检查（数据完整度用）
GET /stores/{id}/saas/health
响应：{ "data": { "reachable": true, "dishes_count":45, "dishes_with_cost":38, "combo_count":3, "cache_status":"hit" } }
```

## 11\.1 错误码

|code|说明|
|---|---|
|`SAAS_AUTH_FAILED`|SaaS 鉴权失败|
|`SAAS_RATE_LIMITED`|SaaS 限流|
|`SAAS_TIMEOUT`|SaaS 接口超时|
|`SAAS_STALE`|数据过期/缺失，结论 preview|

# 12\. 监控与告警

|监控项|阈值|告警|
|---|---|---|
|SaaS 拉取失败率|\> 10%|告警|
|单门店缓存 miss 率高（频繁回源）|小时 \> 5 次|告警（可能缓存失效过快或对话高频）|
|Token 即将过期 < 1h|是|自动刷新，失败告警|
|429 限流频次|小时 \> 10 次|告警|
|cost 覆盖率 < 70%|是|告警（运行时统计）|

# 13\. 开发注意事项

1. **不建主数据表**：菜品/销量/套餐不落库，工具实时聚合 \+ 缓存。
2. **套餐明细必须传 needPkgDetail=1**。
3. **销量不重复计算**：pkg\_flg=2 明细不计入单品销量。
4. **price 取众数非均值**；cost 缺失不臆造。
5. **缓存 key 带 params\_hash**：不同周期/分类的销量/菜品分别缓存。
6. **单门店拉取加锁**：防缓存击穿，并发请求复用结果。
7. **工具必须记 trace**：每次 SaaS 工具调用记 message\_trace(source\_type=saas\_api)，含 synced\_at。
8. **降级走 Gate**：SaaS 失败 → missing\_data → 溯源完整性 Gate 降级 preview，不中断对话。
9. **时区**：settle\_biz\_date 为营业日，本系统统一 Asia/Shanghai。
10. **门店只读视图实时**：运营看菜品/销量页面 = 调工具（带缓存），非查本地表。
