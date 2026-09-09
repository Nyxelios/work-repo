# Galaxycore MES Code Wiki

> Galaxycore MES（MyCIM2）项目的完整代码知识库文档

## 文档概览

本Code Wiki旨在为开发人员提供Galaxycore MES系统的完整技术文档，包括系统架构、模块设计、API参考、数据库设计等关键信息。

## 核心文档

### 📚 架构与设计

| 文档 | 说明 |
|------|------|
| [系统架构](ARCHITECTURE.md) | 整体技术架构、模块关系、部署架构 |
| [核心模块说明](MODULES.md) | 各业务模块的详细设计和职责 |
| [数据库设计](DATABASE.md) | 数据表结构、ER关系、常用SQL |

### 🔧 开发指南

| 文档 | 说明 |
|------|------|
| [开发者指南](DEVELOPER_GUIDE.md) | 开发环境搭建、代码规范、新功能开发流程 |
| [API参考](API_REFERENCE.md) | 核心服务接口、Web Action、数据仓储接口 |

## 快速导航

### 按领域导航

```
工单管理
  └─ WorkOrderService
  └─ WorkOrderRepository
  └─ WORKORDER表

制造执行
  └─ ComWaferTrackService (COM产线)
  └─ CpTrackService (CP产线)
  └─ FtTrackService (FT产线)
  └─ RwTrackService (RW重工)
  └─ LOT表

工艺规划
  └─ FmbService
  └─ Route/RouteStep表

物料管理
  └─ WaferBindSyncService
  └─ WMS物料表

定时任务
  └─ Quartz Jobs
  └─ 自动排料工单
  └─ 工单同步

工作流引擎
  └─ WorkflowEngineService
  └─ 工作流表

打印服务
  └─ PrintService
  └─ 流程卡/标签打印
```

### 按技术栈导航

```
前端
  └─ Ext.js
  └─ JSP

Web层
  └─ Struts 1.x
  └─ Action/Form

业务层
  └─ Spring IoC
  └─ Service

数据层
  └─ Spring JDBC
  └─ JPA Repository
  └─ Oracle

基础设施
  └─ Quartz (定时任务)
  └─ WebLogic (应用服务器)
  └─ Ant + Ivy (构建)
```

## 技术栈

| 组件 | 技术 | 版本 |
|------|------|------|
| Web框架 | Struts | 1.x |
| 前端框架 | Ext.js | - |
| 业务框架 | Spring | 5.x |
| 定时任务 | Quartz | - |
| 数据库 | Oracle | 11g+ |
| 应用服务器 | WebLogic | 12c+ |
| 构建工具 | Ant + Ivy | - |

## 支持的产线

| 代码 | 类型 | 说明 |
|------|------|------|
| `4` | COM | 芯片封装 |
| `1` | FT | 成品测试 |
| `5/6` | CP | 晶圆测试 |
| `3` | WLT | 晶圆级测试 |
| `32` | FT-AE | 车载测试 |
| `RW` | Recon | 重工 |

## 链接

- [业务知识库](../1.MES系统/00-系统概览.md)
- [MES功能导航](../1.MES系统/MES功能导航.md)
- [开发知识库](../1.MES系统/开发知识库/)
- [运维手册](../1.MES系统/运维手册/)
- [工作流记录](../2.工作流/)
