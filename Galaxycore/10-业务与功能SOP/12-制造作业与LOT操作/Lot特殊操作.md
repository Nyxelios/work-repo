---
title: Lot特殊操作
date: 2024-10-21
author: MES Team
status: completed
tags: [制造管理, LOT, 功能]
---

# 优化
## lotLocation

### 异常案例

1. 批次状态 DISPATCH，JobRrn有值但Job表中不存在，getJob(lot.getJobRrn())获取的Job为NuLL

![dfed9311ad300e55d9e655c420ecdc16.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/dfed9311ad300e55d9e655c420ecdc16.png)


> [!NOTE] 处理方案
>  if (job != null && job.getEqptRrn() != null) {
> 	 //job不为空执行原逻辑
>  } else {
> 	 lot.setLotStatus(LotStatus.WAITING);  
> 	 lotServiceInterface.updateLotStatus(lot);
>  }

2. 批次状态为processed，作业报错，原因也是job为NULL

==cancelLotMoveIn==
```java
Long entityRrn = null;  
//job 为NULL时空指针报错  
if (job == null) {  
    entityRrn = lot.getEqptRrn();  
} else {  
    entityRrn = job.getEqptRrn();  
}
```

