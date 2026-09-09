---
title: 工单绑定晶圆代码逻辑
date: 2023-01-16
tags: [MES, 工单, 晶圆]
---
# 工单绑定晶圆代码逻辑

了解工单绑定晶圆代码逻辑，产品定义里的晶圆型号和重测型号只要添加一项，工单类型是什么都可以绑定晶圆

---

如果wafer来自wms，产品型号没有绑定晶圆型号的话，会报"没有晶圆"；

![[工单绑定晶圆代码逻辑-Untitled.png]]

查询的晶圆型号是返工的还是正常的，在这里；

如果wafer来自mes，则没有数据和没有晶圆型号，都不会报错，返回空数据；

![[工单绑定晶圆代码逻辑-Untitled 1.png]]

分为晶圆是否来自wms，如果是来自wms，判断工单的类型，根据产品型号id查；

如果是来自mes，就判断等级，晶圆属性，产品型号id查

---

```java
//如果是RW工单的Recon和工程试验类型，就查询该产品型号绑定的重测型号，其余查询晶圆信息
if(StringUtils.isEqual(WorkOrder.RW, workOrder.getProductClassify())){
						if (StringUtils.isEqual(WorkOrder.Recon, workOrder.getWorkOrderType())
						|| StringUtils.isEqual(WorkOrder.ENGINEERING_EXPERIMENT, workOrder.getWorkOrderType())){
							materialInfo = getProductAndReworkMaterialRelationForShow(instanceRrn);
						}else {
							materialInfo = getProductAndMaterialRelationForShow(instanceRrn);
						}
					}else{
						materialInfo = getProductAndMaterialRelationForShow(instanceRrn);
					}
```
