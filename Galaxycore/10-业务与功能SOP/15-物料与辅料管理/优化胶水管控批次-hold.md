---
title: 优化胶水管控批次HOLD
date: 2024-10-21
author: MES Team
status: completed
tags: [胶水管理, 功能]
---

## 0. 可行性分

原逻辑存在问题：以lot在CDES开始作业时的顺序进行hold

现需要修改为：在cdes开始作业时，判断该批次是否是使用第二个的胶水的批次，是的话就hold

## 1. 逻辑实现

```jsx
//获取该胶水的第二个使用的批次
List < LotConsumesMaterialHistory > list = lotServiceInterface.getLotConsumeHisByLotNumber(glue.getName());
if (list.size() >= 2 && StringUtils.equals(list.get(1).getLotIds(), lot.getLotId())) {
    List < GlueThrust > glueThrusts = glueService.getGlueThrustList(lot.getLotId(), null);
    if (glueThrusts.isEmpty()) {
        TransReason transReason = new TransReason();
        transReason.setReasonCode("GLUEHOLD");
        transReason.setReason("SYSTEM:" + TransReason.GLUE_HOLD);
        transReason.setTransQty1(lot.getQty1());
        transReason.setTransQty2(lot.getQty2());
        transReason.setResponsibility(ThreadLocalContext.getUsername());

        HashMap holdInfo = new HashMap();
        holdInfo.put("lotRrn", new Long(lot.getLotRrn()).toString());
        holdInfo.put("lotId", lot.getLotId());
        holdInfo.put("lotStatus", lot.getLotStatus());
        holdInfo.put("transPerformedBy", ThreadLocalContext.getUsername());
        holdInfo.put("holdBy", ThreadLocalContext.getUserRrn().toString());
        holdInfo.put("transComments", "");
        holdInfo.put("operation", operation.getInstanceId());
        holdInfo.put("holdcode", "");
        holdInfo.put("transReason", transReason);
        lotServiceInterface.holdLot(holdInfo);
        f = false;

        GlueThrust glueThrust = new GlueThrust();
        glueThrust.setGlueNumber(glue.getName());
        glueThrust.setLotNumber(lot.getLotId());
        glueThrust.setThrustTime(new Date());
        glueThrust.setThrustValue(0.0);
        //glueThrust.setEquipNumber(glueService.getGlueEquipmentId(glue.getName(), lot.getLotId()));
        glueThrust.setOwner(ThreadLocalContext.getUsername());
        glueService.saveGlueThrust(glueThrust);
    }
}
```
