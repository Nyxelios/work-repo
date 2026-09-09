---
title: 工单晶圆同步
date: 2025-08-15
author: MES Team
status: completed
tags: [工单, 晶圆, 功能]
---


![](../../附件/Pasted%20image%2020250815142520.png)
# 查询

```java
ResultPage<WaferBindSync> waferBindSyncResultPage = waferBindSyncRepository.pageWaferBindSync(lotId, unitId, workOrderId, pageNumber, pageSize);

for (int i = 0; i < waferBindSyncResultPage.getContent().size(); i++) {  
    WaferBindSync w = waferBindSyncResultPage.getContent().get(i);  
    List<WmsMmsMaterialLotUnit> wmsMmsMaterialLotUnits = searchWms(w.getWorkOrderId(), null, w.getLotId());  
    BigDecimal qty = BigDecimal.valueOf(0);  
    for (WmsMmsMaterialLotUnit wmsMmsMaterialLotUnit : wmsMmsMaterialLotUnits) {  
        qty = qty.add(wmsMmsMaterialLotUnit.getCurrentQty());  
    }  
    w.setQty(qty);  
}
```

## 添加

[[工单投入#添加]]

## 导入

### 导入模板

| LOT_ID | WORK_ORDER_ID |
| ------ | ------------- |
|        |               |
导入查询[[工单投入#导入查询]]，查询后插入`WAFER_BIND_SYNC`

# 页面
## 文本
### 已选数

```jsx
function countSelectedLine() {  
    var records = Ext.getCmp('WaferBindSyncGrid').getSelectionModel().getSelection();  
    var total = 0;  
    Ext.each(records, function(item){  
        total = Number(total) + Number(item.data.qty);  
    });  
    Ext.getCmp('countNum').setValue(total);  
}
```

### 日期

```jsx
{  
    xtype: 'datefield',  
    name: 'startDate',  
    id: 'startDate',  
    fieldLabel: i18n_fld_start_date,  
    columnWidth: 0.25,  
    enableKeyEvents: true,  
    format: 'd-m-Y',  
}
```
![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241028105635.png)

# 绑定晶圆

## 手动

晶圆的绑定其实和已有方法差不多，我做了些修改
```jsx
@Override  
public void bindWorkOrderWaferBindSync(List<Map> waferBindSyncList) throws MyCimException {  
    try {  
        boolean waferFromWmsFlag = SystemPropertyUtils.getWaferFromMesOrWmsFlag();  
        //提取数据，按工单分类  
        Map<String, List<Map<String, Object>>> workOrderMaps = new HashMap<>();  
        for (Map workOrderMap : waferBindSyncList) {  
            String wo = StringUtils.isEmpty((String) workOrderMap.get("WORK_ORDER_ID")) ? (String) workOrderMap.get("workOrderId") : (String) workOrderMap.get("WORK_ORDER_ID");  
            List<Map<String, Object>> tems = workOrderMaps.get(wo);  
            if (tems == null) {  
                tems = new ArrayList<>();  
                tems.add(workOrderMap);  
            } else {  
                tems.add(workOrderMap);  
            }  
            workOrderMaps.put(wo, tems);  
        }  
        //按工单循 
        for (Map.Entry<String, List<Map<String, Object>>> entry : workOrderMaps.entrySet()) {  
  
            String workOrderId = entry.getKey();  
            WorkOrder workOrder = prpSetupService.getWorkOrderById(workOrderId);  
            List<Map<String, Object>> waferBindSyncMaps = entry.getValue();  
  
            for (Map<String, Object> waferBindSyncMap : waferBindSyncMaps) {  
                String lotId = StringUtils.isEmpty((String) waferBindSyncMap.get("LOT_ID")) ? (String) waferBindSyncMap.get("lotId") : (String) waferBindSyncMap.get("LOT_ID");  
                if (workOrder.getComponentMaterialId() != null  
                        || !StringUtils.equals(WorkOrder.COM, workOrder.getProductClassify())) {  
  
                    Long facilityRrn = ThreadLocalContext.getFacilityRrn() != null ? ThreadLocalContext.getFacilityRrn() : Long.parseLong(waferBindSyncMap.get("FACILITY_RRN").toString());  
                    String username = StringUtils.isNotEmpty(ThreadLocalContext.getUsername()) ? ThreadLocalContext.getUsername() : "MES";  
  
                    if (waferFromWmsFlag) {  
                        List<WmsMmsMaterialLotUnit> wmsMmsMaterialLotUnits = searchWms(workOrderId, lotId, facilityRrn);  
                        List<Map<String, Object>> wmsMmsMaterialLotUnitMaps = JSONUtils.toArrayListOfMap(wmsMmsMaterialLotUnits);  
                        if (judgeUnitIdIsAdded(workOrderId, wmsMmsMaterialLotUnits, facilityRrn)) {  
                            continue;  
                            //throw new MyCimException(ToolExceptions.THE_WAFER_ALREADY_BIND_ON_WORKORDER);  
                        }  
                        if (StringUtils.isNotEmpty(lotId) && CollectionUtils.isNotEmpty(wmsMmsMaterialLotUnits)) {  
                            toolService.updateWorkOrderRelationAndMaterialLotUnitAndSaveHis(wmsMmsMaterialLotUnitMaps, workOrderId);  
                            updateWaferSync(WaferBindSync.SYN_STATUS_SUCCESS, lotId, WaferBindSync.SYN_STATUS_SUCCESS_COMMENT);  
                        }  
                    } else {  
                        List<WaferReceive> wrlist = toolService.getWaferByBoxIdForBindWorkorder(lotId, null, workOrder);  
                        String levelTwoCode = workOrder.getLevelTwoCode();  
                        String secLevel = levelTwoCode.substring(0, levelTwoCode.length() - 1);  
                        if (CollectionUtils.isNotEmpty(wrlist)) {  
                            for (WaferReceive waferReceive : wrlist) {  
                                if (!StringUtils.equals(waferReceive.getVersion(), secLevel)) {  
                                    updateWaferSync(WaferBindSync.SYN_STATUS_FAIL, lotId, "二级代码与工单不);  
                                    throw new MyCimException(ToolExceptions.WAFER_VERSION_NOT_MATCH_WRAFER);  
                                } else {  
                                    toolService.updateToMes(facilityRrn, workOrder.getWorkorderRrn(), waferReceive.getWaferId(), username, workOrderId);  
                                }  
                            }  
                            updateWaferSync(WaferBindSync.SYN_STATUS_SUCCESS, lotId, WaferBindSync.SYN_STATUS_SUCCESS_COMMENT);  
                        }  
                    }  
                } else {  
                    updateWaferSync(WaferBindSync.SYN_STATUS_FAIL, lotId, "组件料号为空时不能添加晶);  
                    //throw new MyCimException(ToolExceptions.THE_COMPOENT_MATERIAL_ID_IS_EMPTY_CANT_ADD_WAFER);  
                }  
            }  
        }  
    } catch (Exception e) {  
        throw new MyCimException(e);  
    }  
}
```

## [[自动任务]]

```jsx
@Slf4j  
public class WaferBindSyncJob implements Job {  
    protected WaferBindSyncService waferBindSyncService = AppContext.getBean(WaferBindSyncService.class);  
    @Override  
    public void execute(JobExecutionContext context) throws JobExecutionException {  
        try {  
            //1.获取表内所有非SUCCESS状态的数据  
  
            List<Map> list = waferBindSyncService.getNoSuccessWaferBindSync();  
            waferBindSyncService.bindWorkOrderWaferBindSync(list);  
            log.info("WaferBindSyncJob executed successfully" + list);  
        } catch (Exception e) {  
            log.error(e.getMessage(), e);  
        }  
    }  
  
}
```