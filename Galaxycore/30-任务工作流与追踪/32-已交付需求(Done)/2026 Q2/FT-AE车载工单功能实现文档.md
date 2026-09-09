---
title: FT-AE车载工单功能实现文档
date: 2026-05-03
tags: [MES, 工作流]
---
# FT-AE车载工单功能实现文档

## 一、需求概述

新增FT-AE（车载）业务工单类型，工单属性基本与FT保持一致，仅产品分类不同：

| 业务类型 | 产品分类代码 |
|---------|-------------|
| FT | `1` |
| COM | `4` |
| **FT-AE** | **`32`** |

## 二、实现方案

将FT-AE的waferSource定义为`9FAE`（与FT的`8`、`10`类似格式），在现有逻辑中增加判断即可。

## 三、代码改动清单

### 1. WorkOrderServiceImpl.java

#### 改动位置1: `generateWorkOrder`方法

**文件路径**: `d:\data\MES\gc\core\src\gc\src\main\java\com\mycim\gc\service\impl\WorkOrderServiceImpl.java`

**改动内容**: 新增FT-AE判断，复用FT的工单生成逻辑

```java
} else if (StringUtils.equals(workOrder.getProductClassify(), WorkOrder.FT_AE)) {
    generateFtWorkOrder(workOrder, workOrderForm);
}
```

#### 改动位置2: `queryByLineType`方法

**改动内容**: FT和FT-AE使用相同的物料查询条件

```java
} else if (StringUtils.equals("FT", lineType) || StringUtils.equals(Lot.LINE_TYPE_FT_AE, lineType)) {
    matkl="33001";
}
```

#### 改动位置3: `checkOaNo`方法

**改动内容**: FT和FT-AE使用相同的OA单号前缀验证

```java
} else if (StringUtils.equals(lineType, "FT") || StringUtils.equals(lineType, Lot.LINE_TYPE_FT_AE)) {
    prefix = "ET.";
}
```

### 2. WorkOrderProduceAction.java

**文件路径**: `d:\data\MES\gc\core\src\model\src\com\mycim\webapp\actions\prp\WorkOrderProduceAction.java`

#### 改动位置1: `queryTargetGrade`方法

**改动内容**: 目标等级查询支持FT-AE

```java
if (StringUtils.isNotEmpty(productId)
        && StringUtils.isNotEmpty(routeId) && (StringUtils.equals(lineType, "FT") || StringUtils.equals(lineType, Lot.LINE_TYPE_FT_AE))) {
```

#### 改动位置2: `queryRouteId`方法

**改动内容**: 工艺流程查询支持FT-AE

```java
if (StringUtils.equals("FT", lineType) || StringUtils.equals(Lot.LINE_TYPE_FT_AE, lineType)) {
    //FT工艺流程查询
```

## 四、已有常量定义（无需修改）

| 常量                    | 值         | 所在类              |
| --------------------- | --------- | ---------------- |
| `WorkOrder.FT_AE`     | `"32"`    | `WorkOrder.java` |
| `Lot.LINE_TYPE_FT_AE` | `"FT-AE"` | `Lot.java`       |

## 五、数据库配置

### 5.1 $$NEW_WAFER_SOURCE参考文件

需新增记录：

| 字段 | 值 | 说明 |
| ------ | ----- | ------ |
| key1_value | `9FAE` | waferSource |
| data1_value | `FT-AE车载` | 显示名称 |
| data2_value | `32` | 产品分类代码(FT_AE) |
| data3_value | `FT-AE` | 线别类型 |
| data4_value | `FT-AE` | 页面分类 |

### 5.2 $$PORT_GATEGORY参考文件

需新增记录（用于工单号生成）：

| 字段 | 值 | 说明 |
|------|-----|------|
| key1_value | `32` | 产品分类代码 |
| data3_value | `AE` | 工单号前缀标识（根据实际规则配置） |

## 六、工单号生成规则

工单号格式：`ZJ` + 保税属性代码 + 产品分类代码 + 工单类型代码 + 日期 + 流水号

**FT-AE工单号示例**: `ZJBD`**`AE`**`A260422001`

## 七、功能验证要点

- [x] 创建FT-AE工单时，产品分类自动设为`32`
- [x] 物料查询使用FT相同的matkl条件(`33001`)
- [x] 工艺流程查询逻辑与FT一致
- [x] 目标等级查询逻辑与FT一致
- [x] OA单号验证使用前缀`ET.`（与FT相同）
- [x] 工单生成调用`generateFtWorkOrder`方法

## 八、相关文件清单

| 文件 | 路径 | 说明 |
|------|------|------|
| WorkOrder.java | `core/src/valueobject/src/com/mycim/prp/model/` | 工单实体类，定义FT_AE常量 |
| Lot.java | `core/src/valueobject/src/com/mycim/wip/model/` | 批次实体类，定义LINE_TYPE_FT_AE常量 |
| WorkOrderServiceImpl.java | `core/src/gc/src/main/java/com/mycim/gc/service/impl/` | 工单服务实现，核心业务逻辑 |
| WorkOrderProduceAction.java | `core/src/model/src/com/mycim/webapp/actions/prp/` | 工单生产Action，前端接口 |
| WmsMmsMaterialLot.java | `core/src/valueobject/src/com/mycim/prp/model/` | 物料批次实体，定义waferSource常量 |

## 九、注意事项

1. waferSource配置为`9FAE`，避免与FT的`8`、`10`冲突
2. FT-AE复用FT的所有工单生成逻辑，无需额外编写处理方法
3. 工单号前缀通过`$$PORT_GATEGORY`配置，确保与FT有区分
4. OA单号前缀规则与FT相同，均为`ET.`
