---
name: data-platform-search
description: >
  数据中台查询技能。当用户意图为数据库查询（订单、客户、指标等业务数据）时，
  Hermes 加载此技能获取表结构、MCP 工具映射和 SQL 生成规范，
  自行生成 SQL 后通过 MCP Server 执行查询，解析结果并输出。
version: 1.0.0
metadata:
  hermes:
    tags: [Data, Business, Query, SQL]
    related_skills: [data-visualization, business-report]
    trigger_intents: [data_query, order_lookup, customer_lookup, metrics_query, sales_trend]
---

# Data Platform Search

## 角色定位

本 Skill 是一个「知识注入包」。它不执行任何逻辑，而是为 Hermes 提供以下知识以完成数据查询闭环：

| 知识模块 | 用途 |
|---------|------|
| MCP 工具 | 告知 Hermes 唯一的数据查询入口及其入参/出参 |
| 表结构 Schema | Hermes 据此生成正确的 SQL 语句 |
| SQL 生成规范 | 约束 SQL 方言、分页、安全策略 |
| 结果解读规则 | Hermes 解析 MCP 返回的原始数据，翻译为业务语言 |

## MCP 工具

MCP Server（`data-platform`）只暴露一个 SQL 执行入口。所有查询——单表、联表、聚合、指标——全部通过此工具完成。Hermes 负责生成 SQL，MCP Server 只负责执行。

```
工具名: mcp_data-platform_query
入参:
  - sql: string (必填) — 完整 SQL 语句（SELECT only）
  - params: object (可选) — SQL 占位符参数绑定，键值对
返回:
  - columns: [{name: string, type: string}] — 列定义
  - rows: [object] — 查询结果行数组
  - total: int — 总行数（分页场景为总计数）
  - elapsed_ms: int — 查询耗时
```

## 表结构 Schema

Hermes 生成 SQL 时，务必以以下 schema 为准。

### orders（订单表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | varchar(32) | PK | 订单ID |
| customer_id | varchar(32) | FK → customers.id | 客户ID |
| amount | decimal(12,2) | NOT NULL | 订单金额 |
| status | varchar(20) | NOT NULL | pending / shipped / cancelled / refunded |
| source | varchar(32) | - | 渠道：online / offline / app |
| created_at | datetime | NOT NULL, INDEXED | 创建时间 |
| updated_at | datetime | - | 最后更新时间 |
| remark | text | - | 备注 |

### customers（客户表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | varchar(32) | PK | 客户ID |
| name | varchar(128) | NOT NULL | 客户名称 |
| tier | varchar(10) | NOT NULL | 等级：vip / gold / silver / normal |
| region | varchar(64) | - | 所属区域 |
| contact_phone | varchar(20) | - | 联系电话（敏感字段） |
| created_at | datetime | NOT NULL | 注册时间 |

### metrics（指标表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | bigint | PK AUTO_INCREMENT | 自增ID |
| name | varchar(64) | NOT NULL, INDEXED | 指标名，如 sales_trend、order_volume |
| value | decimal(14,4) | NOT NULL | 指标值 |
| dimensions | json | - | 维度 KV，如 {"region":"华东","channel":"app"} |
| timestamp | datetime | NOT NULL, INDEXED | 数据时间点 |

### 表关联关系

```
customers.id ──(1:N)──> orders.customer_id
```

## SQL 生成规范

Hermes 生成 SQL 时必须遵守以下规则：

### 安全规则（强制）
1. **禁止全表扫描**：WHERE 子句必须包含索引字段（id、created_at、customer_id、name）
2. **必须分页**：预期行数 > 200 时，必须加 `LIMIT {page_size} OFFSET {(page-1)*page_size}`
3. **参数绑定**：用户输入值必须通过 `params` 传参，禁止直接拼入 SQL 字符串
4. **禁止 DDL/DML**：仅允许 SELECT，禁止 INSERT/UPDATE/DELETE/DROP/ALTER/TRUNCATE

### 方言限制
- 数据库：MySQL 8.0
- 字符串用单引号，双引号视为标识符
- DATE 函数：`DATE(created_at)`、`DATE_SUB(NOW(), INTERVAL 30 DAY)`
- 聚合函数：COUNT、SUM、AVG、MAX、MIN，禁止窗口函数嵌套子查询超过 2 层
- JSON 字段访问：`dimensions->>'$.region'`

### 命名约定
- 别名使用 snake_case
- JOIN 时用表别名：`o` for orders, `c` for customers, `m` for metrics

### SQL 生成示例

```sql
-- ✅ 正确：查询某客户近30天订单
SELECT o.id, o.amount, o.status, o.created_at
FROM orders o
WHERE o.customer_id = :customer_id
  AND o.created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
ORDER BY o.created_at DESC
LIMIT 100 OFFSET 0

-- ✅ 正确：各区域销售额趋势
SELECT
  m.dimensions->>'$.region' AS region,
  DATE(m.timestamp) AS report_date,
  SUM(m.value) AS total_sales
FROM metrics m
WHERE m.name = 'sales_trend'
  AND m.timestamp >= :start_time
GROUP BY region, report_date
ORDER BY report_date DESC

-- ❌ 错误：全表扫描，无 WHERE 条件
SELECT * FROM orders
```

## 结果解读规则

Hermes 获得查询结果后，按以下规则翻译为业务语言呈现给用户：

### 订单状态
| status 值 | 含义 | 提示 |
|-----------|------|------|
| pending | 待处理 | 超过 3 天的 pending 订单建议提醒 |
| shipped | 已发货 | - |
| cancelled | 已取消 | - |
| refunded | 已退款 | - |

### 金额预警
- amount > 100,000 → 🔴 特大额订单，重点关注
- amount 10,000 ~ 100,000 → 🟡 大额订单，标记关注
- amount < 100 → 小额订单

### 指标趋势
- 连续 3 个数据点 value 下降 → ⚠️ 下降趋势告警
- 单日 value 较前日下降 > 50% → 🔴 异常下降，需排查
- 单日 value 较前日上升 > 100% → 需确认是否为业务活动导致

### 敏感数据脱敏
返回结果前必须处理：
- `contact_phone`：中间 4 位替换为 `****`，如 `138****5678`
- 其他字段若用户未明确要求详情，默认只展示摘要统计

## 典型工作流

```
用户: "查一下张三最近的订单情况"

Hermes 规划:
  1. 识别意图 → data_query（客户订单查询）
  2. 加载 data_platform_search Skill → 获取表结构和工具信息
  3. 生成 SQL:
     SELECT c.id, c.name, o.id, o.amount, o.status, o.created_at
     FROM customers c JOIN orders o ON c.id = o.customer_id
     WHERE c.name LIKE CONCAT('%', :name, '%')
       AND o.created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
     ORDER BY o.created_at DESC
     LIMIT 50
  4. 调用 mcp_data-platform_query({sql: "...", params: {name: "张三"}})
  5. 解析结果 → 脱敏 → 格式化输出给用户
```

## 错误处理

| 错误类型 | 处理方式 |
|---------|---------|
| SQL 语法错误 | 检查 SQL 方言，修正后重试一次 |
| 查询超时（> 10s） | 缩小时间范围或增加索引条件后重试 |
| QPS 限流 | 等待 1s 后重试，最多 3 次 |
| 返回空结果 | 提示用户「未查到符合条件的数据」，建议放宽条件 |
| 敏感字段未脱敏 | 输出前执行脱敏规则 |
