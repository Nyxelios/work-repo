---
title: WltAutoPlanJob-开发视角流程说明
date: 2026-06-12
author: Codex
status: completed
tags: [MES, WLT, 自动排料, 开发说明]
---

# WltAutoPlanJob 开发视角流程说明

> **状态**: completed
> **创建日期**: 2026-06-12
> **最后更新**: 2026-06-12
> **说明**: 本文按代码执行顺序说明 `WltAutoPlanJob` 的核心逻辑，保留开发排查最需要关注的判断和关键方法，不展开过细的字段级细节。

---

## 一、文档目的

这份文档主要回答 3 个问题：

1. `WltAutoPlanJob` 是怎样从定时任务入口一路走到自动排料的。
2. 哪些配置会被关闭，哪些来料会被过滤掉。
3. 真正进入排料后，工单、`LotPlan`、关系表和日志分别在什么位置落库。

## 二、主流程概览

从代码结构上看，WLT 自动排料主要分成 4 层：

1. `WltAutoPlanJob.execute(...)`
作用：定时任务入口，控制并发、环境校验、循环执行配置、统一收尾。

2. `WorkOrderPlanServiceImpl.wltAutoPlan(config)`
作用：单条配置的主处理方法，负责配置校验、待排批次筛选、更新配置累计数量。

3. `WorkOrderPlanServiceImpl.splicingWltCode(config)`
作用：把配置里的二级代码拼成查询条件。

4. `WorkOrderPlanServiceImpl.wltAutoPlanInfo(useList, config, model)`
作用：对真正可排的数据执行建工单、建 `LotPlan`、绑定关系、回写 WMS。

---

## 三、任务入口：`execute(...)`

入口代码在 [WltAutoPlanJob.java](D:/data/document/Github/work-archive/Code-Projects/gc/core/src/gc/src/main/java/com/mycim/gc/service/quartz/jobs/WltAutoPlanJob.java:31)。

### 3.1 并发控制

最前面先看静态变量 `lockKey`。

```java
if(lockKey>0){
    return;
}
lockKey=1;
```

这段逻辑很直接：

1. 如果上一次任务还没执行完，本次直接返回。
2. 只有拿到锁的这一次任务才继续往下跑。
3. 锁在 `finally` 中释放，所以正常结束和异常结束都会把 `lockKey` 还原成 0。

这一段是整个 Job 的第一道保护，避免重复排料。

### 3.2 环境校验

拿到锁后会读取本机 `serverId`，只允许以下服务器执行：

1. `MESAP01`
2. `MESAP02`
3. `RWAP01`
4. `RWAP02`

如果不在这个名单内，任务直接返回，不做排料。

### 3.3 初始化上下文和开始日志

环境通过后，任务会做几件初始化动作：

1. 记录 `WLT_AUTO_PLAN Start`。
2. 设置 `ThreadLocalContext.setUsername("WLT_AUTO")`。
3. 设置 `ThreadLocalContext.setFacilityRrn(1L)`。
4. 写一条自动排料开始日志 `saveAutoLog(starLog)`。

这里的重点不是业务逻辑，而是为后面创建工单、写履历、查日志提供统一上下文。

### 3.4 查询待执行配置并循环处理

入口层通过下面这行代码查待执行配置：

```java
List<GcAutoPlanConfig> waitConfigList = gcServiceImpl.queryWlaWaiting(Lot.LINE_TYPE_WLA);
```

后续逻辑是：

1. 如果配置列表为空，记录“无可执行自动排料信息”日志。
2. 如果不为空，逐条调用 `workPlanService.wltAutoPlan(config)`。
3. 如果 `wltAutoPlan(config)` 返回了 `productId`，则放入 `prodList`，等任务结束后统一通知。

这里有个很关键的设计点：

`execute(...)` 自己不处理配置细节，它只负责调度和收尾。真正的配置校验和排料都下沉在 `wltAutoPlan(config)` 里。

### 3.5 结束日志和通知

所有配置处理完成后，入口层还会做两件事：

1. 写 `WLT排料结束` 日志。
2. 如果 `prodList` 不为空，调用 `gcServiceImpl.sendWltAutoMsgInfo(prodList)` 发送通知。

这里的 `prodList` 不是“本次成功排料的产品”，而是“本次执行中被自动关闭配置对应的产品”。这一点在排查通知来源时很重要。

---

## 四、单条配置主流程：`wltAutoPlan(config)`

核心代码在 [WorkOrderPlanServiceImpl.java](D:/data/document/Github/work-archive/Code-Projects/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java:1977)。

这个方法可以概括成 5 步：

1. 校验配置是否仍然有效。
2. 拼接查询条件。
3. 查询候选来料并做过滤。
4. 如果有可用数据，则进入 `wltAutoPlanInfo(...)`。
5. 更新配置累计数量和履历。

### 4.1 配置有效性校验

第一步先按 `modelRrn` 查 `GcModelManagement`：

```java
GcModelManagement model = prpSetupService.getGcModelManagementInfoByObjectRrn(config.getModelRrn());
```

后面有两个关闭配置的分支。

#### 分支一：`model` 不存在

如果 `model == null`，当前配置直接失效，处理动作包括：

1. 记录异常日志。
2. `config.setIsOpen(0)` 关闭配置。
3. 更新 `GC_AUTO_PLAN_CONFIG`。
4. 写一条 `GcAutoPlanConfigH` 履历，`transType = CLOSE`。
5. 返回 `config.getIncomingProductId()`。

返回 `productId` 的目的不是继续排料，而是让外层任务后面统一发通知。

#### 分支二：配置和 `model` 不一致

即使 `model` 查到了，还会继续比两个字段：

```java
config.getTestModelId() vs model.getTestModelId()
config.getProcessId()   vs model.getRouteId()
```

只要其中任意一个不一致，也会按“关闭配置”处理，动作和上面基本一致：写日志、关配置、写 `ConfigH`、返回 `productId`。

这里是单条配置最关键的防呆逻辑。也就是说，WLT 自动排料不是只看配置本身，还会实时校验配置关联的主数据有没有变化。

### 4.2 `splicingWltCode(config)` 的作用

配置校验通过后，会调用 `splicingWltCode(config)` 生成查询二级代码。

代码位置在 [WorkOrderPlanServiceImpl.java](D:/data/document/Github/work-archive/Code-Projects/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java:2290)。

这段逻辑的关键点有两个：

1. 配置中的 `*` 会被转成 SQL 通配形式 `_`。
2. WLT 如果存在第二位代码，会默认补第三位 `_`，然后再看第四位。

也就是说，这里不是简单拼字符串，而是在按照 WLT 的二级代码规则构造 `RESERVED1 like queryCode` 的查询条件。

### 4.3 候选来料查询

拼好 `queryCode` 后，会调用仓储查询：

```java
wmsMmsMaterialLotRepository.getWltAutoPlanLot(config.getIncomingProductId(), queryCode)
```

这一层会把明显不符合条件的数据先过滤掉，比如：

1. 状态必须在库或新建态。
2. 不能是 Hold。
3. 不能已经挂工单。
4. 不能有 `RESERVED54`。
5. `RESERVED50 = 5`。
6. 产品、二级代码、仓别、等级都要匹配。

这一段更像“候选集预筛选”，还不是最终可排数据。

### 4.4 候选集二次过滤

候选来料查出来后，代码会继续逐批过滤，核心目标是组出真正可排的 `useList`。

过滤规则有两层。

#### 第一层：已经绑过工单的整批跳过

```java
prpSetupService.getWorkOrderRelationByMLoRrn(lot.getObjectRrn())
```

如果查到关系数据，说明这批料已经和工单建立过关系，直接记日志并 `continue`。

#### 第二层：批次下面只要有一片排过，就整批跳过

如果整批还没有绑工单，则继续查这个批次下面的 `unit`，逐片检查：

```java
lotPlanRepository.getLotPlanByLotId(mLotUnit.getUnitId())
```

只要发现任意一片已经有 `LotPlan`，就会：

1. 记录“已排料”日志。
2. 把当前批次标记为不可用。
3. 跳出当前批次的 `unit` 循环。

这意味着 WLT 的过滤策略比较严格：不是按“片”补排，而是按“整批是否完整未排”来决定是否进入自动排料。

### 4.5 生成 `useList` 和累计数量

如果某个批次通过了上面两层过滤，它会被加入 `useList`：

```java
useList.add(lot);
tempQty = tempQty + lot.getCurrentSubQty().intValue();
```

这里的 `tempQty` 是当前配置本次排料后要回写的累计数量，不是工单数量。

### 4.6 `useList` 为空和不为空的分支

最后有两个结果：

1. `useList` 为空：只记“没有查询到待排批次信息”日志，本条配置结束。
2. `useList` 不为空：调用 `wltAutoPlanInfo(useList, config, model)` 进入真正排料。

排料成功返回后，会继续：

1. `config.setInputQty(tempQty)`。
2. 写 `GcAutoPlanConfigH` 履历，`transType = UPDATE`。
3. 更新 `GC_AUTO_PLAN_CONFIG`。

所以 `wltAutoPlan(config)` 既负责筛数据，也负责维护配置侧的累计状态。

---

## 五、真正排料：`wltAutoPlanInfo(useList, config, model)`

核心代码在 [WorkOrderPlanServiceImpl.java](D:/data/document/Github/work-archive/Code-Projects/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java:2605)。

这个方法建议重点看 4 个点：

1. 工单是怎么分组复用的。
2. 工单数量是在哪一层累计的。
3. `LotPlan` 和 `WorkOrderRelation` 在哪里落库。
4. 最后怎样把结果回写到 WMS。

### 5.1 先准备排料依赖数据

方法开头会先查：

1. 产品 `product`
2. 流程 `process`
3. 流程扩展 `processExt`
4. 仓库列表并组装 `warehouseRrnNameMap`

这些数据本身不复杂，重点是后面建工单时都会用到。

### 5.2 `orderMap` 的作用

```java
Map<String, WorkOrder> orderMap = new HashMap<>();
```

`orderMap` 用来缓存本次任务里已经创建过的工单，避免同一组特征的数据重复建工单。

分组 key 是：

```java
materialName + reserved6 + reserved50 + reserved1 + factoryId
```

如果 `config.getBonded() = Y`，还会再拼上 `reserved27`。

这段是 WLT 自动排料里最值得注意的工单分组策略。也就是说，工单不是“一批料一张”，而是“相同关键属性的批次共用一张”。

### 5.3 新工单创建逻辑

如果 `orderMap` 里没有当前 key，对应逻辑是：

1. 调用 `buildWltWorkorderInfo(mLot, config, product)` 组装工单基础字段。
2. 设置 `plannedQty` 和 `planPutInto` 初值。
3. 根据 `reserved13` 找仓库名，设置 `location`。
4. 根据 `workOrderType` 查 `orderType`，并写入 `reserved59`。
5. 创建并保存 `GcWorkorderDetail`。
6. 设置 `processType`。
7. 调用 `workOrderService.createWorkOrder(workOrder)` 正式建单。
8. 将工单状态改为 `ISSUE`。
9. 放入 `orderMap`。

这里真正生成工单的是 `createWorkOrder(...)`，前面只是补齐建单所需字段。

### 5.4 已有工单复用逻辑

如果当前 key 在 `orderMap` 里已经有工单，则不会重复建单，而是直接复用，并把数量累加到这张工单：

```java
int planQyt = workOrder.getPlannedQty() + mLot.getCurrentSubQty().intValue();
workOrder.setPlannedQty(planQyt);
workOrder.setPlanPutInto(planQyt);
```

这里要注意，WLT 在进入单批处理前，先按批次的 `currentSubQty` 预加了一次工单数量。

### 5.5 `LotPlan` 创建逻辑

每处理一批数据，都会先开启一条事务日志：

```java
TransactionLog transactionLog = baseService.startTransactionLog("AUTO_PLAN", LotPlanHistory.TRANS_TYPE_CREATE);
```

然后逐片检查 `unit`：

```java
LotPlan lp = lotPlanRepository.getLotPlanByLotId(mLotUnit.getUnitId());
```

如果当前片已经有 `LotPlan`：

1. 记录“已排料”日志。
2. 把前面预加到工单上的数量减 1。
3. 更新工单。

这一步很关键，因为它解释了为什么前面先按批次数量预加，后面又在片级发现重复时回退数量。

如果当前片还没有 `LotPlan`，则会：

1. 调用 `saveWltLotInfo(...)` 生成 Lot 基础对象。
2. 补充 `lotId`、`qty1`、`waferMark`、`materialLotId` 等字段。
3. 调用 `saveLotPlan(lot, transactionLog.getTransRrn())` 保存 `LotPlan`。

所以从落库点来看，`LotPlan` 真正创建的位置就在这里。

### 5.6 工单关系绑定逻辑

每成功创建一片的 `LotPlan` 后，紧接着会执行：

```java
saveWorkOrderRelationAndSaveHis(workOrder, mLotUnit, null, WorkOrderRelationH.WO_RESEVED_MLOT);
```

这一行同时说明了两件事：

1. 工单和来料片的绑定关系是在这里建立的。
2. 关系履历也是在这里一起写入的。

如果后面要查“为什么某片已经被认定为已排料”或“为什么某批被判断成已绑工单”，这一层关系数据是核心依据。

### 5.7 批次回写和 WMS 更新

当前批次下所有片处理完成后，会把工单信息回写到当前批次对象，包括：

1. `workOrderId`
2. `planPutTime`
3. `ftWorkorderId`
4. `reserved59`
5. `reserved60`
6. `customerDevice`
7. `bpmUnshipNote`

然后把它加入 `updateList`。

全部批次处理完后，统一调用：

```java
gcService.updateWMSAutoPlanHis(updateList, WmsMmsMaterialLotUnitHis.TRANSTYPE_BIND_WORKORDER);
```

也就是说，WMS 侧回写不是逐片即刻更新，而是按本次配置下的结果统一提交。

---

## 六、开发排查时最值得先看的点

如果后面要排查问题，我建议优先按下面顺序看：

1. 任务有没有真正执行到 `execute(...)`，以及有没有被 `lockKey` 或服务器名单拦掉。
2. `queryWlaWaiting(...)` 有没有查到配置。
3. `wltAutoPlan(config)` 里配置是否因为 `model` 不存在或字段不一致被自动关闭。
4. `splicingWltCode(config)` 拼出来的 `queryCode` 是否符合预期。
5. 候选来料是不是在“已绑工单”或“已有 `LotPlan`”这两层过滤里被排空了。
6. `wltAutoPlanInfo(...)` 里是否成功创建了工单、`LotPlan` 和 `WorkOrderRelation`。
7. 最后 `updateWMSAutoPlanHis(...)` 是否执行，配置 `inputQty` 是否回写成功。

---

## 七、关联代码

| 文件                                                                                                   | 说明                      |
| ---------------------------------------------------------------------------------------------------- | ----------------------- |
| `Code-Projects/gc/core/src/gc/src/main/java/com/mycim/gc/service/quartz/jobs/WltAutoPlanJob.java`    | 定时任务入口，控制锁、环境、配置循环、收尾通知 |
| `Code-Projects/gc/core/src/gc/src/main/java/com/mycim/gc/service/impl/WorkOrderPlanServiceImpl.java` | 单条配置校验、查询条件拼接、候选过滤、正式排料 |

## 八、变更记录

| 日期 | 版本 | 变更内容 | 作者 |
|------|------|---------|------|
| 2026-06-12 | v2.0 | 重写为开发视角精简版，突出关键方法和关键判断 | Codex |
