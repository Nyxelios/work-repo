# OIS 晶圆接收功能开发

> **Workspace:** d:\data\MES\gc

---

## User
我需要写一个类似晶圆线边仓接收的功能，但是是ois用的，目前写了这些文件app.js 
 oisComChipReceive 
 oisComChipReceive.jsp 
 OisComChipReceiveAction.java 
 oisComChipReceiveBindForm.js 
 oisComChipReceiveBindGrid.js 
 oisComChipReceiveForm.js 
 oisComChipReceiveGrid.js 
 oisComChipReceiveViewer.js 
 struts-config.xml 
 ToolService.java 
 ToolServiceImpl.java 
 WaferReceiveRepository.java 
 WaferReceiveRepositoryImpl.java ，不过我现在不想要oisComChipReceiveBindForm这些关联的代码了，这些是展示待绑的数据，现在不要了，就只要扫描箱号，然后展示数据。再接收，不过扫描的数据不仅要在BACKEND_WAFER_RECEIVE里面存在，如果BACKEND_WAFER_RECEIVE不存在，但是workorderrelation里面存在，且WMS_MMS_MATERIAL_LOT表的记录已经是Issue状态，且workorderid栏位有该workorderrelation的workorderrrn的工单绑定了，就也可以扫描出来，并且扫描后自动创建BACKEND_WAFER_RECEIVE的数据

## Assistant
我来先了解一下现有代码的结构，然后进行修改。让我先查看相关文件。

现在让我搜索这两张表之间的关联查询，以及查看相关的DAO和Service文件中的join查询。

让我进一步搜索在Service层中同时使用WorkOrderRelation和WmsMmsMaterialLot的代码，以及查找是否有SQL直接join这两张表。

让我查看这些关键的join查询代码段。

让我继续查看GcServiceImpl.java中涉及这两张表join的SQL查询。

以下是对代码库中所有与 `oisComChipReceive`（OIS COM芯片接收）相关文件，以及晶圆线边仓接收（waferReceive / waferLswRecieve）参考文件的完整搜索结果。

---

## 一、oisComChipReceive 相关文件

### 1. Java 后端文件

| 文件 | 说明 |
|------|------|
| [OisComChipReceiveAction.java](file:///d:/data/MES/gc/core/src/model/src/com/mycim/webapp/actions/tool/OisComChipReceiveAction.java) | Struts Action 类，处理 OIS COM 芯片接收的三个操作：`search`（查询晶圆列表）、`receive`（接收）、`searchBind`（查询绑定晶圆） |

该 Action 的核心逻辑：
- **search**: 通过 Box ID / Carrier ID 查询 `waferSource = OIS("22")` 的晶圆，状态过滤为 `ENGHOLD / LSWISSUE / LSWRECIEVED`
- **receive**: 校验仅 `WAFER_SOURCE_OIS("22")` 来源的晶圆可接收，且必须有工单信息，调用 `toolService.doWaferReceiveAndEngHold()`
- **searchBind**: 通过工单号调用 `toolService.pageOisComWaferReceive()` 分页查询绑定晶圆

### 2. Service / Repository 层

| 文件 | 说明 |
|------|------|
| [ToolService.java](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/service/ToolService.java#L98) | 接口定义：`pageOisComWaferReceive()` 和 `doWaferReceiveAndEngHold()` |
| [ToolServiceImpl.java](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/service/impl/ToolServiceImpl.java#L6702) | 实现类，委托给 `waferReceiveRepository.pageOisComWaferReceive()` |
| [WaferReceiveRepository.java](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/repository/WaferReceiveRepository.java#L37) | Repository 接口定义 |
| [WaferReceiveRepositoryImpl.java](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/repository/impl/WaferReceiveRepositoryImpl.java#L467) | Repository 实现，核心 SQL 查询 `BACKEND_WAFER_RECEIVE` 表，条件为 `WAFER_SOURCE = '25'` 且 `STATUS IN ('LSWISSUE')` 且 `WORKORDER_ID IS NOT NULL`，关联 `WORKORDER` 表 `PRODUCT_CLASSIFY = '18'` |

### 3. Model 层

| 文件 | 说明 |
|------|------|
| [WaferReceive.java](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/model/WaferReceive.java) | 晶圆接收实体模型，定义了关键常量：`WAFER_SOURCE_OIS = "22"`, `WAFER_SOURCE_16 = "16"`(COB), `STATUS_LSWISSUE`, `STATUS_ENGHOLD`, `STATUS_LSWRECIEVED` 等 |

### 4. 前端文件（JSP / JS）

| 文件 | 说明 |
|------|------|
| [oisComChipReceive.jsp](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/oisComChipReceive/oisComChipReceive.jsp) | JSP 页面，标题为 "OIS COM芯片接收"，引入所有 JS 文件 |
| [oisComChipReceiveViewer.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/oisComChipReceive/oisComChipReceiveViewer.js) | 主视图组件，包含上下两个区域：上部为查询/接收表单+Grid，下部为绑定查询表单+Grid |
| [oisComChipReceiveForm.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/oisComChipReceive/oisComChipReceiveForm.js) | 查询表单，含 Box ID、Carrier ID 输入框和查询/重置/接收按钮，请求 `/mycim2/oisComChipReceive.do` |
| [oisComChipReceiveGrid.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/oisComChipReceive/oisComChipReceiveGrid.js) | 晶圆列表 Grid，含 workorderId 列，Store 请求 `action=search` |
| [oisComChipReceiveBindForm.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/oisComChipReceive/oisComChipReceiveBindForm.js) | 绑定查询表单，含工单号输入框，回车触发 `Ext.doSearchBind` |
| [oisComChipReceiveBindGrid.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/oisComChipReceive/oisComChipReceiveBindGrid.js) | 绑定晶圆 Grid，Store 请求 `action=searchBind`，支持分页 |
| [app.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/oisComChipReceive/app.js) | 入口文件，创建 `OisComChipReceiveViewer.App` 实例 |

### 5. Struts 配置

在 [struts-config.xml](file:///d:/data/MES/gc/core/web/WEB-INF/struts-config.xml#L5808) 中的配置：

```xml
<action path="/oisComChipReceive" type="com.mycim.webapp.actions.tool.OisComChipReceiveAction" scope="request">
    <forward name="modify" path="/app/tools/waferReceive/oisComChipReceive/oisComChipReceive.jsp"/>
</action>
```

---

## 二、晶圆线边仓接收参考文件（waferReceive / waferLswRecieve）

### 1. waferLswRecieve（晶圆线边仓接收 - 最直接的参考）

| 文件 | 说明 |
|------|------|
| [WaferLswRecieveAction.java](file:///d:/data/MES/gc/core/src/model/src/com/mycim/webapp/actions/tool/WaferLswRecieveAction.java) | Struts Action，与 OisComChipReceiveAction 结构几乎一致，操作为 `search`、`lswRecieve`、`searchBind` |
| [waferLswRecieve.jsp](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/waferLswRecieve/waferLswRecieve.jsp) | JSP 页面 |
| [waferLswRecieveViewer.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/waferLswRecieve/waferLswRecieveViewer.js) | 主视图组件 |
| [waferLswRecieveForm.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/waferLswRecieve/waferLswRecieveForm.js) | 查询表单 |
| [waferLswRecieveGrid.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/waferLswRecieve/waferLswRecieveGrid.js) | 晶圆列表 Grid |
| [waferLswRecieveBindForm.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/waferLswRecieve/waferLswRecieveBindForm.js) | 绑定查询表单 |
| [waferLswRecieveBindGrid.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/waferLswRecieve/waferLswRecieveBindGrid.js) | 绑定晶圆 Grid |

### 2. waferReceive（晶圆接收基础功能）

| 文件 | 说明 |
|------|------|
| [WaferReceiveAction.java](file:///d:/data/MES/gc/core/src/model/src/com/mycim/webapp/actions/tool/WaferReceiveAction.java) | 基础晶圆接收 Action |
| [waferReceiveViewer.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/waferReceive/waferReceiveViewer.js) | 主视图组件 |
| [waferReceiveForm.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/waferReceive/waferReceiveForm.js) | 查询表单，含 Box ID、删除备注、查询/重置/接收/删除按钮 |
| [waferReceiveGrid.js](file:///d:/data/MES/gc/core/web/app/tools/waferReceive/waferReceive/waferReceiveGrid.js) | 晶圆列表 Grid |

---

## 三、BACKEND_WAFER_RECEIVE 相关文件

`BACKEND_WAFER_RECEIVE` 是数据库表名，出现在以下关键文件中：

| 文件 | 说明 |
|------|------|
| [WaferReceiveRepositoryImpl.java](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/repository/impl/WaferReceiveRepositoryImpl.java#L467) | `pageOisComWaferReceive()` 方法中查询 `BACKEND_WAFER_RECEIVE` 表，条件 `WAFER_SOURCE = '25'`, `STATUS IN ('LSWISSUE')`, 关联 `WORKORDER` 表 `PRODUCT_CLASSIFY = '18'` |
| [WaferReceive.java](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/model/WaferReceive.java) | 对应 `BACKEND_WAFER_RECEIVE` 表的实体模型 |
| [WaferReceiveHistory.java](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/model/WaferReceiveHistory.java) | 对应历史表 |
| [WaferReceiveHistoryRepositoryImpl.java](file:///d:/data/MES/gc/core/src/tool/src/main/java/com/mycim/tool/repository/impl/WaferReceiveHistoryRepositoryImpl.java) | 历史表 Repository |
| [WaferReceivePrp.java](file:///d:/data/MES/gc/core/src/valueobject/src/com/mycim/prp/model/WaferReceivePrp.java) | PRP 模型 |
| [WaferReceiveHisInfo.java](file:///d:/data/MES/gc/core/src/gc/src/main/java/com/mycim/gc/dto/WaferReceiveHisInfo.java) | 历史信息 DTO |
| [typeDefinitions.jsp](file:///d:/data/MES/gc/core/web/typeDefinitions.jsp) | 前端类型定义 |

此外还有大量 Liquibase 数据库变更文件（约 40+ 个 yaml 文件），位于：
- `d:\data\MES\gc\core\web\WEB-INF\config\db\changelog\` 目录下的 `db.changelog-*.waferReceive*.yaml`
- `d:\data\MES\gc\core\web\WEB-INF\config\db\changelog\backend\` 目录下的 `db.changelog-*.waferReceive*.yaml`

---

## 四、关键对比总结

| 对比项 | OisComChipReceive | WaferLswRecieve |
|--------|-------------------|-----------------|
| Action URL | `/oisComChipReceive.do` | `/waferLswRecieve.do` |
| 晶圆来源 | `WAFER_SOURCE_OIS = "22"` | `WAFER_SOURCE_16 = "16"` (COB) |
| 接收操作名 | `action=receive` | `action=lswRecieve` |
| 绑定查询 | `pageOisComWaferReceive()` - 查 BACKEND_WAFER_RECEIVE 表，WAFER_SOURCE='25', PRODUCT_CLASSIFY='18' | `pageWaferReceive()` - 查 BACKEND_WAFER_RECEIVE 表，PRODUCT_CLASSIFY='4' |
| 前端结构 | 上下两个区域（查询Grid + 绑定Grid） | 上下两个区域（查询Grid + 绑定Grid），结构一致 |
| 接收校验 | 仅OIS来源，必须有工单 | COB来源，必须有工单 |

**注意**：`pageOisComWaferReceive` 中的 SQL 查询条件 `WAFER_SOURCE = '25'`，而 Action 中校验的是 `WAFER_SOURCE_OIS = "22"`，这两个值不一致，可能需要注意区分。
