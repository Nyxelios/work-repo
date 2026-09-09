---
title: CP-WLT-自动排料工单模板
date: 2026-05-03
tags: [MES, 工作流]
---

### WLT 自动排料生单信息

```java
workOrder.setNamedSpace(baseService.getNamedSpace(1l, ObjectList.WORKORDER));  
workOrder.setCreateUser("AUTO_PLAN");  
workOrder.setWorkOrderType("1");  
workOrder.setProductClassify(product.getItemCategory());  
workOrder.setItem(product);  
workOrder.setProductId(config.getProductId());  
workOrder.setWaferLevel(mLot.getReserved50());  
workOrder.setComments("仓库领自动排料");  
workOrder.setPlanPutTime(DateUtils.formatDate(new Date(), "dd/MM/yyyy"));  
workOrder.setPlanTime("01/01/2120");  
workOrder.setBondedProperty(mLot.getReserved6());

workOrder.setPlannedQty(mLot.getCurrentQty().intValue());  
workOrder.setWorkorderId(getWorkorderId(ef.getWoType(), mLot.getReserved6(), "04"));  
workOrder.setLocation(MapUtils.getString(warehouseRrnNameMap, Long.parseLong(mLot.getReserved13())));  
workOrder.setOrderType(ef.getWoType());  
workOrder.setReserved59(ef.getWoType());  
workOrder.setReserved60(ef.getProcMethod());  
workOrder.setTestModelId(config.getTestModelId());  
workOrder.setStorageModel(config.getTestModelId());  
  
GcWorkorderDetail gcWorkorderDetail = new GcWorkorderDetail(model);  
gcWorkorderDetail.setObjectRrn(getObjectRRN());  
gcWorkorderDetail.setFacilityRrn(ThreadLocalContext.getFacilityRrn());  
gcWorkorderDetail.setWorkorderRrn(workOrder.getWorkorderRrn());  
gcWorkorderDetailRepository.save(gcWorkorderDetail);  
workOrder.setProcessType(processExt.getAttributeData1());  
//生产工单数据  
Map<String, Object> transInfo = new HashMap<String, Object>();  
transInfo.put("transId", WorkOrderHistory.TRANSTYPE_CREATED);  
transInfo.put("workOrder", workOrder);  
workOrder = prpSetupService.insertWorkOrder(transInfo);  
orderMap.put(key, workOrder);
```



