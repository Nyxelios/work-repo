# Galaxycore MES 核心模块说明

## 目录

- [一、工单管理模块](#一工单管理模块)
- [二、制造执行模块](#二制造执行模块)
- [三、工艺规划模块](#三工艺规划模块)
- [四、晶圆管理模块](#四晶圆管理模块)
- [五、库存管理模块](#五库存管理模块)
- [六、工作流引擎](#六工作流引擎)
- [七、定时任务模块](#七定时任务模块)
- [八、打印服务模块](#八打印服务模块)
- [九、SPC统计过程控制](#九spc统计过程控制)

---

## 一、工单管理模块

### 1.1 模块概述

工单管理是MES系统的核心模块，负责接收ERP下发的生产工单，管理工单的全生命周期。

### 1.2 核心服务

**WorkOrderService**

**包路径**：`com.mycim.gc.service.WorkOrderService`

| 方法 | 说明 |
|------|------|
| `getWorkOrderId()` | 生成工单号 |
| `createWorkOrder()` | 创建工单 |
| `mergeWorkOrder()` | 合并工单 |
| `addWorkOrderRelation()` | 添加物料关系 |
| `deleteWorkOrderRelation()` | 删除物料关系 |
| `checkWorkorderTime()` | 校验工单时间 |
| `autoCloseComStartedZeroWipWorkOrders()` | 自动关闭零在制工单 |

### 1.3 工单状态流转

```
待接收 → 已接收 → 已投入 → 生产中 → 已完工 → 已入库
         ↓
      已取消
```

---

## 二、制造执行模块

### 2.1 模块概述

制造执行模块负责批次（LOT）的生产执行，支持COM、FT、CP、WLT等多种产线类型。

### 2.2 核心服务

| 服务类 | 产线 | 职责 |
|--------|------|------|
| `ComWaferTrackService` | COM | 芯片封装产线晶圆过站 |
| `CpTrackService` | CP | 晶圆测试产线过站 |
| `FtTrackService` | FT | 成品测试产线过站 |
| `RwTrackService` | RW | 重工/重测批次过站 |
| `WltTrackService` | WLT | 晶圆级测试过站 |

### 2.3 批次状态流转

```
Created → Waiting → MoveIn → InProcess → MoveOut → Completed
                         ↓
                      Held (品质异常)
```

---

## 三、工艺规划模块

### 3.1 模块概述

工艺规划模块负责产品和工艺流程的定义、配置和管理。

### 3.2 核心服务

| 服务类 | 功能描述 |
|--------|----------|
| `FmbService` | 工艺流程主服务 |
| `FmbFTDao` | FT工艺查询 |
| `FmbGCDao` | 通用工艺查询 |

### 3.3 工艺对象模型

```
Product (产品) → Route (工艺路线) → RouteStep (工步)
    ├── EquipmentConstraint (设备约束)
    ├── RecipeParameter (配方参数)
    └── TimeLimit (时间限制)
```

---

## 四、晶圆管理模块

### 4.1 核心服务

| 服务类 | 功能描述 |
|--------|----------|
| `WaferBindSyncService` | 晶圆绑定同步服务 |
| `WmsWaferManagerService` | WMS晶圆管理服务 |

### 4.2 晶圆状态

```
Create → Issue → In → Package → Shipped
         ↓
      Scrapped (报废)
```

---

## 五、库存管理模块

### 5.1 核心服务

| 服务类 | 功能描述 |
|--------|----------|
| `WarehouseService` | 仓库管理 |
| `LotInventoryService` | 批次库存管理 |

---

## 六、工作流引擎

### 6.1 核心服务

| 组件 | 说明 |
|------|------|
| `WorkflowEngineService` | 引擎核心服务 |
| `WorkflowManagerService` | 流程管理服务 |
| `WorkflowEditorService` | 流程编辑器服务 |

### 6.2 流程组件

| 组件 | 包路径 | 说明 |
|------|--------|------|
| 引擎核心 | `com.mycim.workflow.engine` | 流程执行引擎 |
| 编辑器 | `com.mycim.workflow.editor` | 流程设计器 |
| 任务服务 | `com.mycim.workflow.task` | 任务执行服务 |

---

## 七、定时任务模块

### 7.1 核心任务

| 任务类 | 功能 |
|--------|------|
| `CpAutoPlanJob` | CP产线自动排料工单 |
| `AutoCloseComWorkOrderJob` | 定时关闭零在制的COM工单 |
| `ReceiveErpWrokOrderJob` | 从ERP中间表同步工单数据 |
| `WaferBindSyncJob` | 晶圆绑定状态同步 |

### 7.2 任务配置

配置文件：`web/WEB-INF/config/spring/applicationContext-quartz.xml`

---

## 八、打印服务模块

### 8.1 核心服务

**PrintService**

| 方法 | 说明 |
|------|------|
| `printFlowcard()` | 打印流程卡 |
| `printLabel()` | 打印标签 |
| `printReport()` | 打印报表 |

### 8.2 打印数据类型

| 类 | 产线/站点 | 说明 |
|---|----------|------|
| `RclnInfo` | CLN清洗站 | 清洗信息 |
| `RiqcInfo` | IQC来料检验 | IQC信息 |
| `RpkgInfo` | PKG包装站 | 包装信息 |
| `RfqcInfo` | FQC终检站 | FQC信息 |

---

## 九、SPC统计过程控制

### 9.1 核心服务

| 服务类 | 功能 |
|--------|------|
| `RuleServerService` | SPC规则服务 |
| `RealTimeDataProcessor` | 实时数据处理器 |

### 9.2 SPC规则

| 规则编号 | 描述 |
|----------|------|
| Rule 1 | 1点超出3σ |
| Rule 2 | 连续9点在中心线同一侧 |
| Rule 3 | 连续6点递增或递减 |
| Rule 4 | 连续14点交替上下 |

---

## 十、模块依赖关系

```
┌─────────────────────────────────────────────────────────────┐
│                     外部系统集成                              │
│  ERP ───── WMS ───── 设备系统 ───── 邮件系统                  │
└──────────────────────────┬────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────┐
│                    公共基础服务层                            │
│  WorkOrderService │ WarehouseService │ WorkflowEngine      │
└──────────────────────────┬────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────┐
│                    业务执行层                               │
│  ComWaferTrack │ CpTrack │ FtTrack │ RwTrack │ WltTrack   │
└──────────────────────────┬────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────┐
│                    工艺规划层                               │
│  FmbService │ RouteConfig │ StepDefinition                 │
└────────────────────────────────────────────────────────────┘
```

---

## 相关文档

- [系统概览](../1.MES系统/00-系统概览.md) - 业务知识
- [技术架构](../1.MES系统/01-技术架构.md) - 技术栈详解
- [模块关系图](../1.MES系统/02-模块关系图.md) - 模块依赖
- [API参考](API_REFERENCE.md) - 详细API说明
