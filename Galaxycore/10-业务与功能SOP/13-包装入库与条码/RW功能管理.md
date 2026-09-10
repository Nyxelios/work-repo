
# RW入库位核验

## 1. 功能概述

RW入库位核验功能对应页面路径 `mycim2/wip/rwAssayNoLocation`，用于按 Assay Lot 维度查看批次信息，并辅助维护库位。  
页面虽然目录名中带有 `NoLocation`，但实际不只是查看无库位数据，还包含查询、回显已有库位、补录库位、合箱候选标记以及导出 Excel 等能力。

页面入口配置：

- Struts 路径：`/rwAssayNoLocation`
- 转发页面：`/wip/rwAssayNoLocation/rwBoxLocation.jsp`
- 后端 Action：`com.mycim.webapp.actions.prp.CpLocationConfAction`

## 2. 页面说明

页面由查询区和结果列表区组成。

### 2.1 页面入口

页面入口配置：

- Struts 路径：`/rwAssayNoLocation`
- 转发页面：`/wip/rwAssayNoLocation/rwBoxLocation.jsp`
- 后端 Action：`com.mycim.webapp.actions.prp.CpLocationConfAction`

### 2.2 查询条件

查询条件包括：

- Box ID
- Product Model
- Location
- Grade

### 2.3 页面按钮

页面按钮包括：

- `See`：查询 Assay Lot 列表
- `Merge Box`：对勾选记录进行合箱候选标记
- `Add`：为指定 Lot 补录库位
- `Export Excel`：导出当前查询结果

### 2.4 展示字段

结果列表主要展示以下信息：

- 分组号 `seqence`
- 库位 `location`
- 在线天数 `onlineTime`
- 工单 `outerOrderNO`
- 工序 `operationId`
- 产品型号 `productId`
- 客户批号 `lotCst`
- 内部批号 `lotId`
- 二级代码 `levelTwoCode`
- 等级 `binGrade`
- 保税属性 `bondedProperty`
- 流程 `processId`
- 片数 `qty1`
- 数量 `qty2`

## 3. 业务规则

### 3.1 查询规则

前端查询调用：

- `/mycim2/cpLocationConf.do?action=showAyLot`

后端处理逻辑如下：

1. 根据输入的 `lotId`、`grade` 调用 `lotServiceInterface.getLocationList` 获取候选 Lot 列表。
2. 再按 `productId`、`grade` 进行二次过滤。
3. 将每条 Lot 的 `buildRouteId` 回填到 `processId` 字段。
4. 查询该 Lot 是否已存在库位配置，如已配置则回填 `location`。
5. 根据主批的 RIQC 创建时间计算在线天数 `onlineTime`。
6. 按业务字段变化对结果自动分组，并生成分组号 `seqence`。
7. 如果输入的 `lotId` 命中某个分组，则最终只返回该分组的数据。

这里需要特别注意，页面不是简单返回所有符合条件的 Lot，而是会在识别出目标 Lot 所属分组后，仅返回该组数据。因此该页面更适合用于查看某个 Lot 所属的同组候选批次。

### 3.2 分组规则

相邻记录只要以下任一字段发生变化，就会切换到下一组：

- `productId`
- `levelTwoCode`
- `binGrade`
- `bondedProperty`
- `outerOrderNO`
- `buildRouteId`
- `operationId`
- `lotId` 最后一个 `.` 后缀

系统通过该分组结果生成页面中的 `seqence` 字段，后续合箱候选判断也依赖该分组号。

### 3.3 库位补录规则

前端 `Add` 按钮调用：

- `/mycim2/cpLocationConf.do?action=saveWtwBox`

后端处理逻辑如下：

1. 根据输入的 `lotId` 查询 Lot。
2. 如果该 Lot 已存在库位配置，则先删除旧配置。
3. 重新组装一条 `CpLocationConf` 数据。
4. 保存字段包括：
   - `location`
   - `productClassify`
   - `productId`
   - `lotId`
   - `pieces`
   - `bondedProperty`
   - `workOrderId`
5. 保存成功后，前端自动重新执行查询。

因此该页面中的新增，本质上是“按 Lot 重建库位配置”，不是对同一 Lot 进行多条追加。

### 3.4 合箱候选规则

前端 `Merge Box` 按钮只做页面标记，不调用后端合箱接口，也不会写数据库。

标记规则如下：

1. 先按 `seqence` 对当前勾选记录分组。
2. 检查同组内所有 `lotId` 的后缀是否一致。
3. 只有当同组勾选记录数大于等于 2 时，该组才算有效。
4. 满足条件的记录会被设置 `status = Y`。
5. 页面通过绿色高亮显示这些记录。

因此该按钮的作用更接近“可合箱候选提示”，而不是真正执行合箱。

### 3.5 导出规则

`Export Excel` 按钮会将当前 Grid 中已经展示的数据直接组装后提交到：

- `/mycim2/exportExcel.do?action=exportToExcel`

导出的内容以当前页面列表为准，不会再次单独向业务查询接口取数。

## 4. 接口说明

页面主入口：

- `/rwAssayNoLocation`

页面查询接口：

- `/mycim2/cpLocationConf.do?action=showAyLot`

库位补录接口：

- `/mycim2/cpLocationConf.do?action=saveWtwBox`

导出接口：

- `/mycim2/exportExcel.do?action=exportToExcel`

## 5. 注意事项

- 页面目录名为 `rwAssayNoLocation`，但实际会显示和维护 `location`，名称与功能并不完全一致。
- 前端入口是 `/rwAssayNoLocation`，但页面中的实际数据请求统一走 `/cpLocationConf.do`。
- `Add` 操作会先删除该 Lot 已有的库位配置，再重新保存。
- `Merge Box` 只是前端标色提示，不代表已完成实际合箱。
- 查询逻辑存在“命中某分组后只返回该分组”的特殊行为，排查结果数量异常时需优先关注这一点。

---

## 5. 详细更新记录

### [2026-09-08] 内批+库位添加自动清空与全部清空功能

- **需求人**：龚钱
- **需求 / 背景**：
  - 在真空包/物料库位配置（`rwAssayNoLocation`）页面中，优化操作录入体验：
    1. 输入内批 + 库位后点击“添加”，保存成功自动清空内批和库位输入值，光标自动重聚焦到内批输入框，便于连续扫码/录入；
    2. 新增“全部清空”按钮，便于一键重置所有筛选表单项与表格列表。
- **核心代码改动**：
  - `core/web/wip/rwAssayNoLocation/rwBoxLocationForm.js`：
    - `Ext.doCreate`：在 `saveWtwBox` 成功回调中增加 `lotId`、`location` 置空，并调用 `focus(false, 100)` 自动聚焦回内批输入框。
    - `buttons`：在操作按钮栏新增 `{ text: '全部清空', handler: Ext.doClearAll }`。
    - `Ext.doClearAll`：重置表单所有输入项（内批、产品型号、库位、等级、工步站点），清空 `RwBoxLocationStore` 列表数据与缓存参数，并将焦点恢复至内批。
- **数据库变动 (SQL)**：
  ```sql
  -- 本次无数据库表结构变动
  ```
- **配置与部署注意**：
  - 前端静态 JS 改动，部署或刷新浏览器缓存后即可生效。
- **自测情况**：
  - 录入内批和库位点击“添加”，保存成功后输入项自动置空且光标停留在内批输入框；
  - 点击“全部清空”，表单各项与表格内数据均已成功清空。

