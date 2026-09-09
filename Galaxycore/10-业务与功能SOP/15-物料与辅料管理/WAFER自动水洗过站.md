---
title: WAFER自动水洗过站
date: 2024-11-27
author: MES Team
status: completed
tags: [晶圆, 功能]
---

# WAFER自动水洗过站

## 需求清单

- [ ] 系统合并 `palms` 和水洗两个站点
- [x] 水洗站点取消人工处理动作，系统抓取清洗记录后自动过站
- [x] 未查询到清洗 Wafer 信息时，下一站“倒膜”不允许过站，并做卡控提示

特殊型号清洗管控：

- [x] `GC5035` 所有型号不清洗，系统默认该型号在清洗站正常过站
- [x] `inline` 线 `GC08A8` / `GC08A3` 两个型号不清洗；MES 在“晶圆处理”界面展示计划排料备注中的 `inline` 标记，并按此默认正常过站

## 系统抓取清洗记录自动过站

### 怎么编写接口

MES 之前已经有不少接口实现，可以先参考现有实现类复制一份模板。接口方法的编写本身和普通方法差别不大，关键在于怎样让外部系统能够正常调用。

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241203172153.png)

> [!WARN]
> 比较麻烦的是，Wafer 相关的方法、实现类和实体类都不能直接导入；导入后虽然编译阶段不报错，但打包时会在控制台直接报错。最终只能重写新的实现类，再通过新类去调用。

### 怎么让接口生效

参考现有接口配置后发现，要让新增接口生效，需要在下面两个配置文件中补充自己的接口名。这里填写的是接口实现的入口服务文件名，作用类似启动入口注册。

#### `server-config.wsdd`

![](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241203172337.png)

#### `deploy.wsdd`

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241203172247.png)

### 验证方式

![](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241203172845.png)

1. 访问 [localhost:7001/mycim2/services](http://localhost:7001/mycim2/services)，如果能看到自己的接口名，说明服务注册已完成。
2. 使用 Postman 调用已编写接口，并结合 IDEA Debug 进行联调。

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/pictures/20241203173117.png)

## 需要写哪些接口？

接口名称、入参和返回值建议按现场清洗记录文件格式统一定义，再与 MES 当前晶圆处理流程对齐。
