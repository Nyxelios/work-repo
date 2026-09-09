---
title: 工单-wipQty-异常增加
date: 2026-05-03
tags: [MES, BUG, 工作流]
---

感觉这个不是最重要的，为什么工单会无缘无故生成一条不存在的在线数据？

# COM 真空包重测跳站导致工单 wipQty 异常增加问题总结
## 一、问题现象
现象 说明 工单 wipQty （在线数量）莫名其妙增加 实际没有新批次生成，但工单在线数量被增加了 WORKORDER_H 表中插入了一条 STARTE 类型的历史记录 事务类型 = STARTE 历史记录的 comments 为空 没有 PUT_QTY 标记 LOT_PLAN 表中没有新记录 批次计划未创建 工单不是返工工单 按业务逻辑不应该走返工路径

## 二、根因分析
### 触发路径
```
前端 COM 真空包检验 → 跳站返工操作
  → ComLotMergeAction / LotMergeAction（真空包检验页面）
    → gcService.vboxDoRework(createLotInfo, userName, packedLotList)
      → PackageServiceImpl.vboxDoRework(...)
        → lotService.createLots(createLotInfo)  ← 批量创建批次
```
### 核心问题代码
PackageServiceImpl.java:L1672-L1673 ：

```
createLotInfo.put("doNotChangeWipQtyFlag", "1");  // 意图：不修改工单
在线数量
List<Map> resultsInfos = lotService.createLots(createLotInfo);  // 
实际：仍然修改了！
```
### 为什么 doNotChangeWipQtyFlag 不生效
createLots() 方法的流程：

```
阶段A：循环创建批次
  ├─ L1162: transInfo.put("unifiyProcessFlag", "1")   ← 这个有效
  ├─ L1163: createLot(transInfo)                       ← 被 
  unifiyProcessFlag 阻止，不修改工单 ✅
  └─ L1192-L1194: workOrder.setWipQty(...)             ← 无条件执行，
  累加 wipQty ❌

阶段B：统一持久化工单变更
  └─ L1203-L1214: merge(workOrder) + insertWorkOrderHistory
  (STARTE)  ← 无条件执行 ❌
```
问题本质 ：

- doNotChangeWipQtyFlag 只在 createLot() （单数）方法内部被检查 L1714
- createLots() （复数） 根本不检查这个标志
- 所以虽然设置了 doNotChangeWipQtyFlag=1 ，但 createLots() 里的工单更新逻辑 照样执行
### unifiyProcessFlag 的作用范围
```
transInfo.put("unifiyProcessFlag", "1")  ← L1162 设置
  ↓
阻止的是 createLot() 内部的工单更新 [L1714-L1734]
  ↓
阻止不了 createLots() 自己的工单更新 [L1166-L1214]
```
## 三、受影响的业务场景
场景 调用方法 是否受影响 COM 真空包检验 → 跳站返工 vboxDoRework() → createLots() ✅ 受影响 （当前问题） COM 批次合并 LotMergeAction → createLots() ✅ 可能受影响 普通工单发料上线 GcServiceImpl.erpWorkOrderSynAndCreateLots() → createLots() ✅ 受影响（但业务期望增加 wipQty）

## 四、修复方案
### 方案一：最小改动（推荐优先实施）
在 createLots() 的阶段B循环中检查 doNotChangeWipQtyFlag

修改文件： LotServiceImpl.java:L1202-L1215

```
@Transactional(rollbackFor = Exception.class)
public List<Map> createLots(Map transInfo) throws MyCimException {
    // ... 前面的代码不变 ...
    
    // ⬇️ 新增：获取标志
    String doNotChangeWipQtyFlag = (String) transInfo.get
    ("doNotChangeWipQtyFlag");
    
    String transtype = WorkOrderHistory.TRANSTYPE_STARTED;
    for (Map.Entry<String, WorkOrder> entry : workOrderMap.entrySet
    ()) {
        
        // ⬇️ 新增：跳过工单更新
        if ("1".equals(doNotChangeWipQtyFlag)) {
            continue;  // 不修改工单，不插入历史
        }
        
        WorkOrder workOrder = entry.getValue();
        workOrderRepository.merge(workOrder);
        
        Integer putQty = orderPut.get(workOrder.getWorkorderId());
        if(StringUtils.isEmpty(workOrder.getComments())){
            workOrder.setComments("PUT_QTY"+putQty.intValue());
        }else{
            workOrder.setComments(workOrder.getComments()+";PUT_QTY"
            +putQty.intValue());
        }
        prpSetupServices.insertWorkOrderHistory(transactionLog.
        getTransRrn(), transtype, workOrder);
    }
    // ... 后面的代码不变 ...
}
```
改动量 ：约 5 行代码

优点 ：

- 改动最小
- 不影响现有正常业务（发料上线仍然会更新 wipQty）
- 直接解决当前问题
### 方案二：同时修复 orderPut 累加 Bug（一并修复）
当前 Bug ： orderPut 在每次循环开始时重置为 0，导致最终只保留最后一个 lot 的数量

L1174-L1175 ：

```
BigDecimal put=new BigDecimal(0);  // ← 每次循环重置
...
orderPut.put(workOrderId, put.intValue());  // ← 只保留最后一个 lot 的
值
```
修改为 ：

```
BigDecimal put = new BigDecimal(orderPut.getOrDefault(workOrderId, 
0));  // ← 从 Map 取旧值
...
orderPut.put(workOrderId, put.intValue());
```
或者 （更好的方式）：

```
orderPut.merge(workOrderId, lotQty.intValue(), Integer::sum);  // 直
接累加
```
### 方案三：统一事务回滚规则（建议一并检查）
createLots() 方法现在已经有：

```
@Transactional(rollbackFor = Exception.class)  // ← 你已经加了
```
但建议确认 createLot() 方法也加上，防止单独调用时出问题：

```
@Transactional(rollbackFor = Exception.class)
public Map createLot(Map transInfo) throws MyCimException
```
## 五、修复后效果
场景 修复前 修复后 COM 真空包跳站返工 wipQty 异常增加 + 插入 STARTE 历史 wipQty 不变 + 不插入历史 ✅ 普通工单发料上线 wipQty 正常增加 + 插入 STARTE 历史 wipQty 正常增加 + 插入 STARTE 历史 ✅ COM 批次合并 可能受影响 取决于是否设置 doNotChangeWipQtyFlag

## 六、其他关联问题（建议一并排查）
问题 位置 说明 createReworkLot() 也会修改 wipQty + 减少 scrapQty LotServiceImpl.java:L1239-L1240 如果工单不是返工场景，scrapQty 会被错误减少 LotPlan 删除用错 RRN LotServiceImpl.java:L1687 deleteOneById(tempLotRrn) 可能删不掉正确的 LotPlan transInfo 共享 Map 污染 createLots() 循环中反复使用同一个 Map 可能影响下一次循环 并发 wipQty 丢失更新 createLots() 先读后写 多个请求同时操作同一工单时可能丢失更新

## 七、建议实施顺序
优先级 操作 改动量 风险 🔴 最高 方案一： createLots() 检查 doNotChangeWipQtyFlag 5 行 低 🟡 中 方案二：修复 orderPut 累加 Bug 2 行 低 🟢 低 方案三：确认事务回滚规则 1 行 极低