---
title: 处理重复键值问题
date: 2026-05-03
tags: [AI协作, MES]
---
# 处理重复键值问题

> **Workspace:** d:\data\MES\gc

---

## User
现在遇到了一种情况，开工单的时候我是根据这个类表，传入key（wafersource）来获取各项属性的，之前不会有相同的第三列（产品分类），key相同，但是现在ft有了个新业务，wafersource相同，但是产品分类的数字是1，但是名称改成了FT-AE，不是FT了，导致@Override 
     public ReferenceFileDetail getReferenceFileValue(String className, String FileKey1) { 
         try { 
             ReferenceFile referenceFile = cimSetupService.getReferenceFile(new ReferenceFile(className, 
                     baseService.getNamedSpace(ThreadLocalContext.getFacilityRrn(), ObjectList.REFERENCE_FILE_KEY), ObjectList.REFERENCE_FILE_KEY)); 
             if (referenceFile == null) { 
                 throw new MyCimException(className + " == null"); 
             } 
             if (referenceFile.getValuesItems().isEmpty()) { 
                 throw new MyCimException(className + " == null"); 
             } 
             for (ReferenceFileDetail referenceFileDetail : referenceFile.getValuesItems()) { 
                 if (StringUtils.equals(referenceFileDetail.getKey1Value(), FileKey1)) { 
                     return referenceFileDetail; 
                 } 
             } 
             throw new MyCimException(className + FileKey1 + " not found"); 
         } catch (Exception e) { 
             throw ExceptionHandler.handlerException(e, log); 
         } 
     } 
 我用的这个代码方法会有问题，我该怎么写能兼容并且改动最小，不要修改代码内容

## Assistant
根据您提供的信息，我理解现在的问题是：

**当前情况：**
- `wafersource` 作为 key（第一列代码）来查询类表值
- 以前 `wafersource=1` 时，第三列（产品分类）都是 `FT`
- 现在新增了 `FT_AE` 这条记录，`wafersource=1`，但产品分类变成了 `FT-AE`

**问题所在：**
您现有的 `getReferenceFileValue` 方法只根据 `FileKey1`（即 wafersource）进行匹配，当有多条记录的 key1 相同时，只会返回第一条匹配的记录。

**解决方案：**

既然您提到**不要修改代码内容**（指的是不要大幅改动现有逻辑），最小的兼容方案是：

1. **修改查询逻辑，传入更多参数**：在调用 `getReferenceFileValue` 时，不仅传入 `wafersource`，还传入"产品分类"作为额外的筛选条件。

2. **使用 `ReferenceFileDetail` 的其他字段**：从您截图看，`ReferenceFileDetail` 应该有多个属性值（key1Value, key2Value, key3Value 等）。您可以修改方法签名，增加一个 `FileKey2` 或 `FileKey3` 参数来匹配第三列。

**具体改动建议：**

```java
// 修改方法签名，增加产品分类参数
public ReferenceFileDetail getReferenceFileValue(String className, String fileKey1, String productCategory) {
    try {
        ReferenceFile referenceFile = cimSetupService.getReferenceFile(new ReferenceFile(className, 
                baseService.getNamedSpace(ThreadLocalContext.getFacilityRrn(), ObjectList.REFERENCE_FILE_KEY), 
                ObjectList.REFERENCE_FILE_KEY));
        if (referenceFile == null) {
            throw new MyCimException(className + " == null");
        }
        if (referenceFile.getValuesItems().isEmpty()) {
            throw new MyCimException(className + " == null");
        }
        
        for (ReferenceFileDetail referenceFileDetail : referenceFile.getValuesItems()) {
            // 同时匹配 key1 和 key3（产品分类）
            if (StringUtils.equals(referenceFileDetail.getKey1Value(), fileKey1) 
                && StringUtils.equals(referenceFileDetail.getKey3Value(), productCategory)) {
                return referenceFileDetail;
            }
        }
        throw new MyCimException(className + fileKey1 + " not found");
    } catch (Exception e) {
        throw ExceptionHandler.handlerException(e, log);
    }
}
```

**调用时需要传入：**
```java
// 原来的调用
// ReferenceFileDetail detail = getReferenceFileValue("类名", wafersource);

// 新的调用（需要传入产品分类）
ReferenceFileDetail detail = getReferenceFileValue("类名", wafersource, productCategory);
```

**如果不想改方法签名，也可以：**

保持方法签名不变，但在方法内部增加一个可选的匹配逻辑：

```java
public ReferenceFileDetail getReferenceFileValue(String className, String FileKey1, String productCategory) {
    // ... 前面代码不变 ...
    
    ReferenceFileDetail firstMatch = null;
    for (ReferenceFileDetail referenceFileDetail : referenceFile.getValuesItems()) {
        if (StringUtils.equals(referenceFileDetail.getKey1Value(), FileKey1)) {
            // 如果传入了产品分类，精确匹配
            if (StringUtils.isNotBlank(productCategory) 
                && StringUtils.equals(referenceFileDetail.getKey3Value(), productCategory)) {
                return referenceFileDetail; // 找到精确匹配，直接返回
            }
            // 记录第一个匹配项作为备选
            if (firstMatch == null) {
                firstMatch = referenceFileDetail;
            }
        }
    }
    
    // 如果没有精确匹配，返回第一个匹配项（保持向后兼容）
    if (firstMatch != null) {
        return firstMatch;
    }
    
    throw new MyCimException(className + FileKey1 + " not found");
}
```

您希望我帮您实现哪种方案？或者您想先看看 `ReferenceFileDetail` 类的结构，确认第三列对应的字段名是什么？

