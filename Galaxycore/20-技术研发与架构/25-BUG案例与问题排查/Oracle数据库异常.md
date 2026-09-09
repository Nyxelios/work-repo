---
title: Oracle数据库异
date: 2024-10-21
author: MES Team
status: completed
tags: [BUG, 运维]
---

### ORA-12638: 身份证明检索失

[https://blog.csdn.net/wjx_jasin/article/details/84649962?spm=1001.2101.3001.6650.4&utm_medium=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~default-4-84649962-blog-1503298.pc_relevant_multi_platform_whitelistv1_exp2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~default-4-84649962-blog-1503298.pc_relevant_multi_platform_whitelistv1_exp2&utm_relevant_index=7](https://blog.csdn.net/wjx_jasin/article/details/84649962?spm=1001.2101.3001.6650.4&utm_medium=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~default-4-84649962-blog-1503298.pc_relevant_multi_platform_whitelistv1_exp2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~default-4-84649962-blog-1503298.pc_relevant_multi_platform_whitelistv1_exp2&utm_relevant_index=7)

使用应用程序连接 Oracle 时碰到了 “ORA-12638: 身份证明检索失败错误，是因为 Oracle 的高级安全性验证导致

解决办法如下

1. 找到 Oracle 安装目录下的 NETWORK/admin/sqlnet.ora 修改 SQLNET.AVTHENTICATION_SERVICE=(NONE)
2. 开-> 程序 -> Oracle -> Configuration and Migration Tools -> Net Manager→本地→概要文件→Oracle 高级安全性→验证→去掉所选方法中"NTS" 就可以了.

### **oracle PL/SQL 查询结果不可更新**

[https://blog.csdn.net/wltsysterm/article/details/121997001](https://blog.csdn.net/wltsysterm/article/details/121997001)

我们在对Oracle数据库进行操作时，有时会在查询完结果后想要对其中的某些数据进行操作，当我们点击编辑（一个锁标志）是，会提示我们上述问题中的错误：这些查询结果不可更新，请使用ROWID或者SELECT……FOR UPDATE获得可更新结果。按照错误提示的信息我们可以采用两种解决办法

`解决办法1`：在查询语句后面写上for update，如：`select * from 表名 for update；`

`解决办法2`：在查询的列中使用rowid属性，如：`select rowID, 表名.* from 表名;`

### sql语句中为什么会有t，l

表别

```sql
select e.expid,e.state,e.perId,p.pername as pername,d.content as stateStr from expense as e

inner join person as p on (e.perid = p1.perid)

inner join diction as d on (e.state=d.keyword and d.type='expense')
```