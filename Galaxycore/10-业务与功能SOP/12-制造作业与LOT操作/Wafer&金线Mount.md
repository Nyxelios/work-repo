---
title: Wafer与金线Mount
date: 2024-12-23
author: MES Team
status: completed
tags: [制造管理, LOT, 功能]
---

# Wafer 与金线 Mount

记录 `equipmentmaterial.jsp` 和 `EquipmentMaterialAction.java` 中与金线结束标识相关的处理逻辑。

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241223142251.png)

## 前端页面：`equipmentmaterial.jsp`

```jsp
<td class="myfield" width="10%" nowrap>
    金线结束标识
</td>
<td class="myfield" width="10%" nowrap>
    <input type="checkbox" name=wireFinFlag id="wireFinFlag">
</td>

$("[name=unmount]").click(function(){
    var objectRrn = $(this).parent().prev().prev().prev().prev().prev().prev().val();
    var materialName = $(this).parent().prev().prev().prev().prev().prev().text();
    var materialLotId = $(this).parent().prev().prev().prev().prev().text();
    var surplusQty = $(this).parent().prev().children().val();
    var params = "?modifyFlag=unmount&objectRrn=" + objectRrn
        + "&materialName=" + materialName
        + "&materialLotId=" + materialLotId
        + "&surplusQty=" + surplusQty;
    if ($('#wireFinFlag').prop("checked")) {
       params = params + "&wireFinFlag=checked";
    }
    $("form").attr('action', doSearchUrl + params);
    $("form").submit();
});
```

## 后端处理：`EquipmentMaterialAction.java`

```java
String wireFinFlag = request.getParameter("wireFinFlag");
if (StringUtils.equals(wireFinFlag, "checked")
       && theform.getEquipmentId().startsWith("CWDB")
       && StringUtils.equals(material.getItemClass(), Item.WIRE)) {
    LotInventoryDO lotInventoryDO = lotInventoryManager.getLotInventory(lotInventory.getLotNumber());
    lotInventoryDO.setStatus(LotInventoryStatus.CLOSE);
    lotInventoryManager.updateLotInventoryForStatus(lotInventoryDO);
}
```


金线解绑增加个结束按钮，功能同解绑，但是结束后金线的状态不一样，区分解绑的状

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241118174156.png)

