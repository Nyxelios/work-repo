# Galaxycore MES 系统架构文档

## 一、项目概述

Galaxycore MES（MyCIM2）是面向半导体制造企业的核心执行系统，负责衔接计划层与生产层，覆盖从工单接收、物料绑定、生产执行、品质控制到成品入库的全过程管理。

### 1.1 项目定位（ISA-95）

```
Level 4：ERP / 经营管理层
  └─ 工单下发、物料与库存计划、财务与经营管理

Level 3：MES / 制造执行层
  └─ 工单管理、在制追踪、工艺执行、品质卡控、生产履历

Level 2：SCADA / 设备控制层
  └─ 设备状态采集、设备结果上传、测试数据交互

Level 1-0：设备与物理生产过程
  └─ Prober、Handler、Tester、晶圆加工、封装、测试、包装
```

### 1.2 支持产线类型

| 产线代码 | 产品分类 | 业务描述 |
|---------|---------|---------|
| `4` | COM | 芯片封装业务（晶圆绑定、辅料管控、封装过站、包装入库） |
| `1` | FT | 成品测试业务（IQC、ATE、合批、FT、FQC） |
| `5/6` | CP | 晶圆测试业务（SensorCP、LCD_CP） |
| `3` | WLT | 晶圆级测试/封装测试（WLFT、WLFTV、WLR） |
| `32` | FT-AE | 车载成品测试 |
| - | RW | 重工/Recon（返工、重测、异常补流程） |

---

## 二、技术架构

### 2.1 整体架构图

```
┌──────────────────────────────────────────────────────────────┐
│                      客户端层 (Browser)                        │
│    Ext.js SPA + JSP 页面  │  HTTP/HTTPS 请求                   │
└─────────────────────────────┬────────────────────────────────┘
                              │
┌─────────────────────────────▼────────────────────────────────┐
│                     Web 层 (Struts 1.x)                       │
│    ActionServlet → Action → JSP/JSON                          │
│    com.mycim.webapp.actions.*                                 │
└─────────────────────────────┬────────────────────────────────┘
                              │
┌─────────────────────────────▼────────────────────────────────┐
│                     业务逻辑层 (Service)                       │
│    Service Interface → ServiceImpl                            │
│    com.mycim.gc.service.impl.*                                │
│    ┌──────────┬──────────┬──────────┬──────────┐              │
│    │ 工单服务  │ LOT服务   │ 晶圆服务  │ 工艺服务  │              │
│    └──────────┴──────────┴──────────┴──────────┘              │
└─────────────────────────────┬────────────────────────────────┘
                              │
┌─────────────────────────────▼────────────────────────────────┐
│                      数据访问层 (DAO)                          │
│    Spring JDBC Template / 原生 SQL                            │
│    com.mycim.gc.dao.* / JPA Repository                        │
└─────────────────────────────┬────────────────────────────────┘
                              │
┌─────────────────────────────▼────────────────────────────────┐
│                      数据库层 (Oracle)                         │
│    Oracle 11g+  │  Tablespace: TBS_MYCIM                      │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 核心技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| **Web框架** | Struts 1.x | MVC框架，Action处理请求路由 |
| **前端框架** | Ext.js | UI组件库，表格/表单/图表 |
| **业务框架** | Spring IoC | Bean管理，依赖注入 |
| **定时任务** | Quartz | 定时任务调度（工单自动关闭等） |
| **数据访问** | Spring JDBC + JPA | 数据库操作封装 |
| **数据库** | Oracle 11g+ | 关系型数据库，Oracle RAC |
| **应用服务器** | WebLogic | 集群部署 |
| **构建工具** | Ant + Ivy | 项目构建与依赖管理 |
| **版本控制** | SVN | 源码版本管理 |

---

## 三、项目结构

### 3.1 目录结构

```
Code-Projects/gc/
├── core/                          # 核心代码模块
│   ├── src/
│   │   ├── gc/                    # Galaxycore业务模块
│   │   │   └── src/main/java/com/mycim/gc/
│   │   │       ├── service/       # 业务服务层
│   │   │       │   ├── impl/      # 服务实现
│   │   │       │   └── quartz/    # 定时任务
│   │   │       │       └── jobs/  # 定时任务实现
│   │   │       ├── model/         # 数据模型
│   │   │       ├── repository/     # 数据仓储
│   │   │       └── dao/           # 数据访问层
│   │   │
│   │   ├── model/                 # Web层（Action）
│   │   │   └── src/com/mycim/webapp/
│   │   │       └── actions/       # Struts Action
│   │   │           ├── prp/       # 生产相关Action
│   │   │           └── wip/       # WIP相关Action
│   │   │
│   │   ├── ejb/                   # EJB组件（遗留代码）
│   │   ├── workflow/              # 工作流引擎
│   │   ├── spc/                   # SPC统计过程控制
│   │   ├── module/                # 独立功能模块
│   │   │   ├── inv/               # 库存管理
│   │   │   └── fmb/               # 工艺流程管理
│   │   │
│   │   └── tool/                  # 工具服务
│   │
│   ├── web/                       # Web资源
│   │   ├── WEB-INF/
│   │   │   ├── config/spring/      # Spring配置
│   │   │   ├── struts-config.xml  # Struts配置
│   │   │   ├── web.xml            # Web部署描述符
│   │   │   └── logback.xml        # 日志配置
│   │   └── jsp/                   # JSP页面
│   │
│   └── build.xml                  # Ant构建脚本
│
├── database/                       # 数据库脚本
├── doc/                           # 项目文档
└── report/                        # 报表模块
```

### 3.2 命名规范

| 类型 | 包前缀 | 示例 |
|------|--------|------|
| Service接口 | `com.mycim.gc.service` | `WorkOrderService` |
| Service实现 | `com.mycim.gc.service.impl` | `WorkOrderServiceImpl` |
| Action | `com.mycim.webapp.actions` | `WorkOrderListAction` |
| DAO | `com.mycim.gc.dao` | `WorkOrderDao` |
| Repository | `com.mycim.gc.repository` | `WorkOrderRepository` |
| Model | `com.mycim.prp.model` | `WorkOrder` |
| Quartz Job | `com.mycim.gc.service.quartz.jobs` | `CpAutoPlanJob` |

---

## 四、核心模块说明

### 4.1 工单管理模块（gc/service）

**职责**：工单的创建、修改、查询、物料绑定、工单投入、工单关闭等全生命周期管理。

| 服务类 | 功能描述 |
|--------|----------|
| `WorkOrderService` | 工单主服务 |
| `WorkOrderPlanServiceImpl` | 工单计划服务 |
| `ErpWorkOrderServiceImpl` | ERP工单同步服务 |

### 4.2 制造执行模块（workflowtask）

**职责**：批次（LOT）的生产执行、过站、品质检验、完工入库等。

| 服务类 | 功能描述 |
|--------|----------|
| `ComWaferTrackService` | COM产线晶圆过站 |
| `CpTrackService` | CP产线过站 |
| `FtTrackService` | FT产线过站 |
| `RwTrackService` | RW重工过站 |
| `WltTrackService` | WLT晶圆测试过站 |

### 4.3 工艺规划模块（fmb）

**职责**：产品工艺流程定义、工步配置、AQL标准设定。

### 4.4 工作流引擎（workflow）

**职责**：通用的业务流程引擎，支持任务调度和流程控制。

### 4.5 定时任务模块（quartz/jobs）

| 任务类 | 触发规则 | 功能 |
|--------|----------|------|
| `CpAutoPlanJob` | Cron | CP自动排料 |
| `WltAutoPlanJob` | Cron | WLT自动排料 |
| `RwAutoPlanJob` | Cron | RW自动排料 |
| `AutoCloseComWorkOrderJob` | Cron | COM工单自动关闭 |
| `ReceiveErpWrokOrderJob` | Cron | ERP工单同步 |
| `ReceiveErpMaterialOrderJob` | Cron | ERP物料同步 |
| `WaferBindSyncJob` | Cron | 晶圆绑定同步 |

---

## 五、数据模型

### 5.1 核心业务对象

| 对象 | 表名 | 说明 |
|------|------|------|
| 工单 | `WORKORDER` | 来自ERP的生产指令 |
| 批次 | `LOT` | MES在制追踪的核心对象 |
| 晶圆 | `WMS_MMS_MATERIAL_LOT_UNIT` | 重要原材料 |
| 工艺流程 | `ROUTE` | 生产步骤定义 |
| 工步 | `ROUTE_STEP` | 流程中的单个站点 |
| 等级 | `BIN_GRADE` | 测试或品质等级 |
| 缺陷 | `DEFECT` | 不良与异常记录 |

---

## 六、部署架构

### 6.1 生产环境

```
┌─────────────────────────────────────────────┐
│           生产环境 (Production)               │
│  ┌───────────────────────────────────────┐  │
│  │  WebLogic Cluster                      │  │
│  │  ┌─────────┐  ┌─────────┐            │  │
│  │  │ Node 1  │  │ Node 2  │            │  │
│  │  └─────────┘  └─────────┘            │  │
│  └───────────────────────────────────────┘  │
│  ┌───────────────────────────────────────┐  │
│  │  Oracle RAC                            │  │
│  │  ┌─────────┐  ┌─────────┐            │  │
│  │  │ Node 1  │  │ Node 2  │            │  │
│  │  └─────────┘  └─────────┘            │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### 6.2 构建发布流程

```
SVN 源码 → Jenkins 构建 → Ant 打包 → WebLogic 部署
```

---

## 七、集成接口

### 7.1 ERP集成

| 方向 | 方式 | 核心表 |
|------|------|--------|
| ERP → MES | 中间表 | `ERP_WORKORDER` |
| MES → ERP | 接口调用 | 工单状态、入库信息 |

### 7.2 WMS集成

| 方向 | 方式 | 核心表 |
|------|------|--------|
| WMS → MES | 数据库同步 | `WMS_MMS_MATERIAL_LOT_UNIT` |
| MES → WMS | 接口调用 | 物料状态变更 |

---

## 八、相关文档

- [系统概览](../1.MES系统/00-系统概览.md) - 业务层面概览
- [技术架构](../1.MES系统/01-技术架构.md) - 技术栈详解
- [模块关系图](../1.MES系统/02-模块关系图.md) - 模块依赖与数据流
- [集成接口](../1.MES系统/03-集成接口.md) - 外部系统集成说明
- [开发者指南](DEVELOPER_GUIDE.md) - 开发环境与编码规范
- [核心模块说明](MODULES.md) - 模块详细设计
- [API参考](API_REFERENCE.md) - 关键类与接口说明
- [数据库设计](DATABASE.md) - 数据表结构说明
