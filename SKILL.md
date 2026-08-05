---
name: data-platform-search
description: >
  数据中台查询技能。当用户意图为数据库查询（订单、客户、指标等业务数据）时，Hermes 加载此技能获取表结构、MCP 工具映射和 SQL 生成规范，
  自行生成 SQL 后通过 MCP Server 执行查询，解析结果并输出。
version: 1.0.0
metadata:
  hermes:
    tags: [Data, Business, Query, SQL, Inventory]
    related_skills: [data-visualization, business-report]
    trigger_intents: [data_query, inventory_query, stock_check, stock_analysis]
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
工具名: mcp_data-platform
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

Hermes 生成 SQL 时，务必以以下 schema 为准。数据库为 SQL Server，schema 为 `dbo`。

### DWD_InventSum_DH（库存汇总表 — 核心表）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| DW_CreatedTime | datetime | NOT NULL | 数仓同步时间 |
| DW_CreateDate | date | NOT NULL | 数仓同步日期 |
| DW_IsDeleted | int | NOT NULL | 逻辑删除标记（0=有效） |
| TransDate | date | NOT NULL | 业务交易日期 |
| Year | int | - | 年 |
| Month | int | - | 月 |
| Day | int | - | 日 |
| Week | int | - | 周次 |
| ItemId | varchar | INDEXED | 物料编码 |
| Trx_ItemClassId | varchar | - | 物料分类编码 |
| VehicleComponentType | varchar | - | 整车零部件类型 |
| Trx_ItemClassName | varchar | - | 物料分类名称 |
| ProductName | varchar | - | 产品名称 |
| InventDimId | varchar | - | 库存维度ID |
| ConfigId | varchar | - | 配置ID |
| InventColorId | varchar | - | 颜色ID |
| InventLocationId | varchar | - | 仓库ID |
| WMSLocationId | varchar | - | 库位ID |
| InventBatchId | varchar | - | 批次ID |
| DataAreaId | varchar | NOT NULL | 数据区域（公司代码），如 "nx" |
| PhysicalInvent | decimal(18,6) | - | 实际库存数量 |
| OnOrder | decimal(18,6) | - | 在途数量 |
| Ordered | decimal(18,6) | - | 已订购数量 |
| AvailOrdered | decimal(18,6) | - | 可用订购数量 |
| AvailPhysical | decimal(18,6) | - | 可用实际数量 |
| RecId | bigint | PK | 记录ID |
| RecVersion | int | - | 记录版本号 |
| Trx_Dept | varchar | - | 部门 |
| Trx_Belong | varchar | - | 归属组织 |
| PhysicalValue | decimal(18,6) | - | 实际库存金额 |
| PostedValue | decimal(18,6) | - | 已过账金额 |
| SumValue | decimal(18,6) | - | 汇总金额 |

### 已知表清单

以下为 DWD 数据库下已确认存在的表，Hermes 可根据查询意图选择：

| 表名 | 说明 |
|------|------|
| dbo.DWD_InventSum_DH | 库存汇总表 — 库存数量、金额、可用量 |

## SQL 生成规范

Hermes 生成 SQL 时必须遵守以下规则：

### 安全规则（强制）
1. **禁止全表扫描**：WHERE 子句必须包含索引字段（TransDate、ItemId、InventLocationId、DataAreaId）
2. **必须分页**：预期行数 > 200 时，必须分页。TOP 和 OFFSET 只能二选一，禁止同时使用：
   - 无需翻页 → `SELECT TOP N ...`
   - 需要翻页 → `OFFSET ... ROWS FETCH NEXT ... ROWS ONLY`（不带 TOP）
3. **参数绑定**：占位符必须是 pymssql 的 `%(name)s` 格式，**绝对禁止** T-SQL 的 `@variable` 语法。传参方式：
   - SQL 中用 `%(name)s` 占位 → `WHERE ItemId = %(item_id)s`
   - 调用时 `params={"item_id": "0022AS000650"}`
4. **禁止 DDL/DML**：仅允许 SELECT，禁止 INSERT/UPDATE/DELETE/DROP/ALTER/TRUNCATE

### 方言限制
- 数据库：SQL Server（T-SQL）
- 字符串用单引号，方括号包裹标识符（如 `[dbo].[DWD_InventSum_DH]`）
- 限制行数用 `SELECT TOP N` 或 `OFFSET ... FETCH NEXT`
- 日期函数：`CONVERT(DATE, GETDATE())`、`DATEADD(DAY, -30, GETDATE())`
- 聚合函数：COUNT、SUM、AVG、MAX、MIN，禁止子查询嵌套超过 2 层
- 字符串拼接用 `CONCAT()` 或 `+`，模糊匹配用 `LIKE '%xxx%'`

### 命名约定
- 别名使用 PascalCase（与表字段风格一致）
- 别名取表名首字母缩写，如 `isd` for `DWD_InventSum_DH`

### SQL 生成示例

```sql
-- ✅ 正确：查询某物料在某仓库的库存（近30天）
SELECT TOP 100
  isd.TransDate, isd.ItemId, isd.ProductName,
  isd.InventLocationId, isd.PhysicalInvent, isd.AvailPhysical,
  isd.SumValue
FROM dbo.DWD_InventSum_DH isd
WHERE isd.ItemId = %(item_id)s
  AND isd.TransDate >= DATEADD(DAY, -30, GETDATE())
  AND isd.DW_IsDeleted = 0
ORDER BY isd.TransDate DESC

-- ✅ 正确：各仓库库存汇总
SELECT
  isd.InventLocationId AS LocationId,
  SUM(isd.PhysicalInvent) AS TotalQty,
  SUM(isd.SumValue) AS TotalValue
FROM dbo.DWD_InventSum_DH isd
WHERE isd.TransDate >= %(start_date)s
  AND isd.DW_IsDeleted = 0
GROUP BY isd.InventLocationId
ORDER BY TotalValue DESC

-- ❌ 错误：全表扫描，无 WHERE 条件
SELECT * FROM dbo.DWD_InventSum_DH
```

## 结果解读规则

Hermes 获得查询结果后，按以下规则翻译为业务语言呈现给用户：

### 库存状态
| 字段 | 解读规则 |
|------|---------|
| DW_IsDeleted = 1 | 已逻辑删除的数据，必须在 WHERE 中过滤 `DW_IsDeleted = 0` |
| AvailPhysical <= 0 | ⚠️ 可用库存为零或负，需关注补货 |
| PhysicalInvent vs AvailPhysical | PhysicalInvent - AvailPhysical = 已预留量 |
| OnOrder > 0 | 有在途采购/调拨，提示预计到货 |

### 金额预警
- SumValue > 1,000,000 → 🔴 高额库存，关注周转
- SumValue < 0 → 异常负数，需排查数据
- PhysicalValue 与 PostedValue 差异过大 → 可能存在未过账交易

### 部门归属
- Trx_Belong、Trx_Dept 为空字符串为正常（未启用维度），不视为异常
- DataAreaId 为必填字段，用于区分不同公司账套

### 敏感数据脱敏
- 目前表中无个人敏感字段（无手机号、身份证等）
- 若涉及具体客户/供应商名称的扩展表，需评估脱敏策略

## 典型工作流

```
用户: "查一下 nx 公司最近30天物料 0022AS000650 的库存情况"

Hermes 规划:
  1. 识别意图 → data_query（库存查询）
  2. 加载 data-platform-search Skill → 获取表结构和工具信息
  3. 生成 SQL:
     SELECT TOP 50
       isd.TransDate, isd.ItemId, isd.ProductName,
       isd.InventLocationId, isd.PhysicalInvent, isd.AvailPhysical,
       isd.SumValue
     FROM dbo.DWD_InventSum_DH isd
     WHERE isd.DataAreaId = %(data_area_id)s
       AND isd.ItemId = %(item_id)s
       AND isd.TransDate >= DATEADD(DAY, -30, GETDATE())
       AND isd.DW_IsDeleted = 0
     ORDER BY isd.TransDate DESC
  4. 调用 mcp_data-platform (query) 执行 SQL
  5. 解析结果 → 解读库存状态 → 格式化输出
```

## 错误处理

| 错误类型 | 处理方式 |
|---------|---------|
| SQL 语法错误 | 检查 SQL 方言，修正后重试一次 |
| 查询超时（> 10s） | 缩小时间范围或增加索引条件后重试 |
| QPS 限流 | 等待 1s 后重试，最多 3 次 |
| 返回空结果 | 提示用户「未查到符合条件的数据」，建议放宽条件 |
| 敏感字段未脱敏 | 输出前执行脱敏规则 |
