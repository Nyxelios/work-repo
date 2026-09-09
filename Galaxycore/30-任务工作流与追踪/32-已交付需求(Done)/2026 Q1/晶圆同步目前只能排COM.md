---
title: 晶圆同步目前只能排COM
date: 2026-05-03
tags: [MES, 工作流]
---
![](https://files-1259440452.cos.ap-nanjing.myqcloud.com/Obsidian/20260312160445478.png)
之前COM晶圆之外的也可以排，排了COG的后，因为COG和COM真空包的调用方法不一样，COG调用COM真空包的代码后会出现绑定异常，导致lotplan数据不会生成。暂时先卡控COM才能排。