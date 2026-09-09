---
title: COM真空包备注修
date: 2024-10-21
author: MES Team
status: completed
tags: [包装, COM, 功能]
---

# 工单在线数量为负

>[!ERROR] 工单在线数量为负
> 该问题之前出现过，查到是由于在入库时，后台没有对包装的状态进行卡控，导致出现了重复入库。即在入库后工单数量减少，又再一次入库

## 解决方法

在入库函数中添加判断，如果vbox状态不是pqc，则不可入库（只有pqc可以入库，入库后状态会被修改为IN
```java
if (!StringUtils.equals(packedLot.getPackedStatus(), PackedLot.TRANS_PQC)){  
   throw new MyCimException(ExceptionContent.VBOX_STATUS_IS_INCORRECT);  
}
```

## 2024/11/4

又一次出现了这个问题，查到是真空包真空检验导致的问题，不卡控IN，重新检验后状态就变PQC了，所以又入库了。这个问题找到了，不过先不改。。。[[真空包真空检验]]