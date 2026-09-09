# Galaxycore MES 数据库设计文档

## 目录

- [一、数据库概述](#一数据库概述)
- [二、核心表结构](#二核心表结构)
- [三、工单相关表](#三工单相关表)
- [四、批次(WIP)相关表](#四批次wip相关表)
- [五、物料相关表](#五物料相关表)
- [六、工艺流程相关表](#六工艺流程相关表)
- [七、品质相关表](#七品质相关表)
- [八、工作流相关表](#八工作流相关表)
- [九、系统配置表](#九系统配置表)
- [十、常用SQL示例](#十常用sql示例)

---

## 一、数据库概述

### 1.1 数据库信息

| 属性 | 值 |
|------|------|
| 数据库类型 | Oracle 11g+ |
| 表空间 | TBS_MYCIM |
| 字符集 | AL32UTF8 / ZHS16GBK |
| 主键策略 | 序列(SEQUENCE) + 触发器 |

### 1.2 表分类

| 分类 | 前缀 | 说明 |
|------|------|------|
| 工单表 | WORKORDER_* | 工单相关 |
| 批次表 | LOT_* | 批次/WIP相关 |
| 物料表 | WMS_* / INV_* | 物料库存相关 |
| 工艺表 | ROUTE_* / FMB_* | 工艺流程相关 |
| 品质表 | DEFECT_* / SPC_* | 品质检验相关 |
| 工作流表 | WF_* | 工作流相关 |
| 系统表 | SYS_* / BAS_* | 系统配置相关 |

---

## 二、核心表结构

### 2.1 ER关系图

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│ WORKORDER   │────<│WORK_ORDER_RELATION│>────│  MATERIAL   │
│  (工单)     │     │  (工单物料关系)  │     │  (物料)     │
└─────────────┘     └─────────────────┘     └─────────────┘
       │
       │ 1:N
       ▼
┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│    LOT      │────<│ LOT_TRANSACTION │     │   ROUTE     │
│  (批次)     │     │  (批次事务)     │     │  (工艺路线) │
└─────────────┘     └─────────────────┘     └─────────────┘
       │
       │ 1:N
       ▼
┌─────────────┐     ┌─────────────────┐
│  LOT_STEP   │     │    DEFECT       │
│  (站点历史) │     │    (缺陷)       │
└─────────────┘     └─────────────────┘
```

---

## 三、工单相关表

### 3.1 WORKORDER（工单主表）

| 字段 | 类型 | 说明 |
|------|------|------|
| RRN | NUMBER(18) | 主键 |
| WORKORDER_ID | VARCHAR2(50) | 工单号 |
| PRODUCT_ID | VARCHAR2(50) | 产品ID |
| PRODUCT_CLASSIFY | VARCHAR2(10) | 产品分类(4=COM,1=FT,5/6=CP,3=WLT) |
| QTY | NUMBER(18,4) | 数量 |
| COMPLETED_QTY | NUMBER(18,4) | 已完成数量 |
| STATUS | VARCHAR2(20) | 状态 |
| WORK_ORDER_TYPE | VARCHAR2(20) | 工单类型 |
| BONDED_PROPERTY | VARCHAR2(20) | 绑定属性 |
| INNER_ORDER_NO | VARCHAR2(50) | 内部订单号 |
| FACTORY_ID | VARCHAR2(20) | 工厂ID |
| ERP_WORKORDER_ID | VARCHAR2(50) | ERP工单号 |
| START_TIME | DATE | 计划开始时间 |
| END_TIME | DATE | 计划结束时间 |
| CREATE_TIME | DATE | 创建时间 |

**索引**：
- `IDX_WORKORDER_01` ON (WORKORDER_ID)
- `IDX_WORKORDER_02` ON (STATUS, FACTORY_ID)

### 3.2 WORK_ORDER_RELATION（工单物料关系表）

| 字段 | 类型 | 说明 |
|------|------|------|
| RRN | NUMBER(18) | 主键 |
| WORK_ORDER_RRN | NUMBER(18) | 工单RRN |
| OBJECT_TYPE | VARCHAR2(30) | 对象类型(WAFER/MATERIAL) |
| OBJECT_RRN | NUMBER(18) | 对象RRN |
| OBJECT_ID | VARCHAR2(50) | 对象ID |
| QTY | NUMBER(18,4) | 数量 |
| STATE | VARCHAR2(20) | 状态 |

---

## 四、批次(WIP)相关表

### 4.1 LOT（批次主表）

| 字段 | 类型 | 说明 |
|------|------|------|
| RRN | NUMBER(18) | 主键 |
| LOT_ID | VARCHAR2(50) | 批次号 |
| WORKORDER_ID | VARCHAR2(50) | 工单号 |
| PRODUCT_ID | VARCHAR2(50) | 产品ID |
| QTY | NUMBER(18,4) | 数量 |
| STATUS | VARCHAR2(20) | 状态 |
| CURRENT_STEP_ID | VARCHAR2(50) | 当前工步ID |
| FACTORY_ID | VARCHAR2(20) | 工厂ID |
| CREATE_TIME | DATE | 创建时间 |
| MOVE_IN_TIME | DATE | 进站时间 |

### 4.2 LOT_TRANSACTION（批次事务表）

| 字段 | 类型 | 说明 |
|------|------|------|
| RRN | NUMBER(18) | 主键 |
| LOT_RRN | NUMBER(18) | 批次RRN |
| TRANSACTION_TYPE | VARCHAR2(30) | 事务类型 |
| STEP_ID | VARCHAR2(50) | 工步ID |
| EQUIPMENT_ID | VARCHAR2(50) | 设备ID |
| OPERATION_TIME | DATE | 操作时间 |

---

## 五、物料相关表

### 5.1 WMS_MMS_MATERIAL_LOT_UNIT（物料单元表）

| 字段 | 类型 | 说明 |
|------|------|------|
| RRN | NUMBER(18) | 主键 |
| UNIT_ID | VARCHAR2(50) | 单元ID |
| MATERIAL_LOT_ID | VARCHAR2(50) | 物料批次号 |
| WAFER_ID | VARCHAR2(50) | 晶圆ID |
| GRADE | VARCHAR2(20) | 等级 |
| STATE | VARCHAR2(20) | 状态 |

**晶圆状态**：
- `Create` - 已创建，未发料
- `Issue` - 已发料
- `In` - 已入站/已接收
- `Package` - 已包装

---

## 六、工艺流程相关表

### 6.1 ROUTE（工艺路线表）

| 字段 | 类型 | 说明 |
|------|------|------|
| RRN | NUMBER(18) | 主键 |
| ROUTE_ID | VARCHAR2(50) | 路线ID |
| ROUTE_NAME | VARCHAR2(100) | 路线名称 |
| PRODUCT_CLASSIFY | VARCHAR2(10) | 产品分类 |
| VERSION | VARCHAR2(20) | 版本 |
| STATE | VARCHAR2(20) | 状态 |

### 6.2 ROUTE_STEP（工艺工步表）

| 字段 | 类型 | 说明 |
|------|------|------|
| RRN | NUMBER(18) | 主键 |
| ROUTE_RRN | NUMBER(18) | 路线RRN |
| STEP_ID | VARCHAR2(50) | 工步ID |
| STEP_SEQ | NUMBER(10) | 工步顺序 |
| STEP_TYPE | VARCHAR2(20) | 工步类型 |
| EQP_GROUP_ID | VARCHAR2(50) | 设备组ID |

---

## 七、品质相关表

### 7.1 DEFECT（缺陷记录表）

| 字段 | 类型 | 说明 |
|------|------|------|
| RRN | NUMBER(18) | 主键 |
| LOT_RRN | NUMBER(18) | 批次RRN |
| DEFECT_CODE | VARCHAR2(50) | 缺陷代码 |
| DEFECT_QTY | NUMBER(18,4) | 缺陷数量 |
| RECORD_TIME | DATE | 记录时间 |

---

## 八、工作流相关表

### 8.1 WF_WORKFLOW（工作流定义表）

| 字段 | 类型 | 说明 |
|------|------|------|
| RRN | NUMBER(18) | 主键 |
| WORKFLOW_ID | VARCHAR2(50) | 工作流ID |
| WORKFLOW_NAME | VARCHAR2(100) | 工作流名称 |
| VERSION | VARCHAR2(20) | 版本 |
| STATE | VARCHAR2(20) | 状态 |

### 8.2 WF_INSTANCE（工作流实例表）

| 字段 | 类型 | 说明 |
|------|------|------|
| RRN | NUMBER(18) | 主键 |
| INSTANCE_ID | VARCHAR2(50) | 实例ID |
| WORKFLOW_RRN | NUMBER(18) | 工作流RRN |
| BUSINESS_KEY | VARCHAR2(100) | 业务键 |
| STATUS | VARCHAR2(20) | 状态 |

---

## 九、系统配置表

### 9.1 REFERENCE_FILE（参照文件表）

| 字段 | 类型 | 说明 |
|------|------|------|
| RRN | NUMBER(18) | 主键 |
| CLASS_NAME | VARCHAR2(100) | 类别名称 |
| FILE_KEY1 | VARCHAR2(100) | 键1 |
| DATA1_VALUE | VARCHAR2(200) | 值1 |
| DATA2_VALUE | VARCHAR2(200) | 值2 |

---

## 十、常用SQL示例

### 10.1 工单查询

```sql
SELECT w.WORKORDER_ID, w.PRODUCT_ID, w.QTY, w.STATUS
FROM WORKORDER w
WHERE w.WORKORDER_ID = :workOrderId;
```

### 10.2 批次查询

```sql
SELECT l.LOT_ID, l.PRODUCT_ID, l.QTY, l.STATUS, l.CURRENT_STEP_ID
FROM LOT l
WHERE l.FACTORY_ID = :factoryId AND l.STATUS IN ('Waiting', 'InProcess');
```

### 10.3 品质统计

```sql
SELECT d.DEFECT_CODE, SUM(d.DEFECT_QTY) AS TOTAL_QTY
FROM DEFECT d
WHERE d.RECORD_TIME BETWEEN :startTime AND :endTime
GROUP BY d.DEFECT_CODE;
```

---

## 相关文档

- [系统概览](../1.MES系统/00-系统概览.md) - 业务知识
- [技术架构](../1.MES系统/01-技术架构.md) - 技术栈详解
- [开发者指南](DEVELOPER_GUIDE.md) - 开发规范
- [核心模块说明](MODULES.md) - 模块详细设计
- [API参考](API_REFERENCE.md) - API接口说明
