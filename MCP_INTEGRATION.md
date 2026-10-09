# 麦当劳 MCP 集成说明

## 1. 使用的 MCP Server

| 项目 | 值 |
| --- | --- |
| Server 名称 | `mcd-mcp`（官方托管） |
| 接入地址 | `https://mcp.mcd.cn` |
| 传输协议 | Streamable HTTP |
| 鉴权 | 请求头 `Authorization: Bearer <MCP_TOKEN>` |
| 协议版本 | `2025-03-26`（initialize 握手实测可用） |
| 限流 | 600 次/分钟/Token，本 Skill 单次生成黄历 ≤ 8 次调用 |

配置示例：

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

> Token 由用户自行申请并保管，本仓库不包含任何真实 Token。

## 2. 实际使用的 Tool

黄历生成主流程（8 个只读 Tool，全部经过真实调用验证）：

| # | Tool | 用途 | 取用的关键字段 |
| --- | --- | --- | --- |
| 1 | `now-time-info` | 确定"今天"与时辰语境 | `data.date`、`data.dayOfWeek`、`data.formatted` |
| 2 | `campaign-calendar` | 当月活动日历 → 今日宜 / 近期机缘 | 活动标题、活动内容、活动起止日期、状态（往期/进行中/未来） |
| 3 | `available-coupons` | 可领未领的麦麦省券 → 财气 | 券标题、领取状态 |
| 4 | `query-my-coupons` | 卡包中的券 → 券到期提醒 | `enable`、`datetimeText`、`discountInfo.discountDesc` |
| 5 | `query-my-account` | 积分资产 → "该花的积分" | `availablePoint`、`currentMouthExpirePoint`、`expiredPoint` |
| 6 | `query-lottery-info` | 抽奖状态 → 手气指数 | 活动状态、剩余次数、单次消耗 |
| 7 | `order-list` | 近期订单 → 重复/连吃检测 | 下单时间、餐品名称 |
| 8 | `list-nutrition-foods` | 营养数据 → 幸运单品热量标注 | 能量、脂肪、钠 |

可选增强 Tool（仅在用户要下单/给地址时调用）：

| Tool | 触发条件 | 业务价值 |
| --- | --- | --- |
| `delivery-query-addresses` / `delivery-query-stores` | 用户要外送 | 让幸运单品从真实在售门店菜单中选 |
| `query-nearby-stores` | 用户给地址问附近 | 到店取餐场景 |
| `query-meals` / `query-meal-detail` | 已选门店 | 校验推荐餐品是否在售、套餐构成 |
| `query-store-coupons` | 准备点单 | 确认"今日宜用的券"在该门店真实可用 |
| `auto-bind-coupons` | 用户明确说"帮我领" | 执行"今日宜：领券" |
| `draw-lottery` | 用户明确说"帮我抽" | 执行"今日宜：一发入魂" |
| `mall-points-products` / `mall-create-order` | 用户要花积分 | 把快过期积分变成实物/券 |

## 3. 调用流程

```
用户："今天麦当劳吃什么？"
   │
   ├─ ① now-time-info ──────────── 定日期/星期/时辰
   ├─ ② campaign-calendar ──────── 今日进行中活动 → 宜、未来3天 → 机缘
   ├─ ③ available-coupons ──────── 可领券 → 财气（可与后续并行）
   ├─ ④ query-my-coupons ───────── 今日到期券 → 忌"让券过期"
   ├─ ⑤ query-my-account ───────── 快过期积分 → 宜"先花积分"
   ├─ ⑥ query-lottery-info ─────── 剩余次数 → 宜"抽奖"、手气
   ├─ ⑦ order-list ─────────────── 昨天/连吃 → 忌重复、忌上头
   └─ ⑧ list-nutrition-foods ───── 幸运单品热量标注
   │
   ▼
references/almanac-rules.md 判定（确定性规则 + 幸运指数算法 + 签文三档）
   │
   ▼
references/card-template.md 渲染老黄历卡片 + 懒人结论
```

并行策略：③④⑤⑥⑦⑧ 相互独立，Agent 可在同一批次并发调用，单次生成总耗时通常 ≤ 30 秒。

**降级策略**：任一 Tool 失败（网络/401/429），对应板块显示"天机暂未泄露"，其余板块照常生成；全部失败时才整体报错并提示检查 Token 配置。

## 4. 业务价值

1. **把"查询接口"变成"每日内容"**：官方 MCP 提供的是点餐原子能力，本 Skill 不新增任何后端，只靠编排 8 个查询 Tool 就产出有传播力的每日内容，示范了 MCP 能力的再创造用法。
2. **真实数据杜绝胡编**：老黄历类内容最大的风险是瞎编。本 Skill 的每条宜忌都绑定具体字段（活动 ID、券到期日、过期积分、订单日期），可核验、可复现。
3. **提醒即价值**：券到期提醒、积分过期提醒、活动开始提醒，直接帮用户省钱，避免"券在包里过期"的常见损失。
4. **克制的写操作**：领券、抽奖、兑换、下单全部需要用户逐条确认，避免"AI 替你乱花钱"。
5. **可分享的输出形态**：Markdown 卡片天然适合截图传播，日更属性带来持续使用与口碑扩散。

## 5. 实测记录

以下调用已在真实环境中验证通过（2026-10-09，`https://mcp.mcd.cn`）：

- `initialize` 握手 → 返回 `serverInfo: mcd-mcp/1.0.0`
- `tools/list` → 返回全部官方 Tool 及其 inputSchema
- `now-time-info` → 返回服务器时间与星期
- `campaign-calendar` → 返回当月活动（含往期/进行中活动标题与文案）
- `query-my-account` → 返回 `availablePoint`、`currentMouthExpirePoint` 等积分字段
- `available-coupons` → 返回麦麦省券列表及领取状态
