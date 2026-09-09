## **问题描述：**

在处理 `Lot` 对象时，`InnerOrderNO` 被赋值的顺序和条件出现了问题。具体情况是：

- 当通过 `LotId` 获取 `WmsMmsMaterialLot` 对象时，`InnerOrderNO` 值为空。
- 当通过 `MaterialLot` 获取时，`InnerOrderNO` 不为空。
- 这可能是由于数据表中存在多条记录导致的，获取方式不同，赋值的 `InnerOrderNO` 也不同。

## **问题场景代码：**

```java
// 判断是否是SencorCP或LcdCP的产品分类
if (WorkOrder.CP_PRODUCT_CATEGORY_LIST.contains(productClassify) || StringUtils.isEqual(productCategory, WmsMmsMaterialLot.PRODUCT_CATEGORY_COB)) {
    // 执行cpSaveUnitAndGetMLotInfo方法，可能会影响lot的InnerOrderNO
    cpSaveUnitAndGetMLotInfo(lot, outerOrderNo);
}

// 获取MMS MaterialLot对象
WmsMmsMaterialLot materialLot = prpSetupService.getMmsMaterialLotById(lot.getLotId());
if (materialLot != null) {
    // 所有产线批次都加上小工单号
    lot.setInnerOrderNO(materialLot.getFtWorkorderId());
}

```

## **问题分析：**

### **1. 数据表中的多条记录：**

- **通过 `LotId` 获取数据时：** `prpSetupService.getMmsMaterialLotById(lot.getLotId())` 获取的 `materialLot` 可能返回为空，或者返回的是没有 `InnerOrderNO` 的数据。
- **通过 `MaterialLot` 获取数据时：** `cpSaveUnitAndGetMLotInfo()` 方法根据 `MaterialLot` 获取的数据可能已经包含了 `InnerOrderNO`，因此在这个情况下，`InnerOrderNO` 不为空。

### **2. 获取方式不同：**

- **通过 `LotId` 获取：** 获取到的数据可能是数据表中多个记录之一，且该记录的 `InnerOrderNO` 字段为空。
- **通过 `MaterialLot` 获取：** 该方法可能是基于其他条件查找的，返回的 `MaterialLot` 对象中 `InnerOrderNO` 已经设置了值。

### **3. 问题发生的原因：**

- **数据表设计问题：** `LotId` 和 `MaterialLot` 可能存在一对多的关系，`LotId` 关联的数据可能没有 `InnerOrderNO`，而 `MaterialLot` 关联的数据则有。
- **代码执行顺序问题：** 在执行 `cpSaveUnitAndGetMLotInfo()` 之前，通过 `LotId` 获取的数据为空或未能覆盖 `InnerOrderNO`，而之后的 `materialLot` 获取操作中，`InnerOrderNO` 被设置。

## **解决方案：**

### **解决方案1：检查 `LotId` 获取的 `materialLot` 是否为空**

在获取 `materialLot` 后，检查是否为空，如果为空，再根据 `MaterialLot` 获取并设置 `InnerOrderNO`。

```java
// 先通过LotId获取 materialLot
WmsMmsMaterialLot materialLot = prpSetupService.getMmsMaterialLotById(lot.getLotId());

// 如果通过LotId获取的 materialLot 为空，再通过MaterialLot获取
if (materialLot == null) {
    materialLot = prpSetupService.getMaterialLotBySomeOtherCriteria(lot);
}

if (materialLot != null) {
    if (StringUtils.isNotEmpty(materialLot.getFtWorkorderId()) && StringUtils.isEmpty(lot.getInnerOrderNO())) {
        lot.setInnerOrderNO(materialLot.getFtWorkorderId());
    }
}

```

### **解决方案2：确保通过 `LotId` 获取的数据包含 `InnerOrderNO`**

如果 `LotId` 的获取结果为空，可以考虑通过其他条件（如 `productCategory` 等）重新查找数据，确保获取的 `materialLot` 包含有效的 `InnerOrderNO`。

```java
// 如果通过LotId获取的MaterialLot没有InnerOrderNO，可以尝试根据其他条件查询
if (StringUtils.isEmpty(lot.getInnerOrderNO())) {
    WmsMmsMaterialLot materialLot = prpSetupService.getMmsMaterialLotByOtherCriteria(lot.getLotId());
    if (materialLot != null && StringUtils.isNotEmpty(materialLot.getFtWorkorderId())) {
        lot.setInnerOrderNO(materialLot.getFtWorkorderId());
    }
}

```

### **解决方案3：调整代码执行顺序**

确保 `cpSaveUnitAndGetMLotInfo()` 方法的调用是在 `InnerOrderNO` 需要赋值时才执行，并且在执行该方法之前，`materialLot` 已经获取并设置了 `InnerOrderNO`。

---

## **总结：**

- **问题本质：** `LotId` 和 `MaterialLot` 获取的 `InnerOrderNO` 不一致，导致了 `InnerOrderNO` 被覆盖或未按预期赋值。
- **解决方案：**
    - 提前确保获取的数据包含有效的 `InnerOrderNO`，避免通过 `LotId` 获取的数据为空或不完整。
    - 如果通过 `LotId` 获取为空，可以尝试通过其他条件重新查询。
    - 调整代码顺序，确保获取 `InnerOrderNO` 的方法在需要赋值时执行。