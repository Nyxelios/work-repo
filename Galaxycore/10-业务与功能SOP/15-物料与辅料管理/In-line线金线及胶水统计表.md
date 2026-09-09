---
title: In-line线金线及胶水统计
date: 2024-06-21
author: MES Team
status: completed
tags: [胶水管理, 功能]
---

> 入职第二周，师傅给我下发了一个任务，尝试自己给项目增加一个金线及胶水的功能。从周二开始下发任务到周三下午下班前大差不差的做完了。（6.21-6.22

# 1. 数据表设

## 1.1 数据结构

师傅是直接给我发了一个excel表格，让我根据表格的字段做一个增删改查的功能。所以就先从创建Orcale数据表开始吧

[In-line线金线及胶水统计xlsx](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9b9e6515-c3de-4767-ba99-059ebbaf2d27/In-line%E7%BA%BF%E9%87%91%E7%BA%BF%E5%8F%8A%E8%83%B6%E6%B0%B4%E7%BB%9F%E8%AE%A1%E8%A1%A8.xlsx)

![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241017101420.png)

## 1.2 liquibase框架

```yaml
databaseChangeLog:
  - changeSet:
      id: create-GLUE_RECORD-table
      author: Natsume_Wang
      comment: Create table GLUE_RECORD
      changes:
        - createTable:
            tableName: GLUE_RECORD
            remarks: inline金线胶水记录
            tablespace: TS_MYCIM_DAT
            columns:
              - column:
                  name: OBJECT_RRN
                  type: bigint
                  autoIncrement: false
                  constraints:
                    primaryKey: true
                    nullable: false
                  remarks: 主键
              - column:
                  name: WORK_ID
                  type: varchar2(64)
                  remarks: 工号
              - column:
                  name: DATE_TIME
                  type: date
                  remarks: 日期
              - column:
                  name: BATCH_NUMBER
                  type: varchar2(64)
                  remarks: 批号
              - column:
                  name: TYPE
                  type: varchar2(24)
                  remarks: 类别
              - column:
                  name: DEVICE_NUMBER
                  type: varchar2(36)
                  remarks: 设备
              - column:
                  name: MODEL_TYPE
                  type: varchar2(32)
                  remarks: 型号   
              - column:
                  name: OUTPUT
                  type: varchar2(32)
                  remarks: 产出
              - column:
                  name: REMARK
                  type: varchar2(36)
                  remarks: 备注
```

# 2. 问题总结

> 由于大体操作和菜单自测相似，所以不做详细介绍，总结下遇到的一些问题
> 
> [菜单功能-自測](https://www.notion.so/9178cb2c8ab04e4a89e41f93bd75b008?pvs=21)

1. 前端方面：遇到objectRrn传值时为null的问题，处理方法防止文件重名，将objectRrn取名为objectId等不同的名字

# 3.数据

`GLUE_RECORD`