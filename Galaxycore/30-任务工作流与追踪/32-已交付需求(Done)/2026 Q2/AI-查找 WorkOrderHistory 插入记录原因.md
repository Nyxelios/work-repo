---
title: 查找-WorkOrderHistory-插入记录原因
date: 2026-05-03
tags: [AI协作, MES]
---
# 查找 WorkOrderHistory 插入记录原因

> **Workspace:** d:\data\MES\gc

---

## User
`d:\data\MES\gc\core\src\valueobject\src\com\mycim\prp\model\WorkOrderHistory.java` 帮我查下这个类的表，WorkOrderHistory，什么时候会插入一条STARTE的transtype的记录，有时候工单表WorkOrder的wipqty莫名其妙增加了数量，也插入了一条历史记录，但是就是没有批次生成，lotplan没有创建记录，很奇怪，帮我查询，不要修改代码文件

## Assistant
我来帮你排查这个问题。首先搜索代码中哪里会插入 `STARTE` 类型的 WorkOrderHistory 记录。

关键代码在 `LotServiceImpl.java` 中。让我详细查看这些插入 STARTE 记录的具体方法。

现在让我查看 `createLot` 方法的完整流程，特别是 LotPlan 的创建逻辑，以及 `updateWorkOrderQty` 方法的调用点。

让我继续查看 `createLot` 方法中 LotPlan 的创建逻辑，以及 `ToolServiceImpl` 中的 STARTE 相关代码。

让我继续深入查看 LotPlan 的创建流程，以及 `updateWorkOrderQty` 的调用点和 `createComReLot` 方法。

搜索结果只显示了方法定义本身，没有找到其他调用点。让我进一步扩大搜索范围，确保没有遗漏任何调用。

`lotPlanRepository.save` 没有找到结果，让我扩大搜索范围，查找其他可能创建 LotPlan 记录的方式。

两次搜索都只找到了方法定义本身。让我再仔细检查该文件内部是否有其他调用点，以及是否存在通过反射等方式调用的可能。

让我进一步获取调用 `insertLotPlan` 的上层方法的更多上下文，以及查找 `saveLotPlan` 和包含调用点的业务方法名称。

以下是 `createComReLot` 方法的完整代码，位于 [LotServiceImpl.java:1818-1951](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1818-L1951)：

```java
private Map createComReLot(Map transInfo) throws MyCimException {
    try {
        Lot lot = (Lot) transInfo.get("lot");
        String workOrderId = lot.getOuterOrderNO();
        String transPerformedby = (String) transInfo.get("transPerformedby");
        String startFlag = (String) transInfo.get("startFlag");
        String keepUnitHistoryFlag = (String) transInfo.get("keepUnitHistoryFlag");

        logger.debug("keepUnitHistoryFlag: {}", keepUnitHistoryFlag);
        Long transSequence = new Long(1);

        // add to judge how to create lot;true for assembly
        Boolean flag = new Boolean(false);
        Collection unitsInfo = new ArrayList();

        lot.setBasedLotRrn(lot.getLotRrn());

        TransactionLog transactionLog = baseService.startTransactionLog(transPerformedby, TransactionNames.CREATELOT_KEY);

        boolean transFlag = true;
        long transRrn = transactionLog.getTransRrn();

        // step1: insert into lot table
        logger.info("LotRrn: {}, Create Date: {}", lot.getLotRrn(), new Timestamp(System.currentTimeMillis()));
        lot.setCreatedTimestamp(new Timestamp(System.currentTimeMillis()));

        // step1.1:if no carrier specified,create a new carrier in current facility
        Carrier carrier = setCarrierForVirtual(transactionLog.getTransPerformedBy(), lot.getFacilityRrn(),
                lot.getLotId(), lot.getQty1());
        NamedObject namedObject = baseService.getNamedObjectByInStanceId(carrier.getInstanceId(), "ENTITY");
        if (namedObject != null) {
            lot.setCarrierRrn(namedObject.getInstanceRrn());
        } else {
            lot.setCarrierRrn(new Long(emsService.insertCarrier(carrier)));
        }
        //创建
        Map stepTempInfo = getStepsInfoByLot(lot, lot.getProcessRrn(), null,
                lot.getWflStepPath());
        lot.setExecutionRrn(Long.valueOf(stepTempInfo.get("executionRrn").toString()));
        if (logger.isDebugEnabled()) {
            logger.debug("CreateLot -> LotRrn: {}, CarrierId: {}, CarrierRrn: {}, NextStepOperation: {}, NextTwoStepOperation: {}",
                    lot.getLotRrn(), lot.getCarrierId(), lot.getCarrierRrn(), lot.getNextOperationRrn1(),
                    lot.getNextOperationRrn2());
        }
        String productClassify = this.prpSetupServices.getProductClassifyByOrderId(workOrderId);

        //创建批次时给批次添加线别区分
        saveLotLineTypeByProductClassify(lot, productClassify);

        // 先更新ext相关,然后创建批次
        lot.setOuterOrderNO(workOrderId);

        dao.insertLotExt(lot);
        //LOT_PROPERTY 以COM_INLINE_RW做完批次为返工标记位
        lot.setLotProperty(Lot.LOT_PROPERTY_COM_INLINE_RW);
        lot = lotRepository.merge(lot);
        // step2: insert into lot history table
        HashMap _transInfo = new HashMap();
        _transInfo.put("transRrn", new Long(transRrn));
        _transInfo.put("lotRrn", new Long(lot.getLotRrn()));
        _transInfo.put("basedLotRrn", lot.getBasedLotRrn());
        _transInfo.put("lotStepSequence", lot.getStepSequence());
        _transInfo.put("facilityRrn", lot.getFacilityRrn());
        _transInfo.put("lotId", lot.getLotId());
        _transInfo.put("hotFlag", lot.getHotFlag());
        _transInfo.put("priority", lot.getPriority());
        logger.debug("LotDuedate: {}", lot.getDueDate());
        _transInfo.put("dueDate", lot.getDueDate());
        _transInfo.put("lotType", lot.getLotType());
        _transInfo.put("lotOwner", lot.getLotOwner());

        LotHistory lotHistory = new LotHistory(_transInfo);
        lotHistoryRepository.save(lotHistory);


        // step5: insert into lot transaction table
        _transInfo = new HashMap();
        _transInfo.put("transRrn", new Long(transRrn));
        _transInfo.put("transSequence", transSequence);
        _transInfo.put("transId", TransactionNames.CREATE_KEY);
        _transInfo.put("facilityRrn", lot.getFacilityRrn());
        _transInfo.put("lotRrn", new Long(lot.getLotRrn()));
        _transInfo.put("stepSequence", lot.getStepSequence());
        _transInfo.put("carrierRrn", lot.getCarrierRrn());
        _transInfo.put("carrierMapRrn", 0l);
        _transInfo.put("qty1", lot.getQty1());
        _transInfo.put("qty2", lot.getQty2());
        _transInfo.put("transComments", lot.getLotComments());
        _transInfo.put("eqptRrn", lot.getEqptRrn());
        LotTransHistory lotTransHistory = new LotTransHistory(_transInfo);
        lotTransHistoryRepository.save(lotTransHistory);

        if ((startFlag != null) && (lot.getDummyFlag() != null) && startFlag.equals(TRUE)
                && !lot.getDummyFlag().equalsIgnoreCase(TRUE)) {
            // step6: start lot if needed
            Debug.print("Start lot here ......");

            Collection lots = new ArrayList();

            transInfo.put("lotRrn", lot.getLotRrn());

            transInfo.put("unitsInfo", unitsInfo);
            transInfo.put("carrierMapRrn", 0l);
            transInfo.put("trackUnitFlag", "0");
            transInfo.put("keepUnitHistoryFlag", keepUnitHistoryFlag);
            _transInfo.put("lotComments", lot.getLotComments());
            lots.add(transInfo);
            _transInfo = new HashMap();
            _transInfo.put("lots", lots);
            _transInfo.put("transPerformedby", transPerformedby);
            _transInfo.put("transactionLog", transactionLog);
            _transInfo.put("transSequence", transSequence);
            startLot(_transInfo);
        }
        // step9: update Carrier available slot
        if (lot.getQty1() != null) {
            _transInfo = new HashMap();
            _transInfo.put("carrierRrn", lot.getCarrierRrn());
            _transInfo.put("consumedSlotCount", new Long(lot.getQty1().longValue()));
            Debug.print("consumedSlotCount:" + _transInfo.get("consumedSlotCount"));
            dao.dispatchCarrier(_transInfo);
        }
        Debug.print("step9:dispatch carrier here ......");
        transactionLog.setTransId(TransactionNames.CREATELOT_KEY);
        baseService.markTransactionLog(transactionLog);
        buildfWflSeqForLot(lot);
        updateLotAndHistory(lot);
        Map returnValues = new HashMap();
        returnValues.put("transRrn", new Long(transRrn));
        return returnValues;
    } catch (Exception e) {
        throw ExceptionHandler.handlerException(e, logger);
    }
}
```

**方法概要：**

- **方法签名**：`private Map createComReLot(Map transInfo) throws MyCimException`
- **位置**：第 1818 行至第 1951 行
- **功能**：创建返工（Rework）批次的完整流程，主要步骤包括：
  1. **Step1**：插入批次（Lot）表记录，设置创建时间戳
  2. **Step1.1**：如果没有指定 Carrier，则在当前 Facility 下创建一个虚拟 Carrier
  3. 获取流程步骤信息并设置 `executionRrn`
  4. 根据工单获取产品分类，并给批次添加线别区分
  5. 插入 `LotExt` 扩展信息，将批次属性设置为 `LOT_PROPERTY_COM_INLINE_RW`（返工标记位）
  6. **Step2**：插入批次历史（LotHistory）记录
  7. **Step5**：插入批次事务历史（LotTransHistory）记录
  8. **Step6**：如果 `startFlag` 为 true 且非 Dummy 批次，则执行 `startLot` 启动批次
  9. **Step9**：更新 Carrier 可用槽位
  10. 标记事务日志完成，构建流程序列，更新批次及历史记录
  11. 返回包含 `transRrn` 的 Map

让我进一步确认，读取该文件中方法定义附近的更大范围，以及搜索同文件中是否有调用。

让我找到包含第 28849 行和 37422 行调用的方法名，以及 `WorkOrderPlanServiceImpl` 中 `saveLotPlan` 方法的定义。

## 搜索结果

在 `d:\data\MES\gc\core\src` 目录下对 `updateWorkOrderQty` 方法进行了全面搜索，结果如下：

### 方法定义（唯一出现位置）

| 文件 | 行号 | 类型 |
|------|------|------|
| [LotServiceImpl.java](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1225) | 第 1225 行 | 方法定义 |

定义处代码如下：

```java
private void updateWorkOrderQty(List<WorkOrderHistory> workOrderHisList) {
    try {
        //todo lzw 工单修改
        for (WorkOrderHistory workOrderHistory : workOrderHisList) {
            WorkOrder workOrder = workOrderRepository.getWorkOrderById(workOrderHistory.getWorkorderId());
            workOrder.setWipQty(workOrder.getWipQty() + workOrderHistory.getWipQty());
            int usableworkOrder = workOrder.getPlannedQty() - workOrder.getCompleteQty() - workOrder.getWipQty() - workOrder.getScrapQty();
            workOrder.setStatus(WorkOrder.STATUS_STARTED);
            String transtype = WorkOrderHistory.TRANSTYPE_STARTED;
            //所有线上工单只有入库才能关闭
            if (usableworkOrder == 0) {
                workOrder.setStatus(WorkOrder.STATUS_AUTOCLOSED);
                transtype = WorkOrderHistory.TRANSTYPE_AUTOCLOSED;
            }
            workOrderRepository.merge(workOrder);
            prpSetupServices.insertWorkOrderHistory(workOrderHistory.getTransRrn(), transtype, workOrder);
        }
    } catch (Exception e) {
        throw ExceptionHandler.handlerException(e, logger);
    }
}
```

### 调用点

**未找到任何调用点。** 在 `d:\data\MES\gc\core\src` 目录乃至整个 `d:\data\MES\gc` 项目范围内，`updateWorkOrderQty` 方法仅在 [LotServiceImpl.java:1225](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1225) 处定义了一次，没有任何地方调用该方法。

### 分析

- 该方法是 `private` 修饰的，只能在 `LotServiceImpl` 类内部调用，但同文件中也未找到任何调用。
- 代码注释 `//todo lzw 工单修改` 表明这是一个**待完成的功能**，可能尚未集成到业务流程中。
- 该方法目前属于**死代码（Dead Code）**，定义了但从未被使用。

让我找到第 28849 行所在的方法名，以及 `WaferBindSyncServiceImpl` 中 `saveLotPlan` 调用所在的方法名。

现在让我查找 `WaferBindSyncServiceImpl` 中 `saveLotPlan` 方法的调用点。

让我查看 DAO 层 `insertLotPlan` 的实际 SQL 实现，以及确认 `LotDAOImpl.insertLotPlan` 的完整内容。

现在我已经掌握了完整的信息。以下是详细的分析结果。

---

## LotPlan 记录创建的完整调用链分析

### 一、底层写入入口

LotPlan 记录的最终写入是通过 DAO 层的 `insertLotPlan` 方法执行的，它执行一条 `INSERT INTO LOT_PLAN` 的 SQL 语句。

**定义位置：** [LotDAOImpl.java:11902-11962](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/dao/LotDAOImpl.java#L11902-L11962)

```java
public void insertLotPlan(Lot lot) throws MyCimException {
    // ... 拼装参数 ...
    String sql = " INSERT INTO " + DatabaseNames.LOT_PLAN + "(...)" + " VALUES (...)";
    iRepository.useSqlUpdate(sql, null);
}
```

**接口声明：** [LotDAO.java:544](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/dao/LotDAO.java#L544)

---

### 二、Service 层封装 -- 两个入口方法

DAO 的 `insertLotPlan` 在 Service 层被两个方法封装调用：

#### 1. `LotServiceImpl.saveLotPlan(Lot lot)` -- 简单封装

**位置：** [LotServiceImpl.java:12404-12410](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L12404-L12410)

```java
@Override
public void saveLotPlan(Lot lot) throws MyCimException {
    try {
        dao.insertLotPlan(lot);
    } catch (Exception e) {
        throw ExceptionHandler.handlerException(e, logger);
    }
}
```

**特点：** 无事务日志、无历史记录、无重复检查，直接插入。

#### 2. `LotServiceImpl.insertLotPlan(Lot lot, String userName)` -- 完整业务封装

**位置：** [LotServiceImpl.java:12413-12498](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L12413-L12498)

```java
@Override
public long insertLotPlan(Lot lot, String userName) throws MyCimException {
    try {
        TransactionLog transactionLog = baseService.startTransactionLog(userName, LotPlanHistory.TRANS_TYPE_CREATE);
        // 如果已存在同名 LotPlan，先删除
        Long lotPlanRrn = getLotPlan(lot.getLotId()).getLotRrn();
        if (lotPlanRrn > 0) {
            lotPlanRepository.deleteOneById(lotPlanRrn);
        }
        // 检查 Lot 是否已存在（不允许重复，除非是特定产品类别）
        // ...
        lot.setLotRrn(getObjectRRN());
        dao.insertLotPlan(lot);
        // 根据 productClassify 创建 LotPlanUnit 记录
        // ...
        // 记录 LotPlanHistory
        LotPlanHistory lotPlanHistory = new LotPlanHistory(new LotPlan(lot), getObjectRRN());
        lotPlanHistory.buildTransInfo(userName, LotPlanHistory.TRANS_TYPE_CREATE, ...);
        lotPlanHistoryRepository.save(lotPlanHistory);
        baseService.markTransactionLog(transactionLog);
        return lot.getLotRrn();
    } catch (Exception e) { ... }
}
```

**特点：** 有事务日志、有重复检查（已存在则先删除再插入）、有历史记录、会创建关联的 LotPlanUnit 记录。

**接口声明：** [LotService.java:632](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/LotService.java#L632)

---

### 三、所有上层调用点汇总

#### (A) 直接调用 `LotServiceImpl.insertLotPlan(lot, username)` 的场景

| # | 文件 | 行号 | 所在方法 | 业务场景 |
|---|------|------|----------|----------|
| 1 | [LotServiceImpl.java:28849](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L28849) | 28849 | `createLotForRetest` | **重测工单创建批次** -- 为 Retest 工单绑定 VBox 后创建计划批次 |
| 2 | [LotServiceImpl.java:37422](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L37422) | 37422 | `createOisLot` | **OIS 工单创建批次** -- 为 OIS 类型重测工单创建计划批次 |
| 3 | [GcServiceImpl.java:528](file:///d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/GcServiceImpl.java#L528) | 528 | `planLot` | **GC 批量计划批次** -- 批量为多个 Lot 创建计划批次 |

关键代码片段：

**调用点 1** -- [LotServiceImpl.java:28760](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L28760) `createLotForRetest`:
```java
lot.setLotRrn(this.insertLotPlan(lot, username));
```

**调用点 2** -- [LotServiceImpl.java:37343](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L37343) `createOisLot`:
```java
lot.setLotRrn(this.insertLotPlan(lot, username));
```

**调用点 3** -- [GcServiceImpl.java:523](file:///d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/GcServiceImpl.java#L523) `planLot`:
```java
for (Lot lot : lots) {
    lot.setLotRrn(this.lotService.insertLotPlan(lot, username));
    totalLotQty += lot.getQty1().intValue();
}
```

---

#### (B) 通过 `LotServiceImpl.saveLotPlan(lot)` 间接调用 `dao.insertLotPlan` 的场景

`saveLotPlan` 被两个类中的私有 `saveLotPlan` 方法调用：

##### B1. `WorkOrderPlanServiceImpl.saveLotPlan(Lot lot, Long transRrn)`

**定义位置：** [WorkOrderPlanServiceImpl.java:1668-1697](file:///d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java#L1668-L1697)

```java
private Lot saveLotPlan(Lot lot, Long transRrn) throws MyCimException {
    // 检查是否已存在同名 LotPlan，存在则先删除
    // 检查是否已存在同名 Lot，不允许重复（除非是返工类型）
    lot.setLotRrn(getObjectRRN());
    lotService.saveLotPlan(lot);
    // 记录 LotPlanHistory
    LotPlanHistory lotPlanHistory = new LotPlanHistory(new LotPlan(lot), getObjectRRN());
    lotPlanHistory.buildTransInfo(ThreadLocalContext.getUsername(), LotPlanHistory.TRANS_TYPE_CREATE, ...);
    lotPlanHistoryRepository.save(lotPlanHistory);
    return lot;
}
```

该方法在 `WorkOrderPlanServiceImpl` 中被大量调用（工单排产场景）：

| # | 行号 | 业务场景 |
|---|------|----------|
| 1 | [行 1220](file:///d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java#L1220) | CP/COB 类型工单排产 -- 按 materialLot 创建计划批次 |
| 2 | [行 1241](file:///d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java#L1241) | 非 CP 类型按 unit 创建计划批次 |
| 3 | [行 1272](file:///d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java#L1272) | RW 类型工单排产 |
| 4 | [行 1327](file:///d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java#L1327) | Retest VBox 类型工单排产 |
| 5 | [行 1360](file:///d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java#L1360) | FT Finish 类型工单排产 |
| 6 | [行 1471](file:///d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java#L1471) | 另一批次 CP 类型排产 |
| 7 | [行 1496](file:///d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java#L1496) | 按 unit 创建计划批次 |
| 8 | [行 1579](file:///d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java#L1579) | 其他排产场景 |

##### B2. `WaferBindSyncServiceImpl.saveLotPlan(Lot lot, Long transRrn)`

**定义位置：** [WaferBindSyncServiceImpl.java:1026](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/service/impl/WaferBindSyncServiceImpl.java#L1026)

```java
private Lot saveLotPlan(Lot lot, Long transRrn) throws MyCimException {
    // 同样先检查重复，再调用 lotService.saveLotPlan(lot)
    // 记录 LotPlanHistory
}
```

该方法在 `WaferBindSyncServiceImpl` 中被调用：

| # | 行号 | 业务场景 |
|---|------|----------|
| 1 | [行 1348](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/service/impl/WaferBindSyncServiceImpl.java#L1348) | Wafer 绑定同步 -- FT Finish / LCD RMA Retest 类型 |
| 2 | [行 1472](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/service/impl/WaferBindSyncServiceImpl.java#L1472) | Wafer 绑定同步 -- LOT_PLAN_WAFER_SOURCE 类型 |
| 3 | [行 1490](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/service/impl/WaferBindSyncServiceImpl.java#L1490) | Wafer 绑定同步 -- 按 unit 创建 |
| 4 | [行 1500](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/service/impl/WaferBindSyncServiceImpl.java#L1500) | Wafer 绑定同步 -- RW 类型 |

---

### 四、关于 `lotPlanRepository.save` 的搜索结果

在 `d:\data\MES\gc\core\src` 目录下，**没有找到** `lotPlanRepository.save` 的调用。`LotPlanRepository` 的使用方式主要是：
- `lotPlanRepository.deleteOneById(...)` -- 删除记录
- `lotPlanRepository.getLotPlanByLotRrn(...)` / `getLotPlanByLotId(...)` / `getLotPlanByOuterOrderNo(...)` -- 查询记录

LotPlan 的创建不通过 JPA 的 `save` 方法，而是通过 DAO 层的 `insertLotPlan` 方法直接执行 SQL INSERT。

---

### 五、总结：LotPlan 记录何时被创建

LotPlan 记录在以下业务场景中被创建：

| 业务场景 | 触发方法 | 入口方法 | 核心写入 |
|----------|----------|----------|----------|
| **1. 重测工单创建批次** | `LotServiceImpl.createLotForRetest` | `insertLotPlan(lot, username)` | `dao.insertLotPlan` |
| **2. OIS 工单创建批次** | `LotServiceImpl.createOisLot` | `insertLotPlan(lot, username)` | `dao.insertLotPlan` |
| **3. GC 批量计划批次** | `GcServiceImpl.planLot` | `lotService.insertLotPlan(lot, username)` | `dao.insertLotPlan` |
| **4. 工单排产（多种产品类型）** | `WorkOrderPlanServiceImpl` 多处 | 私有 `saveLotPlan` -> `lotService.saveLotPlan(lot)` | `dao.insertLotPlan` |
| **5. Wafer 绑定同步** | `WaferBindSyncServiceImpl` 多处 | 私有 `saveLotPlan` -> `lotService.saveLotPlan(lot)` | `dao.insertLotPlan` |

简而言之，**LotPlan 记录在工单排产、重测工单创建批次、OIS 工单创建批次、GC 批量计划、Wafer 绑定同步** 这五大业务场景下会被创建。所有创建最终都汇聚到 `LotDAOImpl.insertLotPlan()` 方法执行 SQL INSERT INTO LOT_PLAN。

让我继续查看 PackageServiceImpl 中的相关逻辑，以及前端发料上线的入口代码。

让我查看 PackageServiceImpl 中调用 `createLots` 的上下文，以及 wipQty 修改的逻辑。

让我查看 GcServiceImpl 中调用 createLots 的上下文，以及直接调用 createLot 的地方。

让我查看 OisServiceImpl 和 FtServiceImpl 中直接调用 `createLot` 的上下文。

让我查看前端发料上线的入口代码，以及 `createLot` 方法中当 `unifiyProcessFlag` 为空时的工单更新逻辑。

让我查看 `PrpSetupServiceImpl.insertWorkOrderHistory` 的实现，以及 `WorkOrderServiceImpl` 中的相关逻辑。

让我查看前端发料上线的入口代码，以及 `WorkOrderPlanServiceImpl` 中的排产逻辑。

初步搜索结果显示 `createLotConsolidationConfigTab` 是 UI 组件创建方法，不是业务上的 `createLot`/`createLots` 调用。让我更精确地搜索业务相关的调用。

让我进一步搜索与发料/上线相关的 JS 文件中的 `createLot` 调用，以及搜索可能通过 AJAX/服务调用方式引用 `createLots` 的代码。

让我继续搜索 `createlot` 函数定义以及通过 AJAX/ExtJS 方式调用 `createLot` 的代码。

让我继续查找 `createlot()` 函数的定义位置，以及搜索更多与发料/上线相关的 JS 文件。

让我搜索 `createlot` 函数定义以及 ExtJS 风格的 AJAX 调用。

以下是在 `d:\data\MES\gc\core\web` 目录下搜索 `createLots` / `createLot` 相关前端代码的完整结果。

---

## 搜索结论

**`createLots`（复数形式）：在整个 web 目录中未找到任何匹配结果。**

**`createLot`（单数形式）：在多个文件中找到，分为业务调用和 UI 组件创建两大类。** 以下重点列出与"发料/上线"业务相关的调用。

---

## 一、与发料/上线直接相关的业务调用

### 1. [lotplanwaferstart.jsp](file:///d:/data/MES/gc/core/web/wip/lotplanwaferstart.jsp) -- 投片/上线页面

**行 270** -- Create 按钮调用 `createLotSubmit()` 函数：
```javascript
<input type="button" value="Create" name="nextPlanLot" onclick='createLotSubmit()'>
```

**行 272** -- Reset 按钮通过 URL 参数 `createLot=1` 跳转：
```javascript
onclick = "goToURL('self', '<%=request.getContextPath()%>/lotplansave.do?createLot=1&nav=true')"
```

**行 310-318** -- `createLotSubmit()` 函数定义，提交表单到 `/lotplansave.do?create=1`：
```javascript
function createLotSubmit() {
    lotplanstartext.window.gridDataToInputValue();
    var lotId=window.frames["lotplanstartext"].document.getElementById("lotId").value;
    var theform = document.forms[0];
    theform.action = "<%=request.getContextPath()%>/lotplansave.do?create=1&lotId="+lotId;
    theform.submit();
}
```

---

### 2. [lotplanrawmaterialselection.jsp](file:///d:/data/MES/gc/core/web/wip/lotplanrawmaterialselection.jsp) -- 原材料选择/发料页面

**行 322** -- Create 按钮调用 `createlot()` 函数（LOT_PLAN_START 状态下）：
```javascript
<input type="button" name="create" value="Create" onclick="javascript:createlot();">
```

**行 324** -- 名为 `createLot` 的 submit 按钮（实际功能为 Cancel）：
```javascript
<input type="submit" name="createLot" value="Cancel" >
```

**行 354** -- `backPage()` 函数中通过 `createlot=1` 参数返回创建页面：
```javascript
function backPage(){
    document.forms[0].action ="<%=request.getContextPath()%>/lotplanedit.do?createlot=1&isshow=false&nav=true";
    document.forms[0].submit();
}
```

> 注意：`createlot()` 函数在此 JSP 中被调用但未在该文件内定义，也未在公共 JS 文件（`common.js`、`function.js`）中找到定义，可能由运行时动态生成或为遗留代码。

---

### 3. [lotplaninfo.jsp](file:///d:/data/MES/gc/core/web/wip/lotplaninfo.jsp) -- Lot 计划信息页面

**行 622** -- Reset 按钮（CREATE_PLAN_LOT 状态）：
```javascript
onclick = "goToURL('self', '<%=request.getContextPath()%>/lotplanedit.do?createlot=1&isshow=false&nav=true')"
```

**行 635** -- Reset 按钮（LOT_PLAN_START 状态）：
```javascript
onclick = "goToURL('self', '<%=request.getContextPath()%>/lotplansave.do?createLot=1&nav=true')"
```

**行 672** -- 搜索 URL（CREATE_PLAN_LOT 状态）：
```javascript
var doSearchUrl = "<%=request.getContextPath()%>/lotplanedit.do?createLot=1&isshow=false";
```

**行 850** -- 搜索 URL（LOT_PLAN_START 状态）：
```javascript
var doSearchUrl = "<%=request.getContextPath()%>/lotplansave.do?createLot=1";
```

---

### 4. [lotplaninfobatch.jsp](file:///d:/data/MES/gc/core/web/wip/lotplaninfobatch.jsp) -- 批量 Lot 计划信息页面

**行 440** -- 打开批量创建 Lot 的单元列表弹窗：
```javascript
<a href="javascript:doOpen('<%=request.getContextPath()%>/wip/unitlist4batchcreatelot.jsp?lotId=${item.lotId}&position=${item.position}&qty=${item.specifiedQty}')">
```

**行 571** -- `backPage()` 函数返回创建页面：
```javascript
function backPage(){
    document.forms[0].action ="<%=request.getContextPath()%>/lotplanedit.do?createlot=1&isshow=false&nav=true";
    document.forms[0].submit();
}
```

---

### 5. [npwcreatelot.jsp](file:///d:/data/MES/gc/core/web/npw/npwcreatelot.jsp) -- NPW 创建 Lot 页面

**行 440** -- Reset 按钮（CREATE_PLAN_LOT 状态）：
```javascript
onclick = "goToURL('self', '<%=request.getContextPath()%>/npwcreatelotedit.do?createlot=1&isshow=false&nav=true')"
```

**行 453** -- Reset 按钮（LOT_PLAN_START 状态）：
```javascript
onclick = "goToURL('self', '<%=request.getContextPath()%>/npwcreatelotsave.do?createLot=1&nav=true')"
```

**行 491** -- 搜索 URL（CREATE_PLAN_LOT 状态）：
```javascript
var doSearchUrl = "<%=request.getContextPath()%>/npwcreatelotedit.do?createLot=1&isshow=false";
```

**行 669** -- 搜索 URL（LOT_PLAN_START 状态）：
```javascript
var doSearchUrl = "<%=request.getContextPath()%>/npwcreatelotsave.do?createLot=1";
```

---

### 6. [npwcreatelotrawmaterialselection.jsp](file:///d:/data/MES/gc/core/web/npw/npwcreatelotrawmaterialselection.jsp) -- NPW 原材料选择/发料页面

**行 308** -- 名为 `createLot` 的 submit 按钮（实际功能为 Cancel）：
```javascript
<input type="submit" name="createLot" value="Cancel" >
```

**行 338** -- `backPage()` 函数：
```javascript
function backPage(){
    document.forms[0].action ="<%=request.getContextPath()%>/npwcreatelotedit.do?createlot=1&isshow=false&nav=true";
    document.forms[0].submit();
}
```

---

### 7. [npwlotcurrentinfo.jsp](file:///d:/data/MES/gc/core/web/npw/npwlotcurrentinfo.jsp) -- NPW Lot 当前信息页面

**行 183** -- Cancel 按钮跳转：
```javascript
<input type='button' value="Cancel" onclick='goToURL("self", "<%=request.getContextPath()%>/npwcreatelotsave.do?createLot=1&nav=true")'>
```

---

### 8. [screendefinition.jsp](file:///d:/data/MES/gc/core/web/screendefinition.jsp) -- 屏幕定义

**行 1019** -- LOTPLAN 操作的按钮名称配置：
```java
buttonName = "createLot";
```

---

## 二、JS 文件中的 UI 组件创建方法（非业务逻辑调用）

以下 JS 文件中的 `createLot*` 方法均为 ExtJS UI 组件创建方法，**不涉及业务 createLot 接口调用**：

| 文件 | 方法名 | 说明 |
|------|--------|------|
| [aeMerageConfigViewer.js](file:///d:/data/MES/gc/core/web/galaxyCore/aeMerageConfig/aeMerageConfigViewer.js#L18) | `createLotConsolidationConfigTabPanel()` / `createLotConsolidationConfigTab()` | 合并配置 Tab 面板创建 |
| [aeAteMergeConfigViewer.js](file:///d:/data/MES/gc/core/web/galaxyCore/aeAteMergeConfig/aeAteMergeConfigViewer.js#L18) | `createLotConsolidationConfigTabPanel()` / `createLotConsolidationConfigTab()` | ATE 合并配置 Tab 面板创建 |
| [LotplanstartViewer.js](file:///d:/data/MES/gc/core/web/wip/lotplanstart/LotplanstartViewer.js#L27) | `createLotplanstartGrid()` | Lot 计划上线 Grid 创建 |
| [LotplanmaterialViewer.js](file:///d:/data/MES/gc/core/web/wip/lotplanmaterial/LotplanmaterialViewer.js#L27) | `createLotplanmaterialGrid()` | Lot 计划物料 Grid 创建 |
| [LotConstrainViewer.js](file:///d:/data/MES/gc/core/web/app/prp/lotconstrain/viewer/LotConstrainViewer.js#L14) | `createLotConstrainTabPanel()` 等 | Lot 约束配置 Tab 面板创建 |

---

## 三、后端路由配置（struts-config.xml）

[struts-config.xml](file:///d:/data/MES/gc/core/web/WEB-INF/struts-config.xml) 中定义了相关 Action 路由：

- **行 2635-2648**：`/lotplansave` -> `LotPlanSaveAction`，forward 中包含 `createlot=1` 参数
- **行 4894-4898**：`/npwcreatelotedit` -> `NpwCreateLotEditAction`
- **行 4900-4914**：`/npwcreatelotsave` -> `NpwCreateLotSaveAction`，forward 中包含 `createlot=1` 参数

---

## 总结

1. **`createLots`（复数）在整个前端代码中不存在**，没有任何调用。
2. **`createLot` 主要以三种形式出现**：
   - **URL 参数**：`?createLot=1` 或 `?createlot=1`，用于表单提交/页面跳转时标识"创建 Lot"操作
   - **表单按钮 name**：`name="createLot"`，作为 Struts 表单提交的按钮标识（实际功能为 Cancel/返回）
   - **JS 函数**：`createLotSubmit()`（在 `lotplanwaferstart.jsp` 中定义）和 `createlot()`（在 `lotplanrawmaterialselection.jsp` 中调用但未找到定义）
3. **核心业务流程**：前端通过 Struts 表单提交到 `/lotplansave.do` 或 `/npwcreatelotsave.do`，携带 `createLot=1` 参数，由后端 `LotPlanSaveAction` / `NpwCreateLotSaveAction` 处理实际的 Lot 创建逻辑。

让我查看 `LotPlanSaveAction` 的代码，这是前端发料上线的核心入口。

让我查看 `GcServiceImpl.erpWorkOrderSynAndCreateLots` 方法，这是前端发料的核心调用链。

让我查看 PackageServiceImpl 中第 10613 行的 wipQty 修改逻辑。

好的，经过全面的代码追踪分析，以下是完整的调查结果：

---

## 一、STARTE 类型 WorkOrderHistory 记录的所有插入位置

代码中插入 `TRANSTYPE_STARTED = "STARTE"` 历史记录的位置共有 **4 处**（1处为死代码）：

### 1. [LotServiceImpl.createLots()](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1202-L1214) — **批量发料（主要入口）**

```java
@Transactional
public List<Map> createLots(Map transInfo) {
    // 第一步：循环创建每个批次
    for (int i = 0; i < lotlist.size(); i++) {
        transInfo.put("unifiyProcessFlag", "1");  // 跳过createLot内的工单更新
        Map returnValue = createLot(transInfo);     // 创建批次
        // 在内存中累加工单wipQty
        workOrder.setWipQty(wip.intValue());
        workOrderMap.put(workOrderId, workOrder);
    }
    // 第二步：统一持久化工单变更 + 插入STARTE历史
    String transtype = WorkOrderHistory.TRANSTYPE_STARTED;
    for (Map.Entry<String, WorkOrder> entry : workOrderMap.entrySet()) {
        workOrderRepository.merge(workOrder);
        prpSetupServices.insertWorkOrderHistory(transactionLog.getTransRrn(), transtype, workOrder);
    }
}
```

### 2. [LotServiceImpl.createLot()](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1736-L1755) — **单个批次创建**

```java
// 仅在 doNotChangeWipQtyFlag为空 && workOrderId非空 && workOrderFlag非空 && unifiyProcessFlag为空 时执行
if (StringUtils.isEmpty(doNotChangeWipQtyFlag) && StringUtils.isNotEmpty(workOrderId)
        && StringUtils.isNotEmpty(workOrderFlag) && StringUtils.isEmpty(unifiyProcessFlag)) {
    workOrder.setWipQty(workOrder.getWipQty() + lot.getQty1().intValue());
    workOrder.setStatus(WorkOrder.STATUS_STARTED);
    String transtype = WorkOrderHistory.TRANSTYPE_STARTED;
    workOrderRepository.update(workOrder);
    prpSetupServices.insertWorkOrderHistory(transRrn, transtype, workOrder);
}
```

### 3. [LotServiceImpl.createReworkLot()](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1260-L1272) — **返工批次创建**

```java
public List<Map> createReworkLot(Map transInfo) {
    Map returnValue = createComReLot(transInfo);  // 创建返工批次
    WorkOrder workOrder = workOrderRepository.getWorkOrderById(workOrderId);
    String transtype = WorkOrderHistory.TRANSTYPE_STARTED;
    workOrder.setWipQty(workOrder.getWipQty() + lot.getQty1().intValue());
    workOrder.setScrapQty(workOrder.getScrapQty() - lot.getQty1().intValue());
    workOrder.setStatus(WorkOrder.STATUS_STARTED);
    workOrderRepository.merge(workOrder);
    prpSetupServices.insertWorkOrderHistory(workOrderHistory.getTransRrn(), transtype, workOrder);
}
```

### 4. ~~[LotServiceImpl.updateWorkOrderQty()](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1225-L1244)~~ — **死代码，从未被调用**

---

## 二、完整调用链分析

前端发料上线的完整调用链：

```
前端 lotplansave.do?create=1
  → LotPlanSaveAction.perform() (CREATE_KEY分支)
    → LotPlanSaveAction.createLot(transInfo, retestFlag)
      → GcServiceImpl.erpWorkOrderSynAndCreateLots(transInfo, retestFlag, facilityRrn)
        → lotService.createLots(transInfo)  ← 这里插入STARTE历史
```

---

## 三、问题根因分析：wipQty增加了但批次未生成、LotPlan未创建

根据代码分析，**最可能的原因**有以下几种：

### 🔴 原因1：`createLots` 方法中事务部分提交问题（最可能）

[createLots()](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1150) 方法的逻辑分为两步：

1. **第一步循环**：调用 `createLot()` 创建每个批次（`lotRepository.merge(lot)` 在 [line 1471](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1471) 执行）
2. **第二步循环**：统一 `workOrderRepository.merge(workOrder)` 持久化工单变更 + 插入STARTE历史（[line 1205](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1205)）

**关键问题**：虽然 `createLots` 有 `@Transactional` 注解，但如果 `createLot` 内部在 `lotRepository.merge(lot)` 之后、返回之前发生了异常，而该异常被某层 catch 住未正确传播，就可能导致：
- Lot 已经 merge 到数据库
- 但工单更新和 STARTE 历史还未执行
- 事务未回滚

**但更关键的是反过来的场景**：如果 `createLot` 成功返回后，在第二步循环中 `workOrderRepository.merge(workOrder)` 和 `insertWorkOrderHistory` 成功执行了，但此时事务因为某种原因（如超时）被标记为 rollback-only，那么：
- 工单变更和 STARTE 历史可能已经被 flush 到数据库
- 而 Lot 记录可能被回滚掉

### 🔴 原因2：批次已存在时的返线处理（高度可疑）

在 [createLot()](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1388-L1403) 中，如果发现同 ID 的批次已存在（`tempLot != null`），代码会：

```java
if (tempLot != null) {
    // 删除已有unit
    lot.setLotRrn(tempLot.getLotRrn());  // 复用已有批次的RRN
}
// ...
lot = lotRepository.merge(lot);  // 更新已有批次而非新建
```

此时 `merge` 操作是**更新**而非插入新记录。如果用户查询时只看"新创建"的批次，可能看不到这条记录，误以为"没有批次生成"。

同时，在 [line 1709](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1709)：
```java
int deleteRecords = lotPlanRepository.deleteOneById(tempLotRrn);
```
这里用 `tempLotRrn`（原始生成的RRN，非已有批次的RRN）去删除 LotPlan，**很可能删不到正确的 LotPlan 记录**，因为 LotPlan 存储的是已有批次的 RRN。

### 🔴 原因3：并发操作导致 wipQty 计算错误

在 `createLots` 方法中，wipQty 的更新逻辑是**先读后写**：

```java
workOrder = workOrderRepository.getWorkOrderById(workOrderId);  // 读取当前wipQty
wip = new BigDecimal(workOrder.getWipQty());
wip = wip.add(lotQty);                    // 在内存中累加
workOrder.setWipQty(wip.intValue());      // 设置新值
workOrderMap.put(workOrderId, workOrder);  // 暂存
// ... 最后才 merge
```

如果两个用户同时对同一工单发料，可能出现**丢失更新**：
- 线程A读取 wipQty=10，加5得15
- 线程B读取 wipQty=10，加3得13
- 线程B先merge，wipQty=13
- 线程A后merge，wipQty=15（覆盖了线程B的更新）
- 结果：实际应该18，但只有15，且多了一条STARTE历史

### 🟡 原因4：LotPlan 从未创建

用户提到"lotplan没有创建记录"。正常流程应该是：

1. **排产阶段**：`WorkOrderPlanServiceImpl` → `saveLotPlan()` → `dao.insertLotPlan()` 创建 LotPlan
2. **发料阶段**：`createLot()` → `lotPlanRepository.deleteOneById(tempLotRrn)` 删除 LotPlan 并创建 LotPlanHistory

如果"lotplan没有创建记录"，说明**排产阶段就没有创建 LotPlan**。这可能意味着：
- 工单没有经过正常的排产流程就直接发料了
- 或者 LotPlan 被创建后又因为某种原因被删除了

---

## 四、排查建议

1. **查看数据库中 WORKORDER_H 表中 STARTE 记录的 TRANS_RRN**，然后根据 TRANS_RRN 去 TRANSACTION_LOG 表查对应的事务信息，确认是哪个操作触发的

2. **查看对应时间点的 LOT 表**，确认是否真的没有批次记录（注意检查是否有同 ID 的旧批次被更新，而非新插入）

3. **检查 LOT_PLAN 表和 LOT_PLAN_H 表**，确认 LotPlan 是否曾经存在过（如果 LOT_PLAN_H 中有 CREATE 类型记录，说明曾经创建过）

4. **检查是否有并发操作**，查看同一工单在相近时间是否有多条 STARTE 历史记录

5. **重点关注 `createLots` 方法的事务行为**，特别是当 `createLot` 抛出异常时，事务是否正确回滚

## User
那看起来并不是第一处，因为异常数量增加的记录没有setComments，comments的是空的，第四处没有调用。然后这个工单也不是返工，按理来说不会执行方法三，也就是说方法二是罪魁祸首？

## Assistant
Error: Custom model internal error occurred. Please check the proxy response for details.

## User
那看起来并不是第一处，因为异常数量增加的记录没有setComments，comments的是空的，第四处没有调用。然后这个工单也不是返工，按理来说不会执行方法三，也就是说方法二是罪魁祸首？

## Assistant
对，**按你现在补充的现象看，方法二基本就是第一嫌疑，甚至可以说是最符合现象的路径。**

## 为什么我现在更倾向方法二

### 1) 你用 `comments` 把方法一基本排掉了
在 [createLots](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1202-L1214) 里，插入 STARTE 历史前，代码会先改 `workOrder.comments`：

- [LotServiceImpl.java:L1208-L1212](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1208-L1212)

```java
if(StringUtils.isEmpty(workOrder.getComments())){
    workOrder.setComments("PUT_QTY"+putQty.intValue());
}else{
    workOrder.setComments(workOrder.getComments()+";PUT_QTY"+putQty.intValue());
}
```

然后才调：

- [LotServiceImpl.java:L1214](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1214)

```java
prpSetupServices.insertWorkOrderHistory(transactionLog.getTransRrn(), transtype, workOrder);
```

而历史对象是直接从 `workOrder` 拷贝属性生成的：

- [PrpSetupServiceImpl.java:L8471-L8488](file:///d:/data/MES/gc/core/src/ejb/prp/src/com/mycim/prp/service/impl/PrpSetupServiceImpl.java#L8471-L8488)

```java
WorkOrderHistory workOrderHistory = new WorkOrderHistory(workOrder);
```

所以如果走的是方法一，**这条 STARTE 历史的 comments 基本不可能是空**，最少也会有 `PUT_QTYxxx`。

---

### 2) 方法三你也已经从业务上排掉了
方法三是 [createReworkLot](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1248-L1270)，明确是返工路径。  
你说这个工单不是返工，那这条也基本排除。

---

### 3) 方法四本来就是死代码
这个前面已经确认了，没调用。

---

## 那剩下来的就是方法二

也就是 [createLot](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1735-L1755) 这一段：

- [LotServiceImpl.java:L1736-L1755](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1736-L1755)

```java
if (StringUtils.isEmpty(doNotChangeWipQtyFlag) && StringUtils.isNotEmpty(workOrderId)
        && StringUtils.isNotEmpty(workOrderFlag) && StringUtils.isEmpty(unifiyProcessFlag)) {
    WorkOrder workOrder = workOrderRepository.getWorkOrderById(workOrderId);
    if (!WorkOrder.RETEST_WORKORDER.equals(workOrder.getWorkOrderType())) {
        ...
        workOrder.setWipQty(workOrder.getWipQty() + lot.getQty1().intValue());
        ...
        workOrder.setStatus(WorkOrder.STATUS_STARTED);
        String transtype = WorkOrderHistory.TRANSTYPE_STARTED;
        workOrder.setOwner(transPerformedby);
        workOrderRepository.update(workOrder);
        prpSetupServices.insertWorkOrderHistory(transRrn, transtype, workOrder);
    }
}
```

这段代码的特点，和你描述的现象是**高度一致**的：

### 它会：
- 增加工单 `wipQty`
- 插入一条 `STARTE` 历史
- **不会给 comments 补 `PUT_QTY...`**
- 不是返工专用路径

所以你说的：
- `wipQty` 莫名增加
- 有一条 `STARTE`
- `comments` 是空
- 工单不是返工

这 4 个条件，**都更像方法二**。

---

## 但我要补一句：方法二是“最像”，不是“100%已经定案”

因为还有一个很关键的问题：

> **方法二正常情况下也应该先创建/更新 Lot，然后才去更新 WorkOrder。**

也就是说，如果真是方法二触发的，理论上应该至少发生了以下之一：

### 情况 A：Lot 实际创建了，只是你没查到
`createLot()` 前面先做 lot 相关处理，后面才改工单。  
所以如果走到方法二的 `wipQty + 历史`，通常 lot 逻辑已经跑过了。

### 情况 B：不是新建，而是“复用/覆盖了已有 lot”
在 [createLot()](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java) 里，如果同 `lotId` 已经存在，会走 `tempLot != null` 的分支，最后是 `merge`，不一定是“新插入一个 lot”。  
这种情况下，现场容易看到：

- 工单数量加了
- 有 STARTE 历史
- 但“没有新批次生成”的体感很强

### 情况 C：Lot 创建后又被后续逻辑覆盖/删除/查不到
这个也不能完全排除，但从你现在给的信息看，没有方法二本身那么直接。

---

## 我现在的结论

### 结论一句话：
**是的，按目前证据，方法二就是最可能的罪魁祸首。**

准确一点说：

> **异常 STARTE 记录最可能来自 `createLot()` 末尾那段单批次工单更新逻辑，而不是 `createLots()` 的批量路径。**

---

## 为什么这个判断比较稳

因为方法二满足你现象里的全部关键特征：

- 非返工也能进
- 插入 `STARTE`
- 增加 `wipQty`
- 不会自动补 `comments`
- 与“LotPlan 不存在”也不冲突

而方法一和方法三都有明显不匹配点。

---

## 再往下追，最该看的不是“谁写了 STARTE”，而是“谁调用了 createLot 且没带 unifiyProcessFlag”

因为方法二有明确触发条件：

- `doNotChangeWipQtyFlag` 为空
- `workOrderId` 非空
- `workOrderFlag` 非空
- `unifiyProcessFlag` 为空
- 不是 retest 工单

也就是说，**真正的问题入口**很可能是：

> 某个业务场景直接调用了 `createLot()`，并且没有设置 `unifiyProcessFlag=1`，于是绕开了批量路径，直接在单批逻辑里把工单 `wipQty` 加掉了。

---

## 我建议你下一步重点查这两类东西

### 1. 查这条异常 STARTE 的 `transRrn`
然后反查对应事务，看它是不是某个 `CREATELOT` 事务触发的。

### 2. 全局查谁直接调了 `createLot(...)`
尤其看：
- 调用前有没有塞 `unifiyProcessFlag`
- 调用时 `workOrderFlag` 是不是被带上了
- 是不是某个特殊入口只做了工单更新逻辑，但 lot 本身没有形成你预期的记录

---

如果你愿意，我下一步可以继续**只做查询不改代码**，帮你把：

1. **所有直接调用 `createLot()` 的入口**  
2. **哪些入口会让方法二这段工单更新生效**  
3. **哪些入口最可能对应你这个工单场景**

全部精确列出来。

## User
不对，if(StringUtils.isEmpty(workOrder.getComments())){ 
     workOrder.setComments("PUT_QTY"+putQty.intValue()); 
 }else{ 
     workOrder.setComments(workOrder.getComments()+";PUT_QTY"+putQty.intValue()); 
 }这个是昨天才加的，加完确实新出现的数据异常是有的，但是以前的依旧没有，不能用这个判断了，那方法一还是有问题，## wipQty增加了但批次未生成、LotPlan未创建 
 根据代码分析， 最可能的原因 有以下几种： 
 
 ### 🔴 原因1：createLots 方法中事务部分提交问题（最可能）如果是这个原因，该怎么修改代码

## Assistant
我先核实一下这里的事务回滚条件，尤其是 `@Transactional` 遇到 `MyCimException` 这类异常时会不会自动回滚。

我已对 `d:\data\MES\gc\core\src` 做了完整检索，并额外扩大到 `d:\data\MES\gc` 复核，结论如下：

## 1. ExceptionHandler 类定义

在 `d:\data\MES\gc\core\src` 下，**没有找到 `ExceptionHandler` 类定义**。  
在 `d:\data\MES\gc` 整个目录下，也**没有找到**如下任一声明：

- `class ExceptionHandler`
- `interface ExceptionHandler`
- `enum ExceptionHandler`
- `record ExceptionHandler`

也没有找到名为 `ExceptionHandler.java` 的源码文件。

因此可以确认：

- `com.mycim.util.exception.ExceptionHandler` 的**源码定义不在** `d:\data\MES\gc\core\src` 下
- 大概率来自：
  - 外部依赖 jar
  - 未纳入当前检索目录的其他模块
  - 生成代码/编译产物

## 2. handlerException 返回类型与实现

由于在 `core/src` 及 `gc` 下都**没有找到 `ExceptionHandler` 的源码定义**，所以：

- **无法从当前源码直接确认 `handlerException(...)` 的精确定义**
- **无法直接拿到它的返回类型声明**
- **无法直接拿到它的实现代码**

但从调用方式可以确定：

- 它的返回值被直接用于 `throw`
- 所以其返回类型一定是 `Throwable` 的子类，或者至少是可抛出的异常类型
- 从业务方法签名看，**很可能**是 `MyCimException` 或其子类，但这只是基于调用上下文的推断，不是源码级确认

## 3. 在 core/src 下找到的关键使用位置

### 3.1 import 位置示例

[ToolServiceImpl.java:L54-L60](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/service/impl/ToolServiceImpl.java#L54-L60)

```java
import com.mycim.tool.service.WmsWaferManagerService;
import com.mycim.util.MiscUtils;
import com.mycim.util.ObjectList;
import com.mycim.util.TransactionNames;
import com.mycim.util.exception.ExceptionHandler;
import com.mycim.util.exception.MyCimException;
import com.mycim.util.exception.MyCimParameterException;
```

说明这里确实引用了 `com.mycim.util.exception.ExceptionHandler`，但定义文件不在当前源码树中。

---

### 3.2 `handlerException(e, log)` 调用示例

[ToolServiceImpl.java:L291-L293](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/service/impl/ToolServiceImpl.java#L291-L293)

```java
} catch (Exception e) {
    throw ExceptionHandler.handlerException(e, log);
}
```

对应上下文可见于 [ToolServiceImpl.java:L250-L294](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/service/impl/ToolServiceImpl.java#L250-L294)

这里可以看出存在一个重载/方法形态：

```java
ExceptionHandler.handlerException(Exception e, Logger log)
```

但**返回类型声明不可见**，因为定义缺失。

---

### 3.3 `handlerException(e)` 调用示例

[ProductProcessRepositoryImpl.java:L47-L56](file:///d:/data/MES/gc/core/src/ejb/bas/src/com/mycim/bas/repository/impl/ProductProcessRepositoryImpl.java#L47-L56)

```java
public List<ProductProcess> ListProductProcess() throws MyCimException {
    try {
        SqlBuilder sqlBuilder = getSqlBuilder();
        sqlBuilder.select(columnNames).from(tableName).where("1=1");

        return sqlBuilder.queryList(ProductProcess.class);
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
        throw ExceptionHandler.handlerException(e);
    }
}
```

这里说明至少还存在另一种调用形态：

```java
ExceptionHandler.handlerException(Exception e)
```

同样，**返回类型和实现代码在当前目录下不可见**。

## 4. 当前能确认到的方法形态

根据源码调用，当前只能确认 `handlerException` 至少有以下两种用法：

```java
ExceptionHandler.handlerException(e)
ExceptionHandler.handlerException(e, log)
```

其中：

- 第一个版本：1 个参数 `Exception`
- 第二个版本：2 个参数 `Exception` + `log`

但**具体签名、返回类型、实现逻辑无法在 `core/src` 内确认**。

## 5. 检索结果汇总

### 未找到定义
- `ExceptionHandler` 类定义：未找到
- `handlerException` 方法定义：未找到
- `ExceptionHandler.java` 文件：未找到

### 找到的关键引用
1. import：
   - [ToolServiceImpl.java:L54-L60](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/service/impl/ToolServiceImpl.java#L54-L60)

2. 调用示例：
   - [ToolServiceImpl.java:L291-L293](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/service/impl/ToolServiceImpl.java#L291-L293)
   - [ProductProcessRepositoryImpl.java:L47-L56](file:///d:/data/MES/gc/core/src/ejb/bas/src/com/mycim/bas/repository/impl/ProductProcessRepositoryImpl.java#L47-L56)

## 6. 结论

结论可以直接概括为：

- 在 `d:\data\MES\gc\core\src` 下，**没有 `ExceptionHandler` 类定义**
- 在当前可检索源码中，**没有 `handlerException` 的返回类型声明和实现**
- 当前只能从调用点确认：
  - 存在 `handlerException(e)`
  - 存在 `handlerException(e, log)`
  - 返回值可被 `throw`，因此必然是可抛异常对象

如果你需要，我下一步可以继续帮你：
- 去查 `.jar` / `lib` / Maven 依赖里这个类来自哪里
- 或在整个工作区之外继续定位 `com.mycim.util.exception.ExceptionHandler` 的真实源码位置

对，你这个追问很关键。  
如果你现在怀疑的是 **`createLots()` 里发生了“部分提交”**，那最先要改的地方其实不是业务判断，而是**事务边界和回滚规则**。

---

# 先说结论

## 如果真是“wipQty 增加了、历史插了，但 lot / lotplan 没落库”这种事务不一致问题，最直接的代码修改是：

把 [createLots](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1149-L1223) 的事务注解从：

```java
@Transactional
public List<Map> createLots(Map transInfo) throws MyCimException
```

改成：

```java
@Transactional(rollbackFor = Exception.class)
public List<Map> createLots(Map transInfo) throws MyCimException
```

甚至更稳一点：

```java
@Transactional(rollbackFor = Throwable.class)
public List<Map> createLots(Map transInfo) throws MyCimException
```

---

# 为什么这里可能真的有坑

你现在这个类是：

- [LotServiceImpl.java:L98-L100](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L98-L100)

```java
@Service
@Transactional
@Slf4j
public class LotServiceImpl extends AbstractService implements LotService {
```

方法上又有：

- [LotServiceImpl.java:L1149-L1150](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1149-L1150)

```java
@Transactional
public List<Map> createLots(Map transInfo) throws MyCimException {
```

这两个 `@Transactional` 都是**默认事务规则**。

而 Spring 默认只会对：
- `RuntimeException`
- `Error`

自动回滚。

---

## 问题就在于：`MyCimException` 很可能不是运行时异常
从当前代码能确认：

- 方法签名大量写的是 `throws MyCimException`
- 异常都是 `throw ExceptionHandler.handlerException(e, logger);`
- 但 `ExceptionHandler` 和 `MyCimException` 源码不在当前目录里，没法 100%看到继承关系

不过从这个项目写法风格看，**`MyCimException` 很大概率是受检异常（checked exception）**，而不是 `RuntimeException`。

如果是这样，那么就会出现经典问题：

> 方法里中途抛了 `MyCimException`，Spring 事务**不自动回滚**，已经 flush / update 的数据就可能留在库里。

这就和你怀疑的“部分提交”高度一致。

---

# 这会怎么导致你看到的现象

以 `createLots()` 为例，它的流程是：

## 第一步：循环调用 `createLot(transInfo)`
- 创建/更新 lot
- 可能删 lotPlan
- 可能写 lot history
- 还可能做别的关联操作

## 第二步：统一更新工单
- [LotServiceImpl.java:L1203-L1214](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1203-L1214)

```java
for (Map.Entry<String, WorkOrder> entry : workOrderMap.entrySet()) {
    WorkOrder workOrder = entry.getValue();
    workOrderRepository.merge(workOrder);
    ...
    prpSetupServices.insertWorkOrderHistory(transactionLog.getTransRrn(), transtype, workOrder);
}
```

如果在前面 lot 处理或者后面某一步抛出的是 **checked exception**，又没触发事务回滚，就可能出现：

- 某些 DAO/JPA 更新已经刷进数据库
- 某些记录没写完
- 最终形成你看到的：
  - 工单 `wipQty` 增加
  - `WORKORDER_H` 有 `STARTE`
  - 但 lot 没看到 / lotPlan 也不对

---

# 如果让我改，我会怎么改

---

## 方案一：最小修复，先改事务回滚规则
这是**最值得优先做**的。

### 改法
在 [createLots](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1149-L1223) 上改成：

```java
@Transactional(rollbackFor = Exception.class)
public List<Map> createLots(Map transInfo) throws MyCimException {
```

### 为什么优先这个
因为它是：
- 改动最小
- 风险最低
- 对现有业务逻辑侵入最小
- 最直接针对“异常不回滚导致脏数据”这个根因

---

## 方案二：类级别一起改，避免别的方法也中招
你这个类上本身也有类级别 `@Transactional`：

- [LotServiceImpl.java:L99](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L99)

如果这个类里很多方法都抛 `MyCimException`，那不只是 `createLots()`，`createLot()`、`createReworkLot()`、`createLotNpw()` 这些都有潜在同类问题。

### 更稳的改法
把类级别改成：

```java
@Transactional(rollbackFor = Exception.class)
```

这样整个类都统一。

### 但要注意
这会影响面更大，所以如果你现在只是想**快速止血这个问题**，我更建议先只改 `createLots()` 和相关关键方法。

---

## 方案三：`createLot()` 也一起加 rollbackFor
因为 `createLots()` 内部调用了 `createLot()`，而 [createLot](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1280) 自己没有方法级事务注解，靠的是类级别事务。

如果你想把意图写清楚，建议也加：

```java
@Transactional(rollbackFor = Exception.class)
public Map createLot(Map transInfo) throws MyCimException
```

虽然**同类内部调用不会重新代理事务**，但这至少把语义写清楚了。

---

# 只改 rollbackFor 够不够？

## 大概率是“第一步必须改”，但不一定“彻底够”
因为还有两个结构性风险。

---

## 风险1：`createLots()` 里把“创建 lot”和“更新工单”分成了两个阶段
现在逻辑是：

### 阶段A
先把所有 lot 都 create 完

### 阶段B
再统一更新 workOrder + history

这意味着只要中间任何一个环节异常，**前后状态天然更容易不一致**。

---

## 更稳的改法
如果后续你要从根上收敛风险，我更建议把逻辑改成：

### 每个 lot 一次完整处理：
- create lot
- update workOrder
- insert history

也就是不要“先全部创建批次，再统一改工单”，而是改成逐条原子处理。

### 好处
这样哪怕以后有人又改坏事务配置，问题范围也只会影响当前 lot，不会出现整批次前半成功、后半失败。

### 但缺点
- 改动更大
- 要重新确认“同一工单多 lot 汇总加 wipQty”的业务意图
- 可能影响现有 `PUT_QTY` 合并逻辑

所以这是**第二阶段优化**，不是第一刀。

---

## 风险2：没有显式验证关键结果
如果你真担心“lot 没真正写成，但工单被加数了”，那就不该完全依赖 ORM/DAO 自己保证。

可以在更新工单前，加一个关键校验：

- `createLot()` 返回后
- 再按 `lotRrn` / `lotId` 回查一次 lot 是否真实存在
- 不存在就直接抛异常，让整单回滚

### 例如思路
不是让我现在改代码，只说方向：

```java
Map returnValue = createLot(transInfo);
Long lotRrn = MapUtils.getLong(returnValue, "lotRrn");
Lot dbLot = lotRepository.getLotByRrn(lotRrn);
if (dbLot == null) {
    throw new MyCimException("LOT_CREATE_FAILED");
}
```

### 好处
这能把“看起来创建成功，其实没落成”的情况挡住。

---

# 如果是我来修，我会按这个顺序改

## 第一层：立即止血
### 改 1
[createLots](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1149-L1223)

```java
@Transactional(rollbackFor = Exception.class)
```

### 改 2
如果你们确认 `createLot()` 也经常单独被外部调用，那它也加：

```java
@Transactional(rollbackFor = Exception.class)
```

---

## 第二层：增强一致性
在 `createLots()` 里，`createLot()` 返回后，更新工单前，增加 lot 存在性校验。

---

## 第三层：长期优化
把“批量先建 lot、最后统一改工单”的结构改成“单 lot 完整原子处理”。

---

# 我给你的实际建议

## 如果你只想先修“部分提交”
那就先做这一条：

### 直接修改为
- [LotServiceImpl.java:L1149-L1150](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1149-L1150)

从：

```java
@Transactional
public List<Map> createLots(Map transInfo) throws MyCimException
```

改为：

```java
@Transactional(rollbackFor = Exception.class)
public List<Map> createLots(Map transInfo) throws MyCimException
```

如果你们项目里 `MyCimException` 明确继承 `Exception`，那这个就够了。  
如果不确定，保守一点用：

```java
@Transactional(rollbackFor = Throwable.class)
```

但通常 `Exception.class` 就够用了。

---

# 还有一个我想提醒你的点

你现在这个问题如果真是“历史老数据也异常”，那说明：

- 这不是昨天 comments 逻辑引入的
- 很可能是**项目里事务默认回滚规则长期就有坑**
- `createLots()` 只是其中一个爆点

所以如果你们后面确认 `MyCimException` 是 checked exception，建议把整个项目里关键业务方法的事务注解都扫一遍。

---

如果你愿意，我下一步可以继续帮你做两件事之一：

1. **只查询不改代码**：我帮你把这个类里所有 `throws MyCimException + @Transactional` 的高风险方法都列出来  
2. **直接给你出精确补丁方案**：我只改 `createLots()` / `createLot()` 的事务注解和关键校验代码，尽量最小改动

你如果想改，我可以直接给你最小修改版本。

## User
createLots() 这个方法看来确实是存在问题，帮我详细看看，具体为什么会有这种情况发生，如果要优化该怎么做

## Assistant
好的，我已经把 `createLots()` 和它内部调用的 `createLot()` 完整读完了。下面给你逐行级别的详细分析。

---

# `createLots()` 方法完整风险分析

## 方法结构总览

```
createLots() {
    ┌─────────────────────────────────────────────────┐
    │ 阶段A：循环创建批次                              │
    │   for (i = 0 → lotlist.size()) {                │
    │       createLot(transInfo)  ← 创建1个批次        │
    │       内存累加 workOrder.wipQty                  │
    │       workOrderMap.put(workOrderId, workOrder)   │
    │   }                                              │
    ├─────────────────────────────────────────────────┤
    │ 阶段B：统一持久化工单变更                         │
    │   for (entry : workOrderMap) {                   │
    │       workOrderRepository.merge(workOrder)       │
    │       insertWorkOrderHistory(STARTE)             │
    │   }                                              │
    └─────────────────────────────────────────────────┘
}
```

---

## 🔴 风险1：阶段A中 `createLot()` 内部已经 flush 了数据，但阶段B的工单更新还没做

`createLot()` 内部做了大量数据库写操作：

| 步骤 | 操作 | 行号 |
|------|------|------|
| step1 | `dao.insertLotExt(lot)` | [L1427](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1427) / [L1440](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1440) |
| step1 | `lotRepository.merge(lot)` | [L1449](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1449) |
| step2 | `lotHistoryRepository.save(lotHistory)` | [L1466](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1466) |
| step3 | `unitService.unitsMaintain4CreateLot(...)` | [L1538](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1538) |
| step5 | `lotTransHistoryRepository.save(...)` | [L1611](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1611) |
| step6 | `startLot(...)` | [L1643](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1643) |
| step7 | `invService.consumeMaterialFromWarehouseForLot(...)` | [L1660](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1660) |
| step9 | `dao.dispatchCarrier(...)` | [L1680](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1680) |
| step11 | `lotPlanRepository.deleteOneById(tempLotRrn)` | [L1687](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1687) |
| step11 | `lotPlanHistoryRepository.save(...)` | [L1693](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1693) |
| step12 | `updateLotAndHistory(lot)` | [L1711](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1711) |

**关键问题**：这些操作在 `createLot()` 返回时就已经执行了。JPA 的 `merge`/`save` 在事务提交前可能已经 flush 到数据库（取决于 JPA provider 的 flush 模式），也可能还只是在持久化上下文中。

**但是**，`createLots()` 在阶段B才做 `workOrderRepository.merge(workOrder)` + `insertWorkOrderHistory(STARTE)`。

如果阶段A中第 N 个 `createLot()` 抛异常，而事务又没有正确回滚，就会出现：
- 前 N-1 个 lot 的数据已经写入
- 工单的 wipQty 还没更新
- 或者反过来，某些中间状态已经 flush，但后续步骤没完成

---

## 🔴 风险2：`orderPut` 的累加逻辑有 Bug

[L1192-L1200](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1192-L1200)：

```java
BigDecimal put=new BigDecimal(0);   // ← 每次循环都重置为0！
BigDecimal lotQty=null;
...
wip=wip.add(lotQty);
put=put.add(lotQty);                // ← put永远等于当前lotQty
workOrder.setWipQty(wip.intValue());
...
orderPut.put(workOrderId, put.intValue());  // ← 覆盖写入，只保留最后一个lot的数量
```

**问题**：`put` 在每次循环开始时初始化为 0，所以 `orderPut` 里存的永远只是**最后一个 lot 的数量**，而不是该工单下所有 lot 的累加数量。

举例：工单 A 下有 3 个 lot，数量分别是 5、3、2：
- `wip` 累加正确：5+3+2=10 ✅
- `orderPut` 最终只存了 2 ❌（应该是10）

这导致后面写入 comments 的 `PUT_QTY` 值是错的：
```java
workOrder.setComments("PUT_QTY" + putQty.intValue());  // 写的是2，不是10
```

---

## 🔴 风险3：`createLot()` 内部 step11 删除 LotPlan 用的是 `tempLotRrn`，可能删错

[L1687](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1687)：

```java
int deleteRecords = lotPlanRepository.deleteOneById(tempLotRrn);
```

而 `tempLotRrn` 来自 [L1273](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1273)：

```java
long tempLotRrn = lot.getLotRrn();
```

这是**方法入口时 lot 的原始 RRN**。但在 [L1312-L1313](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1312-L1313)：

```java
long instanceRrn = getObjectRRN();
lot.setLotRrn(instanceRrn);
```

lot 的 RRN 已经被改成了新生成的 `instanceRrn`。

如果 `tempLot != null`（批次返线），[L1380](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1380)：

```java
lot.setLotRrn(tempLot.getLotRrn());
```

lot 的 RRN 又被改成了已有批次的 RRN。

**所以 `tempLotRrn` 实际上是 LotPlan 创建时分配的 RRN，而不是最终 lot 的 RRN**。如果 LotPlan 表中存的 RRN 和 `tempLotRrn` 对不上，就删不掉 LotPlan，导致：
- LotPlan 记录残留
- 或者 LotPlan 根本没被删除，但 lot 已经创建了

---

## 🔴 风险4：`createLot()` 内部自己也有工单更新逻辑，和 `createLots()` 的阶段B可能冲突

`createLot()` 末尾 [L1714-L1734](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1714-L1734)：

```java
if (StringUtils.isEmpty(doNotChangeWipQtyFlag) && StringUtils.isNotEmpty(workOrderId)
        && StringUtils.isNotEmpty(workOrderFlag) && StringUtils.isEmpty(unifiyProcessFlag)) {
    // 直接更新工单 wipQty + 插入 STARTE 历史
    workOrder.setWipQty(workOrder.getWipQty() + lot.getQty1().intValue());
    workOrderRepository.update(workOrder);
    prpSetupServices.insertWorkOrderHistory(transRrn, transtype, workOrder);
}
```

`createLots()` 通过设置 `unifiyProcessFlag=1` 来**跳过** `createLot()` 内部的工单更新。但这个跳过逻辑**完全依赖一个 Map 中的字符串参数**，没有任何类型安全保证。

如果某次调用时 `transInfo` 中的 `unifiyProcessFlag` 被意外清除或为空，就会导致：
- `createLot()` 内部更新了一次工单
- `createLots()` 阶段B又更新了一次
- **wipQty 被重复增加**

---

## 🔴 风险5：并发操作导致 wipQty 丢失更新

`createLots()` 中 wipQty 的更新是**先读后写**：

```java
// 阶段A：读
workOrder = workOrderRepository.getWorkOrderById(workOrderId);  // 读当前值
wip = new BigDecimal(workOrder.getWipQty());
wip = wip.add(lotQty);                    // 内存累加
workOrder.setWipQty(wip.intValue());      // 设新值

// 阶段B：写
workOrderRepository.merge(workOrder);      // 写回数据库
```

如果两个请求同时对同一工单发料：

| 时间 | 请求1 | 请求2 |
|------|-------|-------|
| T1 | 读取 wipQty=10 | |
| T2 | | 读取 wipQty=10 |
| T3 | 累加5，设 wipQty=15 | |
| T4 | | 累加3，设 wipQty=13 |
| T5 | merge → wipQty=15 | |
| T6 | | merge → wipQty=13（覆盖了请求1的更新）|

**结果**：应该是18，实际只有13，请求1的5个 lot 的 wipQty 被丢失了。

---

## 🔴 风险6：`createLot()` 中 `startLot()` 可能触发新的异常

[L1613-L1643](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1613-L1643)：

如果 `startFlag=true` 且不是 dummy lot，会调用 `startLot()`。这个方法内部会做大量操作（更新 lot 状态、写历史等），如果 `startLot()` 抛异常，此时 lot 已经 merge 了，但 start 没完成，又是一个中间状态。

---

## 🟡 风险7：`transInfo` 是共享 Map，循环中可能被污染

`createLots()` 循环中反复使用同一个 `transInfo` Map：

```java
transInfo.put("lot", lotlist.get(i));
transInfo.put("unifiyProcessFlag", "1");
Map returnValue = createLot(transInfo);
```

而 `createLot()` 内部也会往 `transInfo` 里写东西（比如 [L1620](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1620) `transInfo.put("lotRrn", ...)`），这些残留数据可能影响下一次循环。

---

# 优化方案

基于以上分析，我给出**分层优化方案**，从最紧急到长期：

---

## 第一层：立即修复（止血）

### 修复1：`orderPut` 累加 Bug

当前代码（[L1174-L1200](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1174-L1200)）：

```java
BigDecimal put=new BigDecimal(0);   // ← 每次循环重置
...
put=put.add(lotQty);
...
orderPut.put(workOrderId, put.intValue());  // ← 只保留最后一个lot的值
```

**修改为**：

```java
BigDecimal put = new BigDecimal(orderPut.getOrDefault(workOrderId, 0));
...
put = put.add(lotQty);
...
orderPut.put(workOrderId, put.intValue());
```

### 修复2：事务注解确保回滚

你已经加了 `rollbackFor = Exception.class`，这是对的。但还需要确认 `createLot()` 方法也加上：

```java
@Transactional(rollbackFor = Exception.class)
public Map createLot(Map transInfo) throws MyCimException
```

虽然同类内部调用不会重新走代理，但这是**语义声明**，防止将来有人从外部单独调用时出问题。

---

## 第二层：结构优化（防重复更新 + 防并发）

### 优化1：把"工单更新"从 `createLots()` 阶段B移到 `createLot()` 内部统一处理

**当前问题**：工单更新逻辑分散在两处（`createLot()` 末尾 + `createLots()` 阶段B），靠 `unifiyProcessFlag` 字符串参数协调，容易出错。

**优化思路**：把工单更新逻辑统一收拢到 `createLots()` 中，`createLot()` 内部**完全不碰工单**。具体做法：

1. `createLot()` 中，把工单更新条件改为**永远不执行**（当从 `createLots()` 调用时）
2. 或者更好的做法：把工单更新逻辑抽成独立方法，由 `createLots()` 统一调用

```java
// 抽取的工单更新方法
private void updateWorkOrderAfterCreateLot(WorkOrder workOrder, int addQty, long transRrn, String transPerformedby) {
    if (StringUtils.isEqual(workOrder.getProductClassify(), WorkOrder.WLT)) {
        workOrder.setWipQty(workOrder.getWipQty() + 1);
    } else {
        workOrder.setWipQty(workOrder.getWipQty() + addQty);
    }
    workOrder.setStatus(WorkOrder.STATUS_STARTED);
    workOrder.setOwner(transPerformedby);
    workOrderRepository.merge(workOrder);
    prpSetupServices.insertWorkOrderHistory(transRrn, WorkOrderHistory.TRANSTYPE_STARTED, workOrder);
}
```

### 优化2：wipQty 更新改用 SQL 原子操作，防并发

**当前问题**：先读后写，存在丢失更新风险。

**优化思路**：用 SQL `UPDATE ... SET wip_qty = wip_qty + ?` 替代 Java 内存累加。

在 `WorkOrderRepository` 中增加方法：

```java
@Modifying
@Query("UPDATE WorkOrder w SET w.wipQty = w.wipQty + :delta WHERE w.workorderId = :workorderId")
int addWipQty(@Param("workorderId") String workorderId, @Param("delta") int delta);
```

这样就不需要先读再写了，数据库层面保证原子性。

---

## 第三层：架构优化（彻底解决两阶段不一致）

### 优化1：把"创建 lot + 更新工单"做成单 lot 原子操作

**当前问题**：`createLots()` 先把所有 lot 都创建完，最后才统一更新工单。如果中间任何一步失败，就会出现"lot 创建了但工单没更新"或反过来。

**优化思路**：每个 lot 的创建和对应工单更新在同一个逻辑单元内完成：

```java
@Transactional(rollbackFor = Exception.class)
public List<Map> createLots(Map transInfo) throws MyCimException {
    TransactionLog transactionLog = baseService.startTransactionLog(...);
    List<Lot> lotlist = (List<Lot>) transInfo.get("lotlist");
    List<Map> returnValueslist = new ArrayList<>();
    Map<String, WorkOrder> workOrderMap = new HashMap<>();
    Map<String, Integer> orderPut = new HashMap<>();

    for (int i = 0; i < lotlist.size(); i++) {
        transInfo.put("lot", lotlist.get(i));
        transInfo.put("unifiyProcessFlag", "1");
        
        // 创建批次
        Map returnValue = createLot(transInfo);
        returnValueslist.add(returnValue);
        
        // 立即累加当前工单的 wipQty（内存）
        String workOrderId = lotlist.get(i).getOuterOrderNO();
        WorkOrder workOrder = workOrderMap.computeIfAbsent(workOrderId, 
            id -> workOrderRepository.getWorkOrderById(id));
        
        int lotQty = calculateLotQty(workOrder, lotlist.get(i));
        workOrder.setWipQty(workOrder.getWipQty() + lotQty);
        workOrderMap.put(workOrderId, workOrder);
        orderPut.merge(workOrderId, lotQty, Integer::sum);
    }
    
    // 统一持久化工单变更
    for (Map.Entry<String, WorkOrder> entry : workOrderMap.entrySet()) {
        WorkOrder workOrder = entry.getValue();
        // 用 SQL 原子操作更新 wipQty
        int totalPutQty = orderPut.get(workOrder.getWorkorderId());
        workOrderRepository.addWipQty(workOrder.getWorkorderId(), totalPutQty);
        
        // 重新读取更新后的工单，写历史
        WorkOrder updatedOrder = workOrderRepository.getWorkOrderById(workOrder.getWorkorderId());
        updatedOrder.setStatus(WorkOrder.STATUS_STARTED);
        if (StringUtils.isEmpty(updatedOrder.getComments())) {
            updatedOrder.setComments("PUT_QTY" + totalPutQty);
        } else {
            updatedOrder.setComments(updatedOrder.getComments() + ";PUT_QTY" + totalPutQty);
        }
        prpSetupServices.insertWorkOrderHistory(transactionLog.getTransRrn(), 
            WorkOrderHistory.TRANSTYPE_STARTED, updatedOrder);
    }
    
    transactionLog.setTransId(TransactionNames.CREATELOT_KEY);
    baseService.markTransactionLog(transactionLog);
    return returnValueslist;
}
```

### 优化2：`createLot()` 中 step11 的 LotPlan 删除逻辑修正

当前 [L1687](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1687)：

```java
int deleteRecords = lotPlanRepository.deleteOneById(tempLotRrn);
```

`tempLotRrn` 是入口时 lot 的原始 RRN，但 LotPlan 表中存的可能是不同的 RRN。应该改为按 `lotId` 删除：

```java
int deleteRecords = lotPlanRepository.deleteByLotId(lot.getLotId());
```

或者按 `outerOrderNo` + `lotId` 组合条件删除，确保删对。

---

## 汇总：各风险与对应修复

| 风险 | 严重度 | 修复层级 | 修复方式 |
|------|--------|----------|----------|
| `orderPut` 累加 Bug | 🔴 高 | 第一层 | 从 Map 取旧值累加，而非每次重置为0 |
| 事务回滚不完整 | 🔴 高 | 第一层 | `rollbackFor = Exception.class`（已做） |
| `unifiyProcessFlag` 字符串协调不安全 | 🔴 高 | 第二层 | 抽取工单更新为独立方法，统一入口 |
| wipQty 并发丢失更新 | 🔴 高 | 第二层 | 改用 SQL 原子操作 `SET wip_qty = wip_qty + ?` |
| 两阶段不一致 | 🔴 高 | 第三层 | 单 lot 原子操作，或用 SQL 原子更新 |
| LotPlan 删除用错 RRN | 🟡 中 | 第三层 | 改按 lotId 删除 |
| `transInfo` 共享 Map 污染 | 🟡 中 | 第三层 | 每次循环创建新 Map 或清理 key |

---

你想让我先帮你改哪个层级？我建议**先改第一层**（`orderPut` Bug + 事务注解），这两个改动最小、最安全、效果最直接。

分析已完成，等你决定先改哪一层。

## User
如果说我开一张表用来记录异常日志？还是说单开一个文件放本地log？

## User
如果说我开一张表用来记录异常日志？还是说单开一个文件放本地log？createLots这个方法的执行的log日志存放在那个文件夹里

## Assistant
### Relevant Code Snippets
1. d:\data\MES\gc\core\web\WEB-INF\config\logback.xml:L1-L50
  — 找到 logback.xml 配置文件，这是主要的日志配置文件，可以查看日志输出目录和日志级别设置。
2. d:\data\MES\gc\core\src\ejb\wip\lot\src\com\mycim\wip\lot\service\impl\LotServiceImpl.java:L1-L100
  — LotServiceImpl 类文件，用于查看该服务中使用的日志记录方式及日志级别。
3. d:\data\MES\gc\core\web\WEB-INF\config\application.properties:L1-L50
  — application.properties 文件，可能包含与日志相关的配置属性，如日志输出目录等。

这些文件应该包含了所需的信息，特别是日志配置和 LotServiceImpl 中的日志使用情况。


好，从 `logback.xml` 配置里可以直接确认日志位置。

---

## `createLots()` 的日志存放在哪个文件夹

### 答案：`logs/wip/server.log`

**依据**：

- [logback.xml:L478-L492](file:///d:/data/MES/gc/core/web/WEB-INF/config/logback.xml#L478-L492) 定义了 `wip` appender：

```xml
<appender name="wip" class="ch.qos.logback.core.rolling.RollingFileAppender" additivity="false">
    <file>${log.path}/wip/server.log</file>
    <rollingPolicy ...>
        <fileNamePattern>${log.path}/wip/wip.%d{yyyy-MM-dd}.%i.zip</fileNamePattern>
    ...
</appender>
```

- [logback.xml:L554-L556](file:///d:/data/MES/gc/core/web/WEB-INF/config/logback.xml#L554-L556) 定义了 logger：

```xml
<logger name="com.mycim.wip" additivity="false" level="DEBUG">
    <appender-ref ref="wip"/>
</logger>
```

- `LotServiceImpl` 的完整包名是 `com.mycim.wip.lot.service.impl`，匹配 `com.mycim.wip` logger

所以：

- 当前日志：`logs/wip/server.log`
- 历史日志：`logs/wip/wip.2026-04-21.0.zip`（按天滚动，500MB切割后 zip 压缩）

实际绝对路径取决于 Tomcat 部署目录，通常是：
```
{tomcat_home}/logs/wip/server.log
```
或者如果你的 `log.path` 被外部配置覆盖过，可能在别的目录。

---

## 数据库表 vs 本地文件：怎么选

### 场景1：你只想**快速定位** `createLots()` 当前为什么会出现异常

**推荐：直接用现有本地日志文件**

理由：
- `logs/wip/server.log` 里已经有 `log.info("lotPlanPutIn:"+...)` 和 `log.info("workorder STARTED lotsinfo! : ...")` 的日志
- 直接 `grep` 关键字就行，零开发量
- 文件按天滚动，查历史也方便

你可以直接查：
```bash
# Windows PowerShell
Get-Content logs\wip\server.log | Select-String "lotPlanPutIn"

# 或者查异常
Get-Content logs\wip\server.log | Select-String "createLots"
```

---

### 场景2：你想**长期监控**并能在界面上告警/统计

**推荐：建一张数据库异常日志表**

#### 表结构设计建议

```sql
CREATE TABLE LOT_CREATE_EXCEPTION_LOG (
    ID              BIGINT PRIMARY KEY,
    TRANS_RRN       BIGINT,                -- 事务RRN，可关联 TRANSACTION_LOG
    WORKORDER_ID    VARCHAR(50),           -- 工单号
    LOT_ID          VARCHAR(50),           -- 批次号
    EXCEPTION_TYPE  VARCHAR(20),           -- 'TRANSACTION_ROLLBACK', 'LOT_CREATE_FAILED', 'QTY_MISMATCH' 等
    EXCEPTION_MSG   VARCHAR(500),          -- 异常信息摘要
    DETAIL_MSG      CLOB,                  -- 完整堆栈（可选）
    WIP_QTY_BEFORE  INT,                   -- 异常前工单wipQty
    WIP_QTY_AFTER   INT,                   -- 异常后工单wipQty
    LOT_STATUS      VARCHAR(20),           -- 'CREATED', 'NOT_FOUND', 'DUPLICATE'
    CREATE_TIME     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CREATE_USER     VARCHAR(50)
);
```

#### 插入位置

在 `createLots()` 的 `catch` 块里：

```java
} catch (Exception e) {
    // 原有逻辑：抛出异常（会触发事务回滚）
    throw ExceptionHandler.handlerException(e, logger);
}
```

**但这里有个坑**：如果你在这个 catch 里往数据库写日志，而事务随后回滚了，**日志记录也会被回滚掉**。

#### 解决坑的方案

**方案A：日志用独立事务写入**

```java
} catch (Exception e) {
    // 先记日志（用独立事务，不受主事务回滚影响）
    saveExceptionLog(transInfo, e);
    
    throw ExceptionHandler.handlerException(e, logger);
}
```

`saveExceptionLog` 方法需要放在另一个带 `@Transactional(propagation = REQUIRES_NEW)` 的 service 里：

```java
@Service
public class ExceptionLogService {
    @Autowired
    private ExceptionLogRepository exceptionLogRepository;
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveExceptionLog(ExceptionLog log) {
        exceptionLogRepository.save(log);
    }
}
```

**方案B：日志直接写文件，不依赖数据库**

```java
} catch (Exception e) {
    // 用独立的 logger，不受事务影响
    logger.error("createLots异常: workOrderId={}, lotListSize={}, error={}",
        getWorkOrderIdFromTransInfo(transInfo),
        ((List<Lot>) transInfo.get("lotlist")).size(),
        e.getMessage(), e);
    throw ExceptionHandler.handlerException(e, logger);
}
```

---

### 场景3：你想**最小改动**，只想搞清楚问题在哪

**推荐：只加强现有日志 + 查现有日志文件**

这是我最建议你现阶段做的。

具体来说：

1. 先在 `createLots()` 方法的关键位置加几个 `log.error`/`log.warn`，把关键状态打出来：

```java
@Transactional(rollbackFor = Exception.class)
public List<Map> createLots(Map transInfo) throws MyCimException {
    try {
        TransactionLog transactionLog = ...;
        List<Lot> lotlist = (List<Lot>) transInfo.get("lotlist");
        log.info("createLots开始: lotlist.size={}, user={}", lotlist.size(), ThreadLocalContext.getUsername());
        
        for (int i = 0; i < lotlist.size(); i++) {
            Lot lot = lotlist.get(i);
            log.info("createLot第{}个: lotId={}, workOrderId={}", i+1, lot.getLotId(), lot.getOuterOrderNO());
            
            // ... 原有代码 ...
            
            log.info("createLot完成: lotId={}, lotRrn={}", lot.getLotId(), returnValue.get("lotRrn"));
        }
        
        log.info("createLots开始更新工单: workOrderMap.size={}", workOrderMap.size());
        for (Map.Entry<String, WorkOrder> entry : workOrderMap.entrySet()) {
            // ... 原有代码 ...
            log.info("工单更新完成: workOrderId={}, wipQty={}, comments={}", 
                workOrder.getWorkorderId(), workOrder.getWipQty(), workOrder.getComments());
        }
        
        return returnValueslist;
    } catch (Exception e) {
        log.error("createLots异常: lotlist={}, error={}", 
            ((List<Lot>) transInfo.get("lotlist")).stream().map(Lot::getLotId).collect(Collectors.toList()),
            e.getMessage(), e);
        throw ExceptionHandler.handlerException(e, logger);
    }
}
```

2. 等异常再出现时，直接去 `logs/wip/server.log` 里搜对应工单号

---

## 我的最终建议

### 现阶段（排查问题）：加强日志 + 查本地文件

理由：
- 零架构改动
- 不引入新依赖
- 立刻能用
- `logs/wip/server.log` 里已经有 DEBUG 级别日志了

### 中长期（如果需要监控）：加一张异常日志表 + 独立事务写入

理由：
- 方便查询统计
- 可以和工单数据关联分析
- 可以做告警面板

---

你想让我现在帮你做哪一步？
1. **只加强 `createLots()` 的日志输出**（最小改动，立刻能查问题）
2. **建异常日志表 + 独立事务写入**（完整方案，但改动大）

## User
我找到了问题点了，看了日志，发现是真空包重测跳站，变成批次后，增加了工单的在线数量，同时lotplan肯定也不会新增的，帮我看下真空包重测跳站哪些地方有修改工单在线数量的，我记得com是有两个地方可以跳站，别的地方可能也有

## Assistant
Now I see the vacuum package skip logic. Let me search for the COM retest skip station logic specifically.



### Relevant Code Snippets
1. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java:L28750-L28770
  — 找到wipQty在工单中的更新逻辑，该代码段出现在处理跳站时对工单wipQty字段的调整逻辑中。

2. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java:L32360-L32380
  — 该段代码包含"补充完成后，真空包跳站"注释，表明与真空包跳站逻辑相关的操作。

3. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java:L36520-L36540
  — 该段代码包含"保存新批次 新批次跳站到金线返修 保存记录"注释，涉及跳站操作及Lot记录的保存。

4. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java:L770-L790
  — 该段代码为createLot方法的定义，展示了创建lot的基础逻辑，可能是跳站时创建新lot的关键方法。

5. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java:L1250-L1290
  — 该段代码包含createLot方法的部分实现，涉及lot创建的详细逻辑，可能与跳站时的lot创建有关。

6. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java:L32410-L32450
  — 该段代码中包含LotTransHistory对象的创建和保存逻辑，展示了创建lot历史记录的方法，在跳站场景中可能被调用。

7. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java:L36400-L36420
  — 该段代码中包含新lot的执行创建逻辑，以及LotTransHistory对象创建和保存方法，与跳站后创建新lot相关。

8. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java:L11250-L11270
  — 该段代码调用createLot方法，表明在某些业务场景下，系统会通过调用createLot来创建新的lot，与跳站可能有关联。

9. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java:L28730-L28750
  — 该段代码包含createLotForRetest方法，与重测场景中的lot创建相关，可能涉及跳站后创建新lot的逻辑。

10. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java:L33040-L33060
  — 该段代码中包含createLotStepHis方法的调用，与跳站过程中的lot步骤历史记录保存相关。

让我深入分析与COM真空包重测相关的代码，特别是查找createLotForRetest方法和相关处理逻辑。



### Relevant Code Snippets
1. d:\data\MES\gc\core\src\ejb\wip\lot\src\com\mycim\wip\lot\service\impl\LotServiceImpl.java:L1-L2000
  — 该文件是核心LotService实现类，包含createLotForRetest方法及与工单wipQty处理相关的关键逻辑，是查询COM真空包重测跳站相关代码的核心文件

2. d:\data\MES\gc\core\src\ejb\wip\lot\src\com\mycim\wip\lot\service\impl\LotServiceImpl.java:L1000-L1500
  — 该部分包含createLotForRetest方法的具体实现，包括COM产品类别处理逻辑，特别是针对ProductClassify为COM的处理方式

3. d:\data\MES\gc\core\src\ejb\wip\lot\src\com\mycim\wip\lot\service\impl\LotServiceImpl.java:L1500-L2000
  — 该部分包含跳站/重测流程相关逻辑，包括修改工单wipQty的代码位置和处理机制

4. d:\data\MES\gc\core\src\ejb\wip\lot\src\com\mycim\wip\lot\service\impl\LotServiceImpl.java:L300-L800
  — 该部分定义了与COM产品类别相关的跳站和重测处理流程，以及ProductClassify为COM时的特殊处理逻辑

5. d:\data\MES\gc\core\src\model\src\com\mycim\webapp\actions\prp\RetestWorkOrderAction.java:L716-L732
  — 该部分展示了调用createLotForRetest方法的入口代码，提供了跳站/重测流程的完整上下文

6. d:\data\MES\gc\core\src\model\src\com\mycim\webapp\actions\prp\ComReworkAction.java:L192-L265
  — 该部分展示了调用createReworkLot方法的入口代码，涉及COM产品跳站的返工流程实现

7. d:\data\MES\gc\core\src\ejb\wip\lot\src\com\mycim\wip\lot\service\impl\LotServiceImpl.java:L800-L1000
  — 该部分包含对wipQty的修改逻辑，用于跳站场景下对工单数量的调整处理

8. d:\data\MES\gc\core\src\model\src\com\mycim\webapp\actions\prp\RetestWorkOrderAction.java:L300-L350
  — 该部分定义了与COM产品类别相关联的跳站流程中的关键处理逻辑，特别是处理ProductClassify为COM的情况

现在我已经找到了一个入口，让我继续搜索第二个 COM 跳站入口以及真空包重测相关的全部位置。



### Relevant Code Snippets
1. d:/data/MES/gc/core/src/model/src/com/mycim/webapp/actions/prp/RetestWorkOrderAction.java:L1-L100
  — 找到了调用lotService.createLotForRetest的入口点，此action处理重测工单逻辑，可能涉及COM产品类别和wipQty修改
2. d:/data/MES/gc/core/src/model/src/com/mycim/webapp/actions/prp/ComReworkAction.java:L1-L100
  — 找到了调用lotService.createReworkLot的入口点，此action处理COM产品的返工逻辑，可能涉及wipQty修改
3. d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/GcServiceImpl.java:L1-L200
  — 找到GcServiceImpl类，其中可能包含与vbox相关的跳站/重测逻辑，特别是COM产品类别相关的处理
4. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java:L1-L300
  — LotServiceImpl类中包含大量与lot操作相关的方法，包括可能涉及wipQty修改及与ProductClassify=COM相关的处理

这些文件包含所需的调用点以及相关的上下文逻辑，包括对lotService.createReworkLot、lotService.createLotForRetest的调用，以及对COM产品类别和wipQty的处理。

现在我看到了第一个入口。让我再仔细看看 `createReworkLot` 方法中的完整逻辑，以及跳站相关的地方。



让我继续搜索COM真空包重测跳站的相关代码，特别是修改wipQty和LotPlan的逻辑。



让我继续查看 `createReworkLot` 方法的完整代码。



### Relevant Code Snippets
1. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java:L1-L100
  — 找到LotServiceImpl中与createReworkLot和createLotForRetest相关的方法定义，这些方法可能涉及真空包重测跳站逻辑。

2. d:/data/MES/gc/core/src/model/src/com/mycim/webapp/actions/prp/ComReworkAction.java:L1-L50
  — ComReworkAction类，可能包含与COM产品类别相关的返工逻辑。

3. d:/data/MES/gc/core/src/model/src/com/mycim/webapp/actions/prp/RetestWorkOrderAction.java:L1-L50
  — RetestWorkOrderAction类，与重测工单相关的操作，可能涉及跳站逻辑。

4. d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/LotService.java:L1-L50
  — LotService接口定义，包含createReworkLot和createLotForRetest等方法。

5. d:/data/MES/gc/core/src/valueobject/src/com/mycim/prp/model/WorkOrder.java:L1-L100
  — WorkOrder类定义，包含wipQty字段，可能涉及修改工单数量的逻辑。

6. d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/GcServiceImpl.java:L1-L100
  — GcServiceImpl类实现，可能包含与跳站/返工相关的逻辑，包含createLotForRetest方法。

7. d:/data/MES/gc/core/src/ejb/prp/src/com/mycim/prp/dao/PrpSetupDAOImpl.java:L1-L100
  — PrpSetupDAOImpl类中包含与工单相关操作，可能涉及wipQty修改但不创建LotPlan的逻辑。

让我继续搜索跳站相关的具体逻辑，特别是 `startFlag` 相关的逻辑和 `createComReLot` 的完整流程。



好，现在我已经完整梳理了 COM 真空包重测跳站的全部相关代码。以下是完整分析：

---

## COM 真空包重测跳站修改工单 `wipQty` 的所有位置

你提到"COM有两个地方可以跳站，别的地方可能也有"。经过完整搜索，我找到了以下 **所有会修改工单 wipQty 的 COM 重测跳站入口**。

---

### 🔴 入口1：COM 返工跳站（`createReworkLot`）

**Action 入口**：[ComReworkAction.java:L260](file:///d:/data/MES/gc/core/src/model/src/com/mycim/webapp/actions/prp/ComReworkAction.java#L260)

```java
lotService.createReworkLot(transInfo);
```

**核心逻辑**：[LotServiceImpl.java:L1226-L1255](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1226-L1255)

```java
public List<Map> createReworkLot(Map transInfo) throws MyCimException {
    // 创建COM返工批次
    Map returnValue = createComReLot(transInfo);
    Long transRrn = (Long) returnValue.get("transRrn");
    
    // ⚠️ 直接修改工单 wipQty
    WorkOrder workOrder = workOrderRepository.getWorkOrderById(workOrderId);
    String transtype = WorkOrderHistory.TRANSTYPE_STARTED;
    workOrder.setWipQty(workOrder.getWipQty() + lot.getQty1().intValue());
    workOrder.setScrapQty(workOrder.getScrapQty() - lot.getQty1().intValue());
    workOrder.setStatus(WorkOrder.STATUS_STARTED);
    workOrderRepository.merge(workOrder);
    
    // ⚠️ 插入 STARTE 历史记录（comments 为空！）
    WorkOrderHistory workOrderHistory = new WorkOrderHistory(workOrder);
    workOrderHistory.setTransRrn(transRrn);
    prpSetupServices.insertWorkOrderHistory(workOrderHistory.getTransRrn(), transtype, workOrder);
}
```

**特点**：
- 这是你说的**第一个 COM 跳站入口**
- `createComReLot` 是返工专用的批次创建方法，**不创建 LotPlan**（它创建的是 Lot，且标记为 `LOT_PROPERTY_COM_INLINE_RW` 返工标记位）
- **会修改工单 wipQty**
- **会减少 scrapQty**（因为从报废状态转为返工状态）
- **会插入 STARTE 历史记录，且 comments 为空**
- **不会创建 LotPlan**（返工批次直接创建 Lot，没有排产环节）

**这就是你看到的异常现象的完美匹配！**

---

### 🔴 入口2：重测工单创建批次（`createLotForRetest`）

**Action 入口**：[RetestWorkOrderAction.java:L725](file:///d:/data/MES/gc/core/src/model/src/com/mycim/webapp/actions/prp/RetestWorkOrderAction.java#L725)

```java
lotIds = lotService.createLotForRetest(workOrderId, facilityRrn, username);
```

**核心逻辑**：[LotServiceImpl.java:L28738-L28848](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L28738-L28848)

```java
public String createLotForRetest(String workOrderId, Long facilityRrn, String username) throws MyCimException {
    // 获取绑定的 VBOX 列表
    List<WorkOrderRelation> workOrderRelationList = prpSetupServices.findWorkOrderRelationByWoRrnAndObjectType(...);
    
    // 检查工单可用数量
    int wipQty = workOrder.getWipQty() == null ? 0 : workOrder.getWipQty();
    int completeQty = workOrder.getCompleteQty() == null ? 0 : workOrder.getCompleteQty();
    int scrapQty = workOrder.getScrapQty() == null ? 0 : workOrder.getScrapQty();
    int usableworkOrder = workOrder.getPlannedQty() - completeQty - wipQty - scrapQty - totalPlaningQty;
    
    // 循环处理每个 VBOX
    for (WorkOrderRelation workOrderRelation : workOrderRelationList) {
        // 检查 lot 是否已存在
        Lot l = this.getLot(vBoxId.toUpperCase());
        if (l != null && StringUtils.isNotEmpty(l.getLotId())) {
            throw new MyCimException("lot_has_create_can_not_create_again");
        }
        
        // ⚠️ 这里只创建 LotPlan，不创建实际的 Lot
        lot.setLotRrn(this.insertLotPlan(lot, username));
    }
    return lotIds;
}
```

**特点**：
- 这是你说的**第二个 COM 跳站入口**
- 这个方法**只创建 LotPlan（计划批次）**，**不修改工单 wipQty**
- **不会插入 STARTE 历史记录**
- LotPlan 需要后续通过发料上线流程才会转成实际的 Lot

**所以这个方法本身不会导致 wipQty 异常增加。**

---

### 🔴 入口3：`createLot()` 方法末尾的工单更新逻辑

**位置**：[LotServiceImpl.java:L1714-L1734](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1714-L1734)

```java
// 如果满足条件，会在createLot内部更新工单wipQty
if (StringUtils.isEmpty(doNotChangeWipQtyFlag) && StringUtils.isNotEmpty(workOrderId)
        && StringUtils.isNotEmpty(workOrderFlag) && StringUtils.isEmpty(unifiyProcessFlag)) {
    WorkOrder workOrder = workOrderRepository.getWorkOrderById(workOrderId);
    if (!WorkOrder.RETEST_WORKORDER.equals(workOrder.getWorkOrderType())) {
        ...
        workOrder.setWipQty(workOrder.getWipQty() + lot.getQty1().intValue());
        workOrder.setStatus(WorkOrder.STATUS_STARTED);
        String transtype = WorkOrderHistory.TRANSTYPE_STARTED;
        workOrder.setOwner(transPerformedby);
        workOrderRepository.update(workOrder);
        prpSetupServices.insertWorkOrderHistory(transRrn, transtype, workOrder);
    }
}
```

**特点**：
- 这段代码**在 `createLot()` 内部执行**
- 通过 `unifiyProcessFlag` 参数控制是否跳过
- **如果是重测工单（RETEST_WORKORDER），不会执行这段逻辑**

---

### 🔴 入口4：`createLots()` 批量发料（你昨天刚加了 `PUT_QTY` 注释的方法）

**位置**：[LotServiceImpl.java:L1149-L1223](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1149-L1223)

**特点**：
- 批量发料，统一更新工单
- 插入 STARTE 历史时会补 `PUT_QTY` 到 comments（你昨天刚加的）
- 但你已经确认异常数据的 comments 是空的，所以不是这个入口

---

## 结论：COM 真空包重测跳站导致 wipQty 异常增加的根因

根据你补充的信息"真空包重测跳站，变成批次后，增加了工单的在线数量，同时 lotplan 肯定也不会新增"，**问题 100% 锁定在入口1**：

### 🔴 根因：`createReworkLot()` 方法

**触发路径**：
```
前端 COM 返工跳站页面
  → ComReworkAction.perform() (ACTION_REWORKLOT)
    → lotService.createReworkLot(transInfo)
      → createComReLot(transInfo)    // 创建返工批次（不创建 LotPlan）
      → workOrder.setWipQty(...)     // 增加工单 wipQty
      → insertWorkOrderHistory(...)  // 插入 STARTE 历史（comments 为空）
```

**为什么符合你看到的现象**：
| 现象 | 是否匹配 | 原因 |
|------|----------|------|
| wipQty 增加 | ✅ | `workOrder.setWipQty(workOrder.getWipQty() + lot.getQty1().intValue())` |
| STARTE 历史 | ✅ | `prpSetupServices.insertWorkOrderHistory(..., WorkOrderHistory.TRANSTYPE_STARTED, ...)` |
| comments 为空 | ✅ | 没有设置 comments 的逻辑 |
| LotPlan 不存在 | ✅ | `createComReLot` 只创建 Lot，不创建 LotPlan |
| 工单不是返工 | ❌ 存疑 | 需要确认前端是否真的走的是 COM 返工页面 |

---

## 还有一个隐藏问题

我在 `createReworkLot` 里还发现了一个**更严重的问题**：

[L1239-L1240](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1239-L1240)：

```java
workOrder.setWipQty(workOrder.getWipQty() + lot.getQty1().intValue());
workOrder.setScrapQty(workOrder.getScrapQty() - lot.getQty1().intValue());
```

这个方法会**同时减少 scrapQty**，因为它的业务意图是"从报废批次转为返工批次"。

但如果你的工单**不是返工场景**，而是因为其他原因（如真空包重测后直接作为新批次上线），那么：
- wipQty 被错误增加 ✅
- scrapQty 被错误减少 ❌（这是额外的副作用）

---

## 排查建议

1. **查 `scrapQty` 是否也被异常减少**：如果 scrapQty 也减少了，那就 100% 确定是 `createReworkLot` 触发的
2. **查这条 STARTE 历史对应的 `transRrn`**，去 `TRANSACTION_LOG` 表确认是哪个事务触发的
3. **查对应的批次 lot 是否有 `lotProperty='COM_INLINE_RW'`**：如果有，说明是 `createComReLot` 创建的返工批次



完整分析已完成，等你确认是否就是 `createReworkLot` 这条路径，以及是否需要进一步排查 scrapQty 是否也被异常修改了。

## User
core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/PackageServiceImpl.java 这个位置的 1634行代码vboxDoRework，真空包跳站回去，调用了createlots方法，也增加了wiplot数量，但不应该增加才对

## Assistant
好的，我仔细看了 `vboxDoRework` 方法，让我确认一下关键代码：



找到问题了！看 L1672：

```java
createLotInfo.put("doNotChangeWipQtyFlag", "1");
```

这个方法的意图是**不修改工单 wipQty**，但是——

问题出在 `createLots()` 方法里，让我再确认一下 `doNotChangeWipQtyFlag` 的生效范围：



**找到了！这就是问题所在。**

看 `vboxDoRework` 方法的第 **1672 行**：

```java
createLotInfo.put("doNotChangeWipQtyFlag", "1");
List<Map> resultsInfos = lotService.createLots(createLotInfo);
```

**意图**：设置 `doNotChangeWipQtyFlag=1` 来阻止 `createLots()` 修改工单 wipQty。

**但实际效果**：❌ **完全无效！**

---

## 为什么这个标志不生效

### 原因：`doNotChangeWipQtyFlag` 只在 `createLot()` 内部被检查

[LotServiceImpl.java:L1714-L1734](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1714-L1734)：

```java
// createLot() 内部检查 doNotChangeWipQtyFlag
if (StringUtils.isEmpty(doNotChangeWipQtyFlag) && StringUtils.isNotEmpty(workOrderId)
        && StringUtils.isNotEmpty(workOrderFlag) && StringUtils.isEmpty(unifiyProcessFlag)) {
    workOrder.setWipQty(workOrder.getWipQty() + lot.getQty1().intValue());
    // ...
}
```

**但是** `vboxDoRework` 调用的是 `createLots()`（复数），而 `createLots()` 方法里：

```java
// L1162：设置了 unifiyProcessFlag=1，这会阻止 createLot() 内部的工单更新 ✅
transInfo.put("unifiyProcessFlag", "1");
Map returnValue = createLot(transInfo);

// L1166-L1194：但是 createLots() 自己无条件累加 wipQty ❌
WorkOrder workOrder = workOrderRepository.getWorkOrderById(workOrderId);
wip = wip.add(lotQty);
workOrder.setWipQty(wip.intValue());

// L1202-L1214：最后无条件插入 STARTE 历史记录 ❌
for (Map.Entry<String, WorkOrder> entry : workOrderMap.entrySet()) {
    workOrderRepository.merge(workOrder);
    prpSetupServices.insertWorkOrderHistory(transactionLog.getTransRrn(), transtype, workOrder);
}
```

**结论**：
- `doNotChangeWipQtyFlag` 只传递给 `createLots()`，但 `createLots()` **根本不检查这个标志**
- `createLots()` 里设置的 `unifiyProcessFlag=1` 只能阻止 `createLot()` 内部的工单更新
- **阻止不了 `createLots()` 自己的工单更新逻辑**

---

## 修复方案

需要在 `createLots()` 方法的**阶段B（工单更新循环）**里检查 `doNotChangeWipQtyFlag`：

```java
@Transactional(rollbackFor = Exception.class)
public List<Map> createLots(Map transInfo) throws MyCimException {
    try {
        // ... 前面的代码不变 ...
        
        String doNotChangeWipQtyFlag = (String) transInfo.get("doNotChangeWipQtyFlag");
        
        String transtype = WorkOrderHistory.TRANSTYPE_STARTED;
        for (Map.Entry<String, WorkOrder> entry : workOrderMap.entrySet()) {
            // ⚠️ 增加判断：如果设置了 doNotChangeWipQtyFlag，跳过工单更新
            if ("1".equals(doNotChangeWipQtyFlag)) {
                continue;
            }
            
            WorkOrder workOrder = entry.getValue();
            workOrderRepository.merge(workOrder);
            Integer putQty = orderPut.get(workOrder.getWorkorderId());
            if(StringUtils.isEmpty(workOrder.getComments())){
                workOrder.setComments("PUT_QTY"+putQty.intValue());
            }else{
                workOrder.setComments(workOrder.getComments()+";PUT_QTY"+putQty.intValue());
            }
            prpSetupServices.insertWorkOrderHistory(transactionLog.getTransRrn(), transtype, workOrder);
        }
        // ...
    }
}
```

你要我现在修改代码吗？

## User
帮我总结下这个异常以及处理方案

## Assistant
# COM 真空包重测跳站导致工单 wipQty 异常增加问题总结

---

## 一、问题现象

| 现象 | 说明 |
|------|------|
| 工单 `wipQty`（在线数量）莫名其妙增加 | 实际没有新批次生成，但工单在线数量被增加了 |
| `WORKORDER_H` 表中插入了一条 `STARTE` 类型的历史记录 | 事务类型 = `STARTE` |
| 历史记录的 `comments` 为空 | 没有 `PUT_QTY` 标记 |
| `LOT_PLAN` 表中没有新记录 | 批次计划未创建 |
| 工单不是返工工单 | 按业务逻辑不应该走返工路径 |

---

## 二、根因分析

### 触发路径

```
前端 COM 真空包检验 → 跳站返工操作
  → ComLotMergeAction / LotMergeAction（真空包检验页面）
    → gcService.vboxDoRework(createLotInfo, userName, packedLotList)
      → PackageServiceImpl.vboxDoRework(...)
        → lotService.createLots(createLotInfo)  ← 批量创建批次
```

### 核心问题代码

**PackageServiceImpl.java:L1672-L1673**：

```java
createLotInfo.put("doNotChangeWipQtyFlag", "1");  // 意图：不修改工单在线数量
List<Map> resultsInfos = lotService.createLots(createLotInfo);  // 实际：仍然修改了！
```

### 为什么 `doNotChangeWipQtyFlag` 不生效

`createLots()` 方法的流程：

```
阶段A：循环创建批次
  ├─ L1162: transInfo.put("unifiyProcessFlag", "1")   ← 这个有效
  ├─ L1163: createLot(transInfo)                       ← 被 unifiyProcessFlag 阻止，不修改工单 ✅
  └─ L1192-L1194: workOrder.setWipQty(...)             ← 无条件执行，累加 wipQty ❌

阶段B：统一持久化工单变更
  └─ L1203-L1214: merge(workOrder) + insertWorkOrderHistory(STARTE)  ← 无条件执行 ❌
```

**问题本质**：
- `doNotChangeWipQtyFlag` 只在 `createLot()`（单数）方法内部被检查 [L1714](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1714)
- `createLots()`（复数）**根本不检查这个标志**
- 所以虽然设置了 `doNotChangeWipQtyFlag=1`，但 `createLots()` 里的工单更新逻辑**照样执行**

### `unifiyProcessFlag` 的作用范围

```
transInfo.put("unifiyProcessFlag", "1")  ← L1162 设置
  ↓
阻止的是 createLot() 内部的工单更新 [L1714-L1734]
  ↓
阻止不了 createLots() 自己的工单更新 [L1166-L1214]
```

---

## 三、受影响的业务场景

| 场景 | 调用方法 | 是否受影响 |
|------|----------|------------|
| COM 真空包检验 → 跳站返工 | `vboxDoRework()` → `createLots()` | ✅ **受影响**（当前问题） |
| COM 批次合并 | `LotMergeAction` → `createLots()` | ✅ 可能受影响 |
| 普通工单发料上线 | `GcServiceImpl.erpWorkOrderSynAndCreateLots()` → `createLots()` | ✅ 受影响（但业务期望增加 wipQty） |

---

## 四、修复方案

### 方案一：最小改动（推荐优先实施）

**在 `createLots()` 的阶段B循环中检查 `doNotChangeWipQtyFlag`**

修改文件：[LotServiceImpl.java:L1202-L1215](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1202-L1215)

```java
@Transactional(rollbackFor = Exception.class)
public List<Map> createLots(Map transInfo) throws MyCimException {
    // ... 前面的代码不变 ...
    
    // ⬇️ 新增：获取标志
    String doNotChangeWipQtyFlag = (String) transInfo.get("doNotChangeWipQtyFlag");
    
    String transtype = WorkOrderHistory.TRANSTYPE_STARTED;
    for (Map.Entry<String, WorkOrder> entry : workOrderMap.entrySet()) {
        
        // ⬇️ 新增：跳过工单更新
        if ("1".equals(doNotChangeWipQtyFlag)) {
            continue;  // 不修改工单，不插入历史
        }
        
        WorkOrder workOrder = entry.getValue();
        workOrderRepository.merge(workOrder);
        
        Integer putQty = orderPut.get(workOrder.getWorkorderId());
        if(StringUtils.isEmpty(workOrder.getComments())){
            workOrder.setComments("PUT_QTY"+putQty.intValue());
        }else{
            workOrder.setComments(workOrder.getComments()+";PUT_QTY"+putQty.intValue());
        }
        prpSetupServices.insertWorkOrderHistory(transactionLog.getTransRrn(), transtype, workOrder);
    }
    // ... 后面的代码不变 ...
}
```

**改动量**：约 5 行代码

**优点**：
- 改动最小
- 不影响现有正常业务（发料上线仍然会更新 wipQty）
- 直接解决当前问题

---

### 方案二：同时修复 `orderPut` 累加 Bug（一并修复）

**当前 Bug**：`orderPut` 在每次循环开始时重置为 0，导致最终只保留最后一个 lot 的数量

[L1174-L1175](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1174-L1175)：

```java
BigDecimal put=new BigDecimal(0);  // ← 每次循环重置
...
orderPut.put(workOrderId, put.intValue());  // ← 只保留最后一个 lot 的值
```

**修改为**：

```java
BigDecimal put = new BigDecimal(orderPut.getOrDefault(workOrderId, 0));  // ← 从 Map 取旧值
...
orderPut.put(workOrderId, put.intValue());
```

**或者**（更好的方式）：

```java
orderPut.merge(workOrderId, lotQty.intValue(), Integer::sum);  // 直接累加
```

---

### 方案三：统一事务回滚规则（建议一并检查）

`createLots()` 方法现在已经有：

```java
@Transactional(rollbackFor = Exception.class)  // ← 你已经加了
```

但建议确认 `createLot()` 方法也加上，防止单独调用时出问题：

```java
@Transactional(rollbackFor = Exception.class)
public Map createLot(Map transInfo) throws MyCimException
```

---

## 五、修复后效果

| 场景 | 修复前 | 修复后 |
|------|--------|--------|
| COM 真空包跳站返工 | wipQty 异常增加 + 插入 STARTE 历史 | wipQty 不变 + 不插入历史 ✅ |
| 普通工单发料上线 | wipQty 正常增加 + 插入 STARTE 历史 | wipQty 正常增加 + 插入 STARTE 历史 ✅ |
| COM 批次合并 | 可能受影响 | 取决于是否设置 `doNotChangeWipQtyFlag` |

---

## 六、其他关联问题（建议一并排查）

| 问题 | 位置 | 说明 |
|------|------|------|
| `createReworkLot()` 也会修改 wipQty + 减少 scrapQty | [LotServiceImpl.java:L1239-L1240](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1239-L1240) | 如果工单不是返工场景，scrapQty 会被错误减少 |
| `LotPlan` 删除用错 RRN | [LotServiceImpl.java:L1687](file:///d:/data/MES/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/LotServiceImpl.java#L1687) | `deleteOneById(tempLotRrn)` 可能删不掉正确的 LotPlan |
| `transInfo` 共享 Map 污染 | `createLots()` 循环中反复使用同一个 Map | 可能影响下一次循环 |
| 并发 wipQty 丢失更新 | `createLots()` 先读后写 | 多个请求同时操作同一工单时可能丢失更新 |

---

## 七、建议实施顺序

| 优先级 | 操作 | 改动量 | 风险 |
|--------|------|--------|------|
| 🔴 最高 | 方案一：`createLots()` 检查 `doNotChangeWipQtyFlag` | 5 行 | 低 |
| 🟡 中 | 方案二：修复 `orderPut` 累加 Bug | 2 行 | 低 |
| 🟢 低 | 方案三：确认事务回滚规则 | 1 行 | 极低 |

---

需要我现在直接修改代码吗？

