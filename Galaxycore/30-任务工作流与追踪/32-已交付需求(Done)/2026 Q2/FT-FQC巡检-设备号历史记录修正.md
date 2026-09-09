---
title: FT FQC巡检 - 设备号历史记录修正
date: 2026-05-20
author: MES Team
tags: [变更日志, FT, FQC, 设备号]
---

# FT FQC巡检 - 设备号历史记录修正

## 问题现象

| 场景 | 结果 |
|------|------|
| 页面勾选了新设备号 | 页面显示正常 |
| 保存后查看批次历史 | 仍显示旧设备号 |
| `SAVEBINANDDOWNGRADE` 历史 | 未体现本次设备 |

## 根因分析

**历史记录页面查询的字段来源：**

`lottranshistoryGrid.js` 显示的 `equipment_id` 取自 `LOT_STEP_HISTORY.EQPT_RRN`，不是 `LOT_TRANS_HISTORY.EQPT_RRN`。

SQL 查询（`LotDAOImpl.java:6579`）：
```sql
SELECT ... ls.eqpt_rrn ...
FROM lot_trans_history lt, lot_step_history ls, transaction_log tr
WHERE lt.lot_rrn = ls.lot_rrn AND lt.step_sequence = ls.step_sequence
```

**两处旧逻辑都有"空值才赋值"限制：**

1. **Action 层**（`FtFqcInspectionAction.java`）旧逻辑：
```java
if (lot.getEqptRrn() == null || lot.getEqptRrn() <= 0) {
    // 仅空值才赋设备号 → 已有旧设备时不会覆盖
    lot.setEqptRrn(eqpRrn);
}
```

2. **GcServiceImpl 层**（`GcServiceImpl.java:2369`）旧逻辑：
```java
if (lotStepHistory.getEqptRrn() == null || lotStepHistory.getEqptRrn() == 0) {
    // 仅空值才赋设备号 → 步骤历史已有设备时不会覆盖
    lotStepHistory.setEqptRrn(lot.getEqptRrn());
}
```

## 为什么不能改 GcServiceImpl

`saveLotDieBin` 是公共方法，被 10 个不同场景调用：

| 调用方 | 是否主动设 eqptRrn |
|--------|-------------------|
| FtFqcInspectionAction (FT FQC巡检) | ✅ 有 |
| RcInspectionAction (RC巡检) | ✅ 有 |
| RcQcInspectionAction (RC QC巡检) | ✅ 有 |
| FtServiceImpl.ftDemotionRelease (降级释放) | ❌ 没设 |
| LotDefectUpdateAction (缺陷更新) | ❌ 没设 |
| LotDispatchBinAction (派工分bin) | ❌ 没设 |
| AeAoiAction (AE AOI) | ❌ 没设 |
| FtAoiAction (FT AOI) | ❌ 没设 |
| WltTrackServiceImpl (WLT过站) | ❌ 没设 |
| LotDispatchBinAction 另一处 | ❌ 没设 |

如果去掉 GcServiceImpl 内部的 `if` 判断，会导致没设 `eqptRrn` 的场景用 LOT 表残留旧值覆盖步骤历史，影响报表和设备统计。

## 最终方案

在 Action 层直接更新步骤历史：

```java
// FtFqcInspectionAction.java:324-329
String eqptId = request.getParameter("aoiEquip");
if (StringUtils.isNotEmpty(eqptId)) {
    long eqpRrn = this.baseService.getNamedObjectRrn(
            eqptId,
            this.baseService.getNamedSpace(ThreadLocalContext.getFacilityRrn(), ObjectList.ENTITY_KEY),
            ObjectList.ENTITY_KEY);
    lot.setEqptRrn(eqpRrn);
    // 直接覆盖 LOT_STEP_HISTORY 的设备号，让历史记录页面显示正确
    lotService.updateLotStepEqRrn(lot.getLotRrn(), lot.getStepSequence(), eqpRrn);
}
```

## 数据落点对比

| 表 | 字段 | 改前 | 改后 |
|------|------|------|------|
| `LOT` | `EQPT_RRN` | 旧设备（不更新） | 旧设备（不更新） |
| `LOT_TRANS_HISTORY` | `EQPT_RRN` | 旧设备（被旧if挡住） | ✅ 新设备 |
| `LOT_STEP_HISTORY` | `EQPT_RRN` | 旧设备（被旧if挡住） | ✅ 新设备（直接UPDATE） |

## 测试清单

| 测试项 | 操作 | 预期 |
|------|------|------|
| 旧设备覆盖 | 批次已有设备，页面换新设备保存 | 历史记录页面显示新设备 |
| 无设备批次 | 首次选设备保存 | 历史正常记录设备 |
| 两次不同设备 | 同批次分别选A/B保存 | 每条历史显示对应设备 |
| 备注与步骤 | 同时填写备注与步骤 | `LOT_REMARKS`、`TRANS_COMMENTS` 正常 |
| 其他场景不受影响 | 降级释放、AOI过站等操作 | 步骤历史设备号不被意外覆盖 |

## 关联文档

- [FT_FQC巡检 功能说明](FT_FQC巡检.md)
