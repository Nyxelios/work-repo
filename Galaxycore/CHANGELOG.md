# 个人功能开发与更新日志 (CHANGELOG)

> 💡 **使用提示**：
> - 建议每次完成新需求、接口改造或 Bug 修复后随手记录一笔。
> - 记录需求人、改动原因、核心代码类/方法、SQL变更，方便日后问题回溯与上线部署。
> - 底部附有 [空白模板](#-新增记录模板直接复制)，每次新增直接复制粘贴即可。

---

## 📅 更新总览 (快速索引)

| 更新日期 | 功能 / 任务名称 | 需求人 | 涉及模块 | 状态 |
| :--- | :--- | :--- | :--- | :--- |
| 2026-09-09 | 工单物料绑定关系抽取与工厂/等级校验Bug修复 | - | WIP / 工单模块 (WorkOrderProduce) | ✅ 已完成 |
| 2026-09-08 | 内批+库位添加自动清空与全部清空功能 | 龚钱 | WIP / rwAssayNoLocation | ✅ 已完成 |
| 2026-09-08 | 销售料号自动填充功能 | - | WIP / 工单模块 | ✅ 已完成 |
| 2026-09-08 | 设备物料出库校验与扣料优化 | - | WIP / 设备物料 | ✅ 已完成 |

---

## 📝 详细更新记录

### [2026-09-09] 工单物料绑定关系抽取与工厂/等级校验Bug修复
- **需求人**：-
- **需求 / 背景**：
  - 工单排产绑定物料流程（`WorkOrderProduce`）中，存在 CP 预入 RW 工单排料后再次校验误报“不同工厂不能排一个工单”以及数据关系组装逻辑分散的问题：
    1. **修复校验拦截 Bug**：当工单已绑定 CP 预入 RW 物料（物料存在于 WIP 的 `PACKED_LOT` 打包批次中，而非 WMS 仓库表）时，原逻辑因只查第 0 条且仅查 WMS 导致丢失已有物料的工厂与等级，后续追加物料校验时抛出“不同工厂不可排在一个工单”异常；
    2. **抽取公共关系方法**：在 `WorkOrderRelation` 与 `WorkOrderService` 中抽象封装 `vbox`（箱级）与 `unit`（片级）的构建、卡控校验与入库逻辑，避免在 `WaferBindSyncServiceImpl` 中重复散落关系组装代码；
    3. **规范字段存储**：CP 预入 RW 的 `lotId` 规范存储为晶圆原始批号/CST号（`lotPlan.getAttributeData8()`），以便后续回退逆向准确查询打包数据；补全排料时间 `createTime` 等字段。
- **核心代码改动**：
  - `WorkOrderRelation.java`：
    - 新增 `vbox(materialLot, workOrder)` 与 `unit(unit, workOrder)` 方法，标准化字段赋值（`objectId`、`objectType`、`materialLotId`、`mlotRrn`、`lotId` 及数量计算）；
    - 在 `WorkOrderRelation(WorkOrder, LotPlan, String)` 构造函数中将 `lotId` 由 MES 批号修正为原始批号 `lotPlan.getAttributeData8()`；
    - 统一通过 `generatorBasic(workOrder)` 初始化上下文主键、数量与 `createTime = new Date()`。
  - `WorkOrderService.java` / `WorkOrderServiceImpl.java`：
    - 接口与实现类中新增 `vboxWorkOrderRelation(materialLot, workOrder)` 与 `unitWorkOrderRelation(unit, workOrder)`，将累加投入量（`planPutInto`）、工单超额卡控与历史表记录保存统一收拢到服务层。
  - `WaferBindSyncServiceImpl.java`：
    - `getToBindWmsMaterialLots`：由只看第 0 条改为遍历所有已有关系（修正为当前 `relation.getLotId()`），并在 WMS 仓库表查不到时回退查询 `packageService.getPackedLotByCstId(relation.getLotId())`，准确提取工单已有物料的 `grade` 和 `factoryId`；
    - `bindWorkOrderVbox` 与 `planWorkOrderUnit`（含 `planComMaterial`）：统一调用服务层 `vboxWorkOrderRelation` / `unitWorkOrderRelation`；
    - `generatorLotAndWorkOrderRelation`：补全 `createTime` 设置，避免前端排料时间显示为空。
- **数据库变动 (SQL)**：
  ```sql
  -- 本次无数据库表结构变更
  ```
- **配置与部署注意**：
  - 涉及后端核心业务编译（`core/prp`、`core/gc`、`core/tool`），构建更新后重启部署即可生效。
- **自测情况**：
  - CP 预入 RW 工单排料后进行二次追加绑料，正常识别原批次的工厂地点与等级，不再误报“不同工厂不可排在一个工单”；
  - 箱级（COM / Retest VBox）与片级（Unit）绑料流程正常，计划投料量累加及超额卡控校验通过，`WORK_ORDER_RELATION` 及历史表正常生成。

---

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

---

### [2026-09-08] 设备物料出库校验与扣料优化
- **需求人**：-
- **需求 / 背景**：
  - 针对设备物料出站扣料场景，完善工步与 BOM 一致性校验，规范 IRA 批号绑定和解绑逻辑。
- **核心代码改动**：
  - `EquipmentMaterialController.java`
    - `execute()` (`/outIraLot` 接口)：完善多批次 BOM 与工步一致性检查，补充扣料明细返回。
    - `iraBind()` (`/iraBind` 接口)：增加 IRA 烘烤历史校验及上料/下料状态流转。
    - `checkRunningLotBom()`：机台运行批次物料有效性与失效周期检查。
- **数据库变动 (SQL)**：
  ```sql
  -- 本次无表结构变更（如有时在此记录 DDL / DML）
  ```
- **配置与部署注意**：
  - 依赖默认厂区配置（`DEFAULT_FACILITY_ID = "GC"`）及默认用户。
- **自测情况**：
  - 多批次合并出站扣料逻辑校验通过；
  - 未烘烤 IRA 上料时正常拦截并提示。

---

### [2026-09-08] 销售料号自动填充功能
- **需求人**：-
- **需求 / 背景**：
  - 在相关业务流程中，依据当前物料或产品信息，自动推导并填充对应的销售料号，避免手工填写的失误。
- **核心代码改动**：
  - 完善对应 Service / Controller 中的物料映射与销售料号推导逻辑。
  - 前端界面数据绑定与回填联动。
- **自测情况**：
  - 验证下单/工单录入流程，销售料号能根据选定物料自动且正确带出。

---

## 📋 新增记录模板（直接复制）

```markdown
### [YYYY-MM-DD] 功能 / 需求 / Bug 修复名称
- **需求人**：提出人姓名
- **需求 / 背景**：简要说明为什么做此改动，解决什么问题（或关联 JIRA/TAPD 需求单号）。
- **核心代码改动**：
  - `xxxController.java`：
    - `methodName()`：修改说明 / 新增接口
  - `xxxService.java`：
    - `methodName()`：核心逻辑改动
  - 前端 / XML 配置等：
- **数据库变动 (SQL)**：
  ```sql
  -- 如无变更写“无”；有变更则贴具体 DDL/DML
  ```
- **配置与部署注意**：
  - 是否有新增配置项、系统参数或需刷新的缓存。
- **自测情况**：
  - 核心场景测试结果（正常/异常用例）。
```
