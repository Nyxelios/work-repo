---
title: 工单流程卡打印-一体机tray和出货tray
date: 2026-05-03
tags: [MES, 工作流]
---
这两个功能属于 **PRP (Production Route Planning)** 模块，采用经典的 **Struts + ExtJS** 前后端架构：

**code**

```
前端 (ExtJS 4)          →  Struts Action         →  Service 层                →  数据库
JSP → app.js → Viewer   →  XXXAction.java        →  PrpSetupServiceImpl.java →  TRAY_CONFIG 表
     → Form.js → Grid   →  (路由action参数)       →  (CRUD + 历史记录)
```

### 功能一：COM 流程卡打印 (comProcessCardPrint)

**用途**：查询工单/Lot 信息，选择后调用 BarTender 打印 COM 流程卡标签。

**流程**：

1. 用户输入**工单号** (`outerOrderNo`) 或 **Lot ID**，点击"查询"
2. 后端 `ComProcessCardPrintAction.queryInfoList` → 调用 `lotService.getLotPlanList` 查询工单计划
3. 后端在 `dataListToJsonArray` 中为每条记录：
    - 根据**工艺路径 RRN** 获取工艺流程名，按名称含 `CAOI`/`INLINE`/其他，分别选择不同**打印模板** (.btw 文件)
    - 根据产品型号 (`productId`) 查 `TRAY_CONFIG` 表，获取 `productSpec`(出货托盘+颜色) 和 `sealFlag`(一体托盘)
4. 前端展示结果列表，用户勾选后点击"打印"
5. `Ext.doPrintLabel` 拼接打印参数，调用 `bartenderPrint()` 调起 BarTender 打印

**"补打查询"** 功能：调用 `lotService.getComProcessCardLotList`，将结果追加到已有列表中（不清空），用于补打已过站的流程卡。

### 功能二：托盘配置 (trayConfig)

**用途**：维护 `TRAY_CONFIG` 表的 CRUD，为 COM 流程卡打印提供托盘规格数据。

**数据模型**：

|字段|含义|
|---|---|
|BOM_NO|产品型号（唯一键）|
|INTEGRATED_TRAY|一体托盘规格|
|SHIPMENT_TRAY|出货托盘规格|
|TRAY_COLOR|托盘颜色（可选）|

**CRUD 流程**：

1. **查询**：支持按 BOM_NO / 一体托盘 / 出货托盘 / 托盘颜色模糊查询
2. **新增**：弹窗填写，校验必填字段 + BOM_NO 唯一性，插入后记录历史到 `TRAY_CONFIG_HIS`
3. **编辑**：双击行或选中后点 Edit，校验 BOM_NO 不与其他记录冲突，更新后记录历史
4. **删除**：选中后确认删除，先记录历史再物理删除

**历史表 `TRAY_CONFIG_HIS`**：记录每次 INSERT/UPDATE/DELETE 操作，含操作类型、操作人、操作时间。

### 两者关系

**code**

```
trayConfig (维护配置)  ──提供数据──→  comProcessCardPrint (打印时通过 productId/BOM_NO 查询)
```

COM 流程卡打印时，后端通过 `prpSetupService.getTrayConfigByBomNo(productId)` 查 `TRAY_CONFIG` 表获取托盘信息，拼入打印数据中。

### 数据库变更

- `db.changelog-0.0.1.tray_config.yaml`：建 `TRAY_CONFIG` 表 + `TRAY_CONFIG_HIS` 历史表 + 唯一约束 + 序列
- `db.changelog-master.yaml`：已 include 该 changelog

### Struts 路由

- `/comProcessCardPrint.do` → `ComProcessCardPrintAction` → `comProcessCardPrint.jsp`
- `/trayConfig.do` → `TrayConfigAction` → `trayConfig.jsp`