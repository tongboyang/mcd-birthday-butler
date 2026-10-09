# MCP_INTEGRATION.md —— 麦当劳 MCP 集成说明

本文档说明 McdBirthdayButler 实际使用的麦当劳 MCP Server 与 Tools、完整调用流程、每一步的业务价值，以及安全与限流约束。

---

## 1. MCP Server 概况

| 项目 | 内容 |
|---|---|
| Server 名称 | mcd-mcp（麦当劳中国官方 MCP Server） |
| 接入地址 | `https://mcp.mcd.cn` |
| 传输协议 | Streamable HTTP |
| 鉴权方式 | 请求头 `Authorization: Bearer <MCP_TOKEN>`（Token 经麦当劳开放平台申请） |
| 限流规则 | 每个 Token 600 次/分钟，超限返回 429 |
| 版本依据 | v1.0.7（2026-07-29）新增派对、品鉴会等主题活动查询、预约与下单功能 |

配置示例（Token 使用占位符，真实 Token 不入库）：

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "streamablehttp",
      "url": "https://mcp.mcd.cn",
      "headers": {
        "Authorization": "Bearer YOUR_MCP_TOKEN"
      }
    }
  }
}
```

## 2. 实际使用的 Tools 一览

| # | Tool | 角色 | 说明 |
|---|---|---|---|
| 0 | `mall-points-products` | 前置 | 查询可兑换/可购买商品列表，从中确定派对主题商品的 `spuId` |
| 0 | `mall-product-detail` | 前置 | 查看派对商品详情，获取 `skuId`、`partyType`（1 包场 / 2 拼团 / -1 待确认）、确认 `shopId=5` 才支持下单 |
| 1 | `query-party-city` | 主链 | 查询主题活动覆盖城市列表，确认目标城市是否支持 |
| 2 | `query-party-store` | 主链 | 查询指定城市下可办生日会的门店列表 |
| 3 | `query-party-store-date` | 主链 | 查询指定门店可预约日期列表 |
| 4 | `query-party-store-session` | 主链 | 查询指定门店指定日期的场次（时间、价格、余位、成团人数） |
| 5 | `party-order-create` | 主链 | 创建派对订单（仅在用户明确确认后调用） |
| 辅 | `now-time-info` | 辅助 | 获取服务器当前时间，校验用户所给日期的合法性 |
| 辅 | `calculate-price` | 辅助 | 核验加购餐品价格（返回单位为"分"，展示需 ÷100） |

## 3. 完整调用流程

```
                    ┌────────────────────────────┐
                    │ Step 0  派对商品确认        │
                    │ mall-points-products       │
                    │   └→ 选中生日会 SPU (spuId) │
                    │ mall-product-detail        │
                    │   └→ skuId / partyType /   │
                    │      shopId(必须=5)        │
                    └────────────┬───────────────┘
                                 ▼
                    ┌────────────────────────────┐
                    │ Step 0.5 日期校验          │
                    │ now-time-info              │
                    │   └→ 排除过去日期/提示备选  │
                    └────────────┬───────────────┘
                                 ▼
        ┌────────────────────────────────────────────────┐
        │ Step 1  query-party-city(spuId)                │
        │   业务价值：确认"这个城市能不能办"，避免空跑    │
        │   产出：城市 code + latitude/longitude          │
        └────────────────────┬───────────────────────────┘
                             ▼
        ┌────────────────────────────────────────────────┐
        │ Step 2  query-party-store(code, spuId, 经纬度) │
        │   业务价值：把"全城能办生日会的店"收敛为候选集  │
        │   产出：storeCode 列表 + 门店地址               │
        └────────────────────┬───────────────────────────┘
                             ▼
        ┌────────────────────────────────────────────────┐
        │ Step 3  query-party-store-date(storeCode,spuId)│
        │   业务价值：锁定"哪天有"，替代逐店打电话        │
        │   产出：dateStr（目标月份可约日期）             │
        └────────────────────┬───────────────────────────┘
                             ▼
        ┌────────────────────────────────────────────────┐
        │ Step 4  query-party-store-session(storeCode,   │
        │              dateStr, spuId)                   │
        │   业务价值：拿到可对比的"场次 × 价格 × 余位"    │
        │   产出：场次 id、timeStart/timeEnd、price、     │
        │         leftNum、partyMin/partyMax             │
        └────────────────────┬───────────────────────────┘
                             ▼
        ┌────────────────────────────────────────────────┐
        │ Step 5  用户决策 + 订单确认单（人工确认红线）   │
        │   表格对比 2-3 方案 + 管家点评 → 展示完整订单   │
        │   摘要 → 必须获得用户明确肯定指令               │
        └────────────────────┬───────────────────────────┘
                             ▼  仅在确认后
        ┌────────────────────────────────────────────────┐
        │ Step 6  party-order-create(spuId, skuId,       │
        │            partyType, code, storeCode, dateStr,│
        │            id, timeStart, timeEnd, leftNum,    │
        │            count)                              │
        │   业务价值：对话内直接锁场，全流程闭环          │
        └────────────────────┬───────────────────────────┘
                             ▼
        ┌────────────────────────────────────────────────┐
        │ Step 7（可选） calculate-price 核验加购餐品价格 │
        └────────────────────────────────────────────────┘
```

### 每步参数速查

| Tool | 必填参数 | 参数来源 |
|---|---|---|
| `mall-points-products` | — | — |
| `mall-product-detail` | 商品 ID | 上一步返回 |
| `query-party-city` | `spuId` | Step 0 |
| `query-party-store` | `code`；建议附 `latitude/longitude/spuId` | query-party-city 返回 |
| `query-party-store-date` | `storeCode`、`spuId` | query-party-store / Step 0 |
| `query-party-store-session` | `storeCode`、`dateStr`、`spuId` | query-party-store / query-party-store-date |
| `party-order-create` | `spuId`、`skuId`、`partyType`、`code`、`storeCode`、`dateStr`、`id`、`timeStart`、`timeEnd`、`leftNum`、`count` | 全部来自前序步骤真实返回，原样透传 |
| `now-time-info` | — | — |
| `calculate-price` | `storeCode`、`orderType`、`beType`、`items[]` | 门店信息 + 用户选品 |

## 4. 业务价值分析

| 工具 | 解决的传统痛点 | 管家化增量价值 |
|---|---|---|
| `query-party-city` | 家长不知道"我家这边能不能办" | 一步确认覆盖城市，不支持时主动推荐最近支持城市 |
| `query-party-store` | 逐店打电话问"你们这办生日会吗" | 一次拉出全城可办门店，按城区/距离预筛选 |
| `query-party-store-date` | "哪天有空"全靠现场问 | 整月可约日期一屏掌握，节假日档期早知道 |
| `query-party-store-session` | 场次、价格、剩余名额信息分散 | 场次 × 价格 × 余位结构化呈现，支撑多方案对比 |
| `party-order-create` | 要到店或电话才能锁场 | 对话内确认即下单，全程不离开聊天窗口 |
| `now-time-info` | 模型时间感知不可靠，易推荐过期日期 | 每次规划先校验日期合法性，杜绝无效查询 |
| `calculate-price` | 加购餐品价格算不清 | 下单前精确核价（注意"分"→"元"换算），避免结账争议 |
| `mall-points-products` / `mall-product-detail` | 派对主题、套餐内容不透明 | 让用户先选主题再查门店，且前置校验 `shopId=5`、`partyType`，减少无效下单 |

**整体价值**：把"家长需要跑 2-3 家门店、打 N 个电话、信息记在脑子里对比"的传统流程，压缩为一轮结构化对话——这正是麦当劳 MCP 派对工具族 + 管家式 Skill 组合的差异化所在。

## 5. 安全与工程约束

1. **下单确认红线**：`party-order-create` 是唯一产生真实交易的工具。调用前必须展示完整订单摘要并获得用户明确肯定指令（"确认下单"等）；"挺好的""可以"等模糊回应不构成确认。
2. **数据透传**：下单参数全部来自前序工具的真实返回值，禁止由模型记忆改写或补全，避免错场/错价。
3. **限流控制**：600 次/分钟限额下，候选门店的日期/场次查询收敛到 2-3 家重点门店，不做全量轮询；429 时降速重试并告知用户。
4. **Token 安全**：MCP Token 仅存于连接器配置或环境变量占位符，绝不写入仓库与对话输出。
5. **错误兜底**：401（Token 失效）→ 提示用户重新配置连接器；工具返回空 → 如实告知"暂无场次"并给出换日期/换门店的替代路径。
6. **隐私边界**：不收集、不存储儿童及家长的真实个人信息，下单仅依赖 MCP 账号体系内的既有信息。
