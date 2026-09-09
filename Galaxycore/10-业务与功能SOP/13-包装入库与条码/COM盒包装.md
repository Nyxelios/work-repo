---
title: 盒包装功能文档
date: 2026-05-20
author: MES Team
status: completed
tags: [包装, 制造管理, COM, FT]
---

# 盒包装功能文档

## 1. 功能概述

盒包装（Box Package）是 MES 系统中用于将晶圆批次（Lot）按标准数量打包成盒（TBox）的核心功能。系统支持 COM 和 FT 两种产品类型的包装，提供整包、散包、尾数包三种包装模式，并在包装完成后自动打印盒标签和零头批次标签。

### 1.1 业务范围

- **COM 产品**：按盒标准数量整包或尾数包装
- **FT 产品**：按盒标准数量整包或尾数包装
- **散包**：将多个批次合并包装到一个盒中，不受标准数量严格限制

### 1.2 核心术语

| 术语 | 说明 |
|------|------|
| TBox | 小盒（实体包装单位） |
| VBox | 真空包（大包装单位，由多个 TBox 组成） |
| 盒标准数量 | 每个 TBox 应装的晶圆数量，由产品定义配置 |
| 尾数包 | 当剩余数量不足一盒时的包装方式，盒号流水码从 7001 开始 |
| 零头标签 | 包装完成后，对仍有剩余未包装数量的批次打印的提示标签 |
| LotDieBin | 批次分档信息，记录每个批次各等级的未包装数量 |

---

## 2. 业务流程

### 2.1 整体流程图

```
┌─────────────────┐
│   查询待包装批次   │
│  (产品/工单/等级) │
└────────┬────────┘
         ▼
┌─────────────────┐
│   扫描/选择批次   │
│  (加入包装列表)   │
└────────┬────────┘
         ▼
┌─────────────────┐
│   校验包装规则    │
│ (数量/等级/工单)  │
└────────┬────────┘
         ▼
┌─────────────────┐     ┌─────────────────┐
│    整包/尾数包   │────▶│     散包包装     │
│   (doPackage)   │     │ (doLoosePackage) │
└────────┬────────┘     └─────────────────┘
         ▼
┌─────────────────┐
│   生成盒号并保存  │
│  (PackedLot/Detail)│
└────────┬────────┘
         ▼
┌─────────────────┐
│    打印盒标签    │
│   (TBOX.btw)    │
└────────┬────────┘
         ▼
┌─────────────────┐
│   打印零头标签   │
│ (packageLotInfo.btw)│
└─────────────────┘
```

### 2.2 包装模式对比

| 模式 | 触发条件 | 盒标准数量校验 | 盒号规则 | 适用场景 |
|------|----------|--------------|----------|----------|
| 整包 | 点击【包装】按钮 | 必须等于盒标准数量 | 正常流水码 | 数量充足，整盒包装 |
| 尾数包 | 点击【尾数包装】按钮 | 可小于盒标准数量 | 流水码从 7001 开始 | 最后剩余不足一盒 |
| 散包 | 点击【散包装】按钮 | 不严格卡控 | 正常流水码 | 多个小批次合并 |

---

## 3. 功能界面

### 3.1 查询条件区

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| 产品型号 | 下拉框 | 是 | 选择要包装的产品 |
| 等级 | 下拉框 | 是 | 选择 Bin 等级（如 KA、JA、B、C 等） |
| 工单号 | 搜索框 | 是 | 输入工单号 |
| 二级代码 | 文本框 | 否 | 过滤条件 |
| 保税属性 | 文本框 | 否 | 过滤条件 |
| 扫描批次 | 文本框 | 否 | 扫描批次号快速加入包装列表 |

### 3.2 批次列表区

显示符合条件的待包装批次，包含以下字段：

- 工单号、产品型号、二级代码、Wafer ID
- 批次号、等级、未包装数量、最后操作时间、保税属性

**交互**：
- 单击行：自动将该批次加入右侧包装列表（特定等级自动触发）
- 复选框：批量选择批次

### 3.3 包装操作区

| 字段 | 说明 |
|------|------|
| 盒总数量 | 当前查询条件下所有批次的未包装数量合计 |
| 已选数量 | 已加入包装列表的批次数量合计 |
| 盒标准数量 | 当前产品的标准包装数量（可修改） |
| 销售备注 | 包装时记录的销售相关信息 |

**操作按钮**：

| 按钮 | 功能 |
|------|------|
| 包装 | 整包模式，盒标准数量必须等于配置值 |
| 散包装 | 散包模式，多个批次合并到一个盒 |
| 尾数包装 | 尾数模式，盒标准数量可小于配置值 |
| 打印剩余数量 | 打印当前查询条件下的总剩余数量标签 |

---

## 4. 核心业务规则

### 4.1 包装前校验规则

#### 4.1.1 基础校验

- 盒标准数量不能为空或 0
- 已选批次数量必须大于盒标准数量
- 所有批次必须是同一工单、同一等级、同一二级代码、同一保税属性

#### 4.1.2 数量一致性校验

```
if (!mantissaFlag) {  // 整包模式
    if (Golden等级) {
        盒标准数量必须等于 Golden 标准数量(100)
    } else {
        盒标准数量必须等于产品配置的盒标准数量
    }
} else {  // 尾数包模式
    if (Golden等级) {
        盒标准数量不能大于 Golden 标准数量
    } else {
        盒标准数量不能大于产品配置的盒标准数量
    }
}
```

#### 4.1.3 批次数量变更校验

包装前会再次从数据库查询每个批次的 `LotDieBin` 未包装数量，如果与前端传入的数量不一致，则抛出异常：

```
LOT_UNPACKED_QTY_HAS_CHANGED
```

防止在查询和包装之间批次数量被其他操作修改。

### 4.2 等级分类与卡控规则

#### 4.2.1 等级分组定义

系统中等级分为以下三组，每组有不同的卡控逻辑：

| 等级分组 | 包含等级 | 代码位置 |
|----------|----------|----------|
| **Golden 等级** | KA, JA, LA, IA, GA, CA | `exceptGoldenGrade()` |
| **F 等级** | F, F3 | `exceptFGrade()` |
| **自动扫描等级** | B, C, F, F2, B2, SA0, SA1, MA0, MA1, MA2, SA2, HA0, HA1, C1, C2, C3, TA1 | `autoScanLot()` |

#### 4.2.2 自动扫描规则

点击批次列表行时，仅当等级属于自动扫描等级列表时，才会自动将批次加入包装列表（store3）：

| 等级 | 是否自动扫描 | 说明 |
|------|:----------:|------|
| B | ✅ | - |
| C | ✅ | - |
| F | ✅ | 同时属于 F 等级，散包有特殊卡控 |
| F2 | ✅ | - |
| F3 | ❌ | 属于 F 等级，但不在自动扫描列表中 |
| B2 | ✅ | - |
| SA0 | ✅ | - |
| SA1 | ✅ | - |
| SA2 | ✅ | - |
| MA0 | ✅ | - |
| MA1 | ✅ | - |
| MA2 | ✅ | - |
| HA0 | ✅ | - |
| HA1 | ✅ | - |
| C1 | ✅ | - |
| C2 | ✅ | - |
| C3 | ✅ | - |
| TA1 | ✅ | - |
| KA | ❌ | Golden 等级，需手动扫描 |
| JA | ❌ | Golden 等级，需手动扫描 |
| LA | ❌ | Golden 等级，需手动扫描 |
| IA | ❌ | Golden 等级，需手动扫描 |
| GA | ❌ | Golden 等级，需手动扫描 |
| CA | ❌ | Golden 等级，需手动扫描 |

> **不在自动扫描列表中的等级**，需要通过批次号输入框回车扫描或点击查询按钮来加入包装列表。

#### 4.2.3 各等级在不同包装模式下的卡控

| 等级分组 | 整包卡控 | 尾数包卡控 | 散包卡控 | 二级代码 |
|----------|----------|----------|----------|----------|
| **Golden** (KA/JA/LA/IA/GA/CA) | 盒标准数量 **必须等于** Golden 标准数量（默认100） | 盒标准数量 **不能大于** Golden 标准数量 | 盒标准数量 **不能大于** Golden 标准数量 | 正常生成 |
| **F 等级** (F/F3) | 盒标准数量必须等于产品配置的盒标准数量 | 盒标准数量不能大于产品配置的盒标准数量 | **不卡控**盒标准数量上限 | 正常生成 |
| **其他等级** (B/C/B2/SA0/SA1/MA0/MA1/MA2/SA2/HA0/HA1/C1/C2/C3/TA1/F2) | 盒标准数量必须等于产品配置的盒标准数量 | 盒标准数量不能大于产品配置的盒标准数量 | 盒标准数量不能大于产品配置的盒标准数量 | 正常生成 |
| **HA/MA** (HA0/HA1/MA0/MA1/MA2) | 同其他等级 | 同其他等级 | 同其他等级 | **不生成**二级代码 |

#### 4.2.4 卡控逻辑伪代码

**整包（mantissaFlag = false）**：

```
if (Golden等级) {
    盒标准数量 == Golden标准数量   → 通过，否则提示"盒标准数量与golden标准数不一致"
} else {
    盒标准数量 == 产品盒标准数量    → 通过，否则提示"包装数量与盒标准数量不一致"
}
```

**尾数包（mantissaFlag = true）**：

```
if (Golden等级) {
    盒标准数量 <= Golden标准数量   → 通过，否则提示"盒标准数量不能大于golden标准数"
} else {
    盒标准数量 <= 产品盒标准数量    → 通过，否则提示"盒标准数量不能大于标准数量"
}
```

**散包**：

```
if (Golden等级) {
    盒标准数量 <= Golden标准数量   → 通过，否则提示"Golden等级包装数量不能大于标准数量"
} else if (F等级) {
    不卡控盒标准数量上限            → 直接通过
} else {
    盒标准数量 <= 产品盒标准数量    → 通过，否则提示"盒标准数量不能大于标准数量"
}
```

### 4.3 盒号生成规则

#### 4.3.1 COM 产品盒号

```
格式: [产品类别码]T[保税属性][yyMMdd][流水码]

示例: CTSHB2308150001
- C: 产品类别码（从配置表获取）
- T: TBox 标识
- SHB: 保税属性
- 230815: 日期
- 0001: 流水码
```

#### 4.3.2 尾数包盒号

尾数包的流水码从 **7001** 开始，与其他包装分开编号：

```
格式: [产品类别码]T[保税属性][yyMMdd]7[流水码]

示例: CTSHB2308157001
```

#### 4.3.3 FT 产品盒号

```
格式: T[保税属性][yyMMdd][流水码]

流水码从 30001 开始
```

---

## 5. 包装算法详解

### 5.1 整包/尾数包算法

核心逻辑在 `boxPackageLotAndGetBoxInfo` 方法中：

#### 5.1.1 单批次包装流程

1. **获取批次信息**：`qty1`（批次总数量）、`unpackedQty`（当前未包装数量）
2. **检查当前盒状态**：
   - 如果当前有未装满的盒，先往里面装
   - 如果当前盒剩余容量 >= 批次未包装数量：全部装入当前盒
   - 如果当前盒剩余容量 < 批次未包装数量：装满当前盒，剩余数量继续处理
3. **生成新盒**：
   - 当批次还有剩余时，按盒标准数量循环生成新盒
   - 最后不足一盒的剩余数量：
     - 如果总剩余数量 < 盒标准数量：作为零头保留，不生成盒
     - 否则：生成一个尾数盒

#### 5.1.2 关键代码逻辑

```java
// 当前盒有剩余空间且批次有未包装数量
if (packedLot.getPackedLotRrn() != null && (boxStandardQty - quantity) > 0 && unpackedQty > 0) {
    Integer boxUnpackageQty = boxStandardQty - quantity; // 盒中剩余容量
    
    if (boxUnpackageQty >= unpackedQty) {
        // 批次全部装入当前盒
        packedLot.setQuantity(packedLot.getQuantity() + unpackedQty);
        lot.setUnpackedQty(0); // 批次已包完
    } else {
        // 装满当前盒，批次还有剩余
        packedLot.setQuantity(boxStandardQty);
        lot.setUnpackedQty(unpackedQty - boxUnpackageQty);
        
        // 继续按标准数量生成新盒
        while (lot.getUnpackedQty() >= boxStandardQty) {
            // 生成新盒...
            lot.setUnpackedQty(lot.getUnpackedQty() - boxStandardQty);
        }
        
        // 最后剩余不足一盒
        if (lot.getUnpackedQty() > 0 && lotQty >= boxStandardQty) {
            // 生成尾数盒
        }
    }
}
```

### 5.2 散包算法

散包逻辑在 `boxLoosePackageLotAndGetBoxInfo` 方法中：

1. 创建一个盒，标准数量为用户输入的值
2. 遍历批次列表，依次往盒里装：
   - 如果批次未包装数量 <= 盒剩余容量：全部装入
   - 如果批次未包装数量 > 盒剩余容量：只装入剩余容量部分，停止
3. 更新批次和 `LotDieBin` 的未包装数量

### 5.3 数据更新

包装完成后，系统会更新以下数据：

| 数据表 | 更新内容 |
|--------|----------|
| `PackedLot` | 插入盒主记录（盒号、数量、状态等） |
| `PackedLotDetail` | 插入盒与批次的关联记录（每批次一条） |
| `LotDieBin` | 更新未包装数量（`unpacked_qty`） |
| `Lot` | 更新批次数量，如果全部包完则更新状态为 `COM` |
| `LotTransHistory` | 记录批次操作历史 |
| `TransactionLog` | 记录事务日志 |

---

## 6. 标签打印

### 6.1 盒标签（TBOX.btw）

包装完成后，为每个生成的盒打印标签，包含以下信息：

| 字段 | 说明 |
|------|------|
| DEVICEID | 产品型号 |
| NUMBER | 盒内实际数量 |
| SUBCODE | 子代码 |
| BOXID | 盒号 |
| PACKED1~PACKED5 | 盒内批次号（COM 最多显示 5 个，FT 最多显示 4 个） |
| PACKEDQTY1~PACKEDQTY5 | 对应批次的包装数量 |
| QRCODEINFO | 二维码信息（盒号\|产品\|日期\|批次\|类别\|数量） |

**超过显示上限的处理**：
- COM：前 4 个批次单独显示，第 5 个显示为 "OTHERS"（数量为剩余批次合计）
- FT：前 3 个批次单独显示，第 4 个显示为 "OTHERS"

COM二级代码标签不再显示等级

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241118173622.png)


### 6.2 零头批次标签（packageLotInfo.btw）

包装完成后，系统根据后端返回的 `remainingLots` 数据决定是否打印零头标签：

| 字段 | 说明 |
|------|------|
| GRADE | 等级 |
| LOTID | 批次号 |
| NUM | 剩余未包装数量 |
| PRODUCTID | 产品型号 |
| WORKORDERID | 工单号 |

#### 6.2.1 核心规则

- **前端不计算**：前端不做任何零头数量计算，完全依赖后端返回的数据
- **有数据就打印**：后端返回 `remainingLots` 数组有元素，就打印零头标签
- **没数据就不打印**：后端返回空数组或 `undefined`，则不打印
- **所有包装操作统一**：整包、尾数包、散包三个操作的后端接口都返回 `remainingLots`

#### 6.2.2 后端计算逻辑

包装完成后，后端遍历所有参与包装的批次，通过 `LotDieBin` 查询数据库最新的未包装数量：

```java
JSONArray remainingLotJson = new JSONArray();
for (Lot lot : lotsInfo) {
    LotDieBin lotDieBin = lotService.getLotDieBinByRrn(lot.getLotDieBinRrn());
    if (lotDieBin.getUnpackedQty() != null && lotDieBin.getUnpackedQty() > 0) {
        JSONObject remaining = new JSONObject();
        remaining.put("lotId", lot.getLotId());
        remaining.put("unpackedQty", lotDieBin.getUnpackedQty().intValue());
        remaining.put("grade", lot.getWaferLevel());
        remaining.put("productId", productId);
        remaining.put("workOrderId", workOrderId);
        remainingLotJson.add(remaining);
    }
}
map.put("remainingLots", remainingLotJson);
```

#### 6.2.3 前端打印逻辑

```javascript
function printRemainingLotLabel(remainingLots) {
    if (!remainingLots || remainingLots.length == 0) {
        return;  // 没有零头，直接返回不打印
    }

    var remainRecord = remainingLots[0];
    var printInfo = "GRADE=" + remainRecord.grade
        + ";LOTID=" + remainRecord.lotId
        + ";NUM=" + remainRecord.unpackedQty
        + ";PRODUCTID=" + remainRecord.productId
        + ";WORKORDERID=" + remainRecord.workOrderId;
    bartenderPrint(printInfo, "packageLotInfo.btw", 1);
}
```

---

## 7. 接口说明

### 7.1 前端请求接口

| Action | URL | 说明 |
|--------|-----|------|
| query | `/mycim2/packageLot.do?action=query` | 查询待包装批次 |
| packageLot | `/mycim2/packageLot.do?action=packageLot` | 整包/尾数包装 |
| MantissaPackage | `/mycim2/packageLot.do?action=MantissaPackage` | 散包装 |
| queryProduct | `/mycim2/packageLot.do?action=queryProduct` | 查询产品列表 |
| queryBinGroup | `/mycim2/packageLot.do?action=queryBinGroup` | 查询等级列表 |

### 7.2 包装请求参数

**`packageLot` 请求参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| salesNote | String | 销售备注 |
| productId | String | 产品型号 |
| workOrderId | String | 工单号 |
| grade | String | 等级 |
| lotQty | Integer | 已选批次总数量 |
| boxStandardQty | Integer | 盒标准数量 |
| secondCode | String | 二级代码 |
| bondedProperty | String | 保税属性 |
| lotInfoList | JSON | 批次信息列表 |
| mantissaFlag | Boolean | 是否尾数包 |

### 7.3 包装响应数据

```json
{
  "success": true,
  "msg": {
    "data": [
      {
        "boxId": "CTSHB2308150001",
        "productId": "GC5035",
        "workorderId": "WO20230815001",
        "grade": "KA",
        "count": 100,
        "NUMBER": 100,
        "PACKED1": "LOT001",
        "PACKEDQTY1": 60,
        "PACKED2": "LOT002",
        "PACKEDQTY2": 40,
        "QRCodeInfo": "CTSHB2308150001|GC5035|20230815|001|C|100"
      }
    ],
    "productClassify": "COM",
    "remainingLots": [
      {
        "lotId": "LOT003",
        "unpackedQty": 25,
        "grade": "KA",
        "productId": "GC5035",
        "workOrderId": "WO20230815001"
      }
    ]
  }
}
```

---

## 8. 相关代码文件

### 8.1 前端文件

| 文件 | 说明 |
|------|------|
| [packageLot.jsp](file:///d:/data/document/Github/work-archive/Code-Projects/gc/core/web/wip/packageLot/packageLot.jsp) | 页面入口 |
| [packageLotForm.js](file:///d:/data/document/Github/work-archive/Code-Projects/gc/core/web/wip/packageLot/packageLotForm.js) | 查询表单、扫描逻辑 |
| [packageLotGrid.js](file:///d:/data/document/Github/work-archive/Code-Projects/gc/core/web/wip/packageLot/packageLotGrid.js) | 批次列表、包装操作、标签打印 |
| [boxPackageGrid.js](file:///d:/data/document/Github/work-archive/Code-Projects/gc/core/web/wip/packageLot/boxPackageGrid.js) | 已包装盒列表展示 |
| [packageLotViewer.js](file:///d:/data/document/Github/work-archive/Code-Projects/gc/core/web/wip/packageLot/packageLotViewer.js) | 页面布局容器 |

### 8.2 后端文件

| 文件 | 说明 |
|------|------|
| [PackageLotAction.java](file:///d:/data/document/Github/work-archive/Code-Projects/gc/core/src/model/src/com/mycim/webapp/actions/wip/PackageLotAction.java) | Action 层，处理前端请求 |
| [PackageService.java](file:///d:/data/document/Github/work-archive/Code-Projects/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/PackageService.java) | Service 接口 |
| [PackageServiceImpl.java](file:///d:/data/document/Github/work-archive/Code-Projects/gc/core/src/ejb/wip/lot/src/com/mycim/wip/lot/service/impl/PackageServiceImpl.java) | Service 实现，核心包装逻辑 |

---

## 9. 常见问题

### Q1: 为什么提示"批次未包装数量已变更"？

在查询批次和点击包装之间，如果有其他用户或操作修改了批次的未包装数量，系统会检测到不一致并阻止包装，防止数据错误。

**解决方法**：重新查询批次后再进行包装。

### Q2: 尾数包和整包有什么区别？

- **整包**：盒标准数量必须等于产品配置的标准数量，用于正常包装
- **尾数包**：盒标准数量可以小于标准数量，用于最后剩余不足一盒的情况

### Q3: 零头标签为什么不打印了？

零头标签的打印完全由后端驱动：

- **后端返回 `remainingLots` 有数据** → 前端自动打印零头标签
- **后端返回空数组** → 表示所有批次已包完，没有零头，不打印

如果预期有零头但没有打印，请检查：

1. 后端是否已部署最新代码（`boxLoosePackage` 和 `doBoxPackageLot` 都需要返回 `remainingLots`）
2. 数据库中 `LotDieBin.unpacked_qty` 是否确实大于 0

### Q4: 零头标签数量为什么不准确？

零头数量由后端在包装完成后，**直接从数据库查询 `LotDieBin` 最新的未包装数量**，不是用前端传入的旧数据计算。如果发现问题，请检查：

1. 是否有其他操作同时修改了批次数量
2. 包装事务是否正常提交

### Q5: F 等级有什么特殊处理？

- F / F3 等级在散包时不卡控盒标准数量
- F 等级 wafer 在包装时会提示 "F等级wafer"

### Q6: 为什么点击批次列表行没有自动加入包装列表？

只有特定等级才会触发自动扫描加入包装列表，详见 [4.2.2 自动扫描规则](#422-自动扫描规则)。Golden 等级（KA/JA/LA/IA/GA/CA）和 F3 等级不在自动扫描列表中，需要通过批次号输入框回车扫描或点击查询按钮来加入。

---

## 10. 变更历史

| 日期         | 变更内容                                 | 关联工作流                                  | 负责人      |
| ---------- | ------------------------------------ | --------------------------------------- | -------- |
| 2024-10-21 | 初始文档                                 | -                                       | MES Team |
| 2026-05-20 | 重构文档结构，补充业务流程和接口说明                   | -                                       | MES Team |
| 2026-05-20 | 零头标签数量改为后端实时计算返回                     | [盒包装零头标签后端驱动改造](../../../2.工作流/2026%20Q2/Done/盒包装零头标签后端驱动改造.md) | MES Team |
| 2026-05-20 | 所有包装操作统一返回 `remainingLots`；前端不再做零头计算 | [盒包装零头标签后端驱动改造](../../../2.工作流/2026%20Q2/Done/盒包装零头标签后端驱动改造.md) | MES Team |
| 2026-05-20 | 补充等级分类与卡控规则详细说明                      | -                                       | MES Team |

### 关联文档

- **工作流任务**：[盒包装优化查询以及出零头标签](../../../2.工作流/2026%20Q2/TODO/TBC/盒包装优化查询以及出零头标签.md)
- **变更日志**：[盒包装零头标签后端驱动改造](../../../2.工作流/2026%20Q2/Done/盒包装零头标签后端驱动改造.md)
