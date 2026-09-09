---
title: FT测试型号和程序关系维
date: 2024-10-21
author: MES Team
status: completed
tags: [工艺, FT, 功能]
---

> 数据表需要增加一个字段名，然后在前端页面显示出来，数据表是`lot`表`GC_FT_MODEL_MANAGEMENT`表

# 增加科迪程序
## 数据表修
💡 首先新建一个linquibase文件，然后重新打包项目，再启动完成数据表的修
![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241017143216.png)

## 前端修改
![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241017143232.png)

## DO

> 这次的问题看似简单，实际上也有些小复杂，昨天只是把`lot表`添加了`TEST_PROGRAM_ID_KEDI`字段，同时在`ftProgramRelationshipMaintenanceGrid.js`把和`programId`有关的代码都给`programIdKedi`复制了一

💡 弹窗和普通页面的`form`和`grid`都要把`programIdKedi`写一

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241017143513.png)

💡 `FtProgramRelationshipMaintenanceAction`文件中把所有有关`programId`属性设置的都设置下

💡 `GcFtModelManagement`和`Lot`实体类都添加一

```java
@Column(name="PROGRAM_ID_KEDI")
private String programIdKedi;
```

 💡 同时在`FtServiceImpl.java`配置一行代
 
![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241017143536.png)

## 出现问题

💡 开始，点击加工批次报错，原因是数据库问题，用的是本地数据库，改用测试库后可以了

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241017143609.png)
![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241017143620.png)

## 检验成

1. FT测试型号和程序关系中，添加型号以及流程号，FT，RT，FA ![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241017143647.png)
 1. 然后在lot操作中的开始结束作业里，点击FT、RT、FA的三种操作测
 ![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241017143711.png)
2. 三种类似，选择RT作为示例，选择自己添加的入库型号和流程号对应的一项，添加到批次，然后开始处
![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241017143732.png)

1. 开始加工批
2. 查找数据
![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241017150212.png)

查找到了自己添加的，表示成功了

💡 每次操作结束，都要在这里输入批次号，然后确认，点击结束，进入下一站，一般顺序是`FT>RT>FA`

# FT测试型号和程序关系维护栏位修
> 工艺规划下FT配置管理的FT测试型号和程序关系维护，删除StepId一栏，增加目标等级和优先级两个栏位

## 增加数据库字

### db.changelog-0.0.4.gc_ft_model_management.yaml

```yaml
databaseChangeLog:
  - changeSet:
      id: addcolumn-0.0.4-gc_ft_model_management
      author: Natsume_wang
      comment: addcolumn TARGET_GRADE and PRIORITY
      changes:
        - addColumn:
            tableName: GC_FT_MODEL_MANAGEMENT
            columns:
              - column:
                  name: TARGET_GRADE
                  type: varchar2(128)
              - column:
                  name: PRIORITY
                  type: number(3)
```

### db.changelog-0.0.4.gc_ft_model_management_h.yaml

```yaml
databaseChangeLog:
  - changeSet:
      id: addcolumn-0.0.4-gc_ft_model_management_h
      author: Natsume_wang
      comment: addcolumn TARGET_GRADE and PRIORITY
      changes:
        - addColumn:
            tableName: GC_FT_MODEL_MANAGEMENT_H
            columns:
              - column:
                  name: TARGET_GRADE
                  type: varchar2(128)
              - column:
                  name: PRIORITY
                  type: number(3)
```

### db.changelog-master.yaml

```yaml
- include:
      file: db.changelog-0.0.4.gc_ft_model_management.yaml
      relativeToChangelogFile: true
  - include:
      file: db.changelog-0.0.4.gc_ft_model_management_h.yaml
      relativeToChangelogFile: true
```

## 前端

### 增删栏位

前端有关`stepId`的代码都注释掉，同时加上关于`targetId`和`priority`的代

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241017142932.png)

## 后端

### 实体

```java
@Column(name="TARGET_GRADE")
private String targetGrade;

@Column(name="priority")
private Integer priority;
```

💡 有`stepId`代码的地方，都改成`targetId`和`priority`的代码就行了

```mermaid
graph TD;
    A[FT配置管理] --> B(删除 StepId);
    A --> C(增加 目标等级);
    A --> D(增加 优先;

```

