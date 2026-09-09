---
title: FT FQC巡检
date: 2026-05-20
author: MES Team
status: completed
tags: [包装, FT, FQC, 巡检]
---

# FT FQC巡检

## 1. 功能速览

| 项目 | 说明 |
|------|------|
| 功能名称 | FT FQC巡检 |
| 页面目录 | `ftFqcInspection` |
| 主要用途 | 抽检、缺陷录入、等级调整、降级保存、AQL HOLD |
| 关键输入 | 巡检步骤、设备号、扫描批次、批次备注 |
| 关键输出 | bin信息、defect信息、批次历史、步骤历史 |

## 2. 页面流程

```text
选择巡检步骤
  -> 加载设备号
  -> 扫描批次
  -> 查询批次信息
  -> 加载 bin / defect 数据
  -> 执行降级
  -> 点击检验/保存
  -> 写入 SAVEBINANDDOWNGRADE 历史
  -> 命中 AQL 时执行 HOLD
```

## 3. 主要文件

| 层级 | 文件 |
|------|------|
| 前端入口 | `Code-Projects/gc/core/web/wip/ftFqcInspection/ftFqcInspectionViewer.js` |
| 前端表单 | `Code-Projects/gc/core/web/wip/ftFqcInspection/ftFqcInspectionForm.js` |
| 前端主表 | `Code-Projects/gc/core/web/wip/ftFqcInspection/ftFqcInspectionGrid.js` |
| 后端 Action | `Code-Projects/gc/core/src/model/src/com/mycim/webapp/actions/wip/FtFqcInspectionAction.java` |
| 业务实现 | `Code-Projects/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/GcServiceImpl.java` |
| bin保存 | `Code-Projects/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java` |

## 4. 前端提交参数

| 参数 | 说明 |
|------|------|
| `step` | 巡检步骤 |
| `aoiEquip` | 设备号 |
| `scanningLot` | 扫描批次 |
| `lotRemarks` | 批次备注 |
| `lotId` | 批次主键 |
| `binDefinitionList` | 等级信息 |
| `defectBinList` | 缺陷信息 |

## 5. 数据落点

| 表/对象 | 关键字段 | 说明 |
|------|------|------|
| `TRANSACTION_LOG` | `TRANS_START_TIMESTAMP` | 检验时间 |
| `TRANSACTION_LOG` | `TRANS_PERFORMED_BY` | 检验工 |
| `LOT_TRANS_HISTORY` | `EQPT_RRN` | 设备号 |
| `LOT_TRANS_HISTORY` | `LOT_REMARKS` | 批次备注 |
| `LOT_TRANS_HISTORY` | `TRANS_COMMENTS` | 巡检步骤 |
| `LOT` | `LOT_ID` | 批次号 |
| `LOT_EXT` | `LEVEL_TWO_CODE` | 二级代码 |
| `WIP_LOT_DIE_BIN_HIS` | `BIN_ID` / `QTY` | 等级与数量 |
| `DEFECT_BIN_RELATION_H` | `DEFECT_CODE` / `QTY` | 缺陷代码与数量 |
| `LOT_STEP_HISTORY` | `SAMPLE_QTY` | 抽检数量 |

## 6. 关键业务

### 6.1 业务总览

| 业务点 | 说明 |
|------|------|
| 查询 | 按扫描批次查询批次信息 |
| 降级 | 保存 defect 临时记录，调整 bin |
| 检验 | 保存 bin、defect、备注、步骤 |
| AQL HOLD | 命中抽检标准时对批次执行 HOLD |
| 历史生成 | 写入 `SAVEBINANDDOWNGRADE` |

### 6.2 AQL HOLD 是什么

系统先根据批次数量、产品、工步、等级，算出本次应该抽多少颗，同时算出当前抽检规则下"最多允许多少颗不良"。操作员录入缺陷并保存后，系统汇总本次缺陷总数，如果缺陷总数大于等于拒收数，则当前批次触发 `AQL_HOLD`。

核心判断：

```text
本次缺陷总数 >= 当前AQL规则下的拒收数
```

### 6.3 AQL HOLD 数据从哪里来

**1. 页面展示的抽检数 / 拒收数**

页面查询批次 bin 信息时，会查 AQL 配置：

- 根据 `binId + productId + operationId` 取 `AQLProductRelation`
- 再调用 `getLotDieAqlInfo(...)`
- 计算并回填：`sampleQty`、`rejectQty`、`aqlValue`

**2. 缺陷总数**

操作员录入 defect 后，保存时先落到 `DEFECT_BIN_RELATION_H`。后续执行 `handleFtAqlHold(...)` 时，按 `lotRrn + operationRrn` 取本次工步的 defect 历史，把每条 defect 的 `qty` 累加得到 `totalDefectQty`。

**3. AQL 是否启用**

只有当当前工步 `operation` 配置了 `AQL_MOVEOUT_VIEW_FLAG = true`，系统才会做 AQL HOLD 判断。

### 6.4 AQL HOLD 判定流程

```text
查询批次
  -> 读取 AQL 配置
  -> 计算 sampleQty / rejectQty / aqlValue
  -> 页面展示抽检信息

录入 defect 并保存
  -> 保存 defect 历史
  -> 汇总 totalDefectQty
  -> 再次读取当前工步 AQL 信息
  -> 取 rejectQty
  -> 比较 totalDefectQty >= rejectQty
  -> 满足则执行 AQL_HOLD
```

### 6.5 命中 AQL HOLD 后系统做什么

| 动作 | 说明 |
|------|------|
| 生成 `TransReason` | `reasonCode = AQL_HOLD` |
| 组织 hold 参数 | 带上 `lotId`、`lotRrn`、责任人等 |
| 部分工步先移动工步 | 非 `IQC_MERGE` 时先 `moveToOperationId(nextOperation)` |
| 执行 hold | 调用 `holdLotAction(...)` |
| 更新批次状态 | 批次进入 HOLD |

### 6.6 不执行 AQL 卡控的例外场景

| 场景 | 说明 |
|------|------|
| `operationId = IQC_WG_TEST` | 直接跳过 AQL HOLD |
| `operationId = IQC_FUNCTION_TEST` 且 `processId = ENG_ATE_E` | 明确跳过 AQL 卡控 |

---

## 7. 变动日志

| 日期 | 变更摘要 |
|------|----------|
| 2026-05-20 | 修正设备号历史记录显示问题 |
| 2024-10-21 | 初版 FT FQC巡检功能上线 |

> 详细变更记录请查看 [工作流/2026 Q2/FT-FQC巡检-设备号历史记录修正.md](../../../../../../2.工作流/2026%20Q2/FT-FQC巡检-设备号历史记录修正.md)
