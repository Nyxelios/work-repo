---
title: 产品良率HOLD不生效
date: 2026-05-03
tags: [MES, 工作流]
---

![](https://files-1259440452.cos.ap-nanjing.myqcloud.com/Obsidian/e33fee9337728eeb2de62b2399267d7e.png)

配置了CAOI不生效，我查了代码发现，代码竟然有两处问题
1.CAOI已经是合大批次了，lotext表的wdbid字段已经不会有waferid的信息了，但是代码执行需要有值才会进入
![](https://files-1259440452.cos.ap-nanjing.myqcloud.com/Obsidian/20260312144141454.png)
这里我加了一个判断语句，去获取workorder的waferlevel去兜底
2.当我以为万事大吉后，发现还是执行不了，因为，它里面还有一层是按照工步执行的，只有指定的工步才会hold，不是看你配了什么工步就执行。
![697](https://files-1259440452.cos.ap-nanjing.myqcloud.com/Obsidian/20260312144312383.png)
这里我加了一个工步CAOI。