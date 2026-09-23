---
title: BOM程序Recipe配置与版本校验规则
date: 2026-09-10
author: MES Team
requester: 黄志平
status: completed
tags: [BOM管理, Recipe配置, PDCU, AU, 校验规则, PRP, SOP]
---

# BOM程序Recipe配置与版本校验规则

## 1. 业务背景与概述

在半导体芯片封装与测试（Packaging & Testing / FT）生产中，不同产品（BOM）、不同工艺阶段版本（BOM Version）、各道工序（Operation）以及特定工单对应的机台程序（Recipe）存在着严格的映射约束。

为规范机台 Recipe 程序的集中维护与生命周期管理，系统提供了 **Recipe程序配置（Recipe Program Configuration）** 功能模块（`/mycim2/recipeProgramConfig.do`），统一管理产品各站点所使用的机台主程序（`Recipe程序`）与串联设备从程序（`Inline Recipe程序`）。

在键合/打线（Wire Bonding）等关键工艺制程中，不同线材工艺的程序选型存在严格的互斥与卡控规范：
1. **测试/试产工艺（T 开头 BOM）**：研发测试、工程打样或特定试产工艺（如镀钯铜丝 PdCu 键合工艺）其 BOM 版本以 **`T`**（或小写 `t`）开头，要求所配置的机台主程序及 inline 程序中必须包含关键字 **`PDCU`**；
2. **常规量产工艺（非 T 开头 BOM）**：常规量产工艺（如标准金线 Au 键合工艺），其 BOM 版本不以 T 开头，要求所配置的机台主程序及 inline 程序中必须包含关键字 **`AU`**。

为防范因程序选型或手工录入失误导致的混线、调错程序等严重工艺事故，系统建立了强校验卡控机制。

---

## 2. 数据结构与表设计

Recipe 程序配置在数据库中采用主表与历史轨迹表成对记录设计：

### 2.1 核心数据表

| 表名 | 实体类名 | 说明 |
| :--- | :--- | :--- |
| `GC_RECIPE_PROGRAM_CONFIG` | `RecipeProgramConfig` | 存放当前有效的 Recipe 程序配置主记录 |
| `GC_RECIPE_PROGRAM_CONFIG_H` | `RecipeProgramConfigH` | 记录配置的生命周期历史变更（CREATE / MODIFY / DELETE） |

### 2.2 关键字段说明

| 字段名称 | 数据库列名 | 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| 主键 | `OBJECT_RRN` | NUMBER(15) | 系统唯一标识 RRN |
| BOM编号 | `BOM_ID` | VARCHAR2(64) | 归属的 BOM 物料编号，必填 |
| BOM版本 | `BOM_VERSION` | VARCHAR2(64) | BOM 版本号（如 `01`, `V1.0`, `T01`），必填 |
| 工步号 | `OPERATION_ID` | VARCHAR2(64) | 适用工步/站点代号，必填 |
| 配置类型 | `CONFIG_TYPE` | VARCHAR2(3) | `PVT`（小批量试产）或 `MP`（量产），必填 |
| 工单号 | `WORK_ORDER_ID` | VARCHAR2(64) | 归属工单号；`PVT` 模式必填，`MP` 模式固定为空 |
| Recipe程序 | `RECIPE_PROGRAM` | VARCHAR2(500) | 机台主程序名 |
| Inline Recipe程序 | `RECIPE_PROGRAM_INLINE` | VARCHAR2(500) | 串联设备/联机辅助程序名 |
| 创建人 / 时间 | `CREATED_BY` / `CREATED_DATE` | VARCHAR2 / DATE | 记录创建信息 |
| 修改人 / 时间 | `UPDATED_BY` / `UPDATED_DATE` | VARCHAR2 / DATE | 记录最后更新信息 |

---

## 3. 功能操作指南

Recipe 程序配置前端界面位于 `/mycim2/recipeProgramConfig.do`，提供完整的单条维护与批量处理能力：

### 3.1 查询与检索
- 支持按 **配置类型（PVT / MP）**、**BOM编号**、**BOM版本**、**工步号**、**工单号**、**Recipe程序**、**Inline Recipe程序** 多维度组合筛选；
- 支持重置筛选条件并刷新表格数据。

### 3.2 新增与编辑配置
1. 点击工具栏 **【新增】** 或选中一条记录点击 **【编辑】** 弹出配置维护窗口；
2. 录入 BOM 编号、BOM 版本、工步号，选择配置类型（PVT / MP）：
   - **MP（量产）模式**：工单号字段自动置空并禁用；
   - **PVT（试产）模式**：工单号必须填写；同时支持勾选 **【是否抓取量产程序】**，勾选后后台将自动获取该 BOM 对应工步已有的 MP 配置程序名填入；
3. 录入 `Recipe程序` 与 `Inline Recipe程序`；
4. 点击 **【保存】**，系统执行业务规则校验后入库并记录历史表。

### 3.3 批量修改程序
1. 在表格中通过复选框勾选多条需要批量替换程序名的配置记录；
2. 点击工具栏 **【批量修改程序】**；
3. 在弹窗中输入新的 `Recipe程序` 和/或 `Inline Recipe程序`（留空表示保持原值不变）；
4. 确认提交后，系统将统一执行前置校验并批量更新选中的全部记录。

### 3.4 删除配置
- 选中一条或多条配置记录，点击 **【删除】**，二次确认后执行物理删除并写入历史变更表（`DELETE` 类型事务）。

### 3.5 LOT测试程序配置联动抓取量产Recipe（需求人：刘姣）
在【LOT测试程序配置】（`/mycim2/lotTestPragramConfig.do`）页面添加工单测试程序时：
1. 勾选 **【抓取量产Recipe】** 按钮；
2. 系统自动解析当前工单型号（去除最后一个 `-` 及其后尾缀，保留主型号）作为 `BOM_ID`，工单自身的 `bomVersion` 作为版本号；
3. 查询本表 `GC_RECIPE_PROGRAM_CONFIG` 中对应 `CONFIG_TYPE = 'MP'` 的量产程序；
4. 联动更新/新增本表中当前工单的 `PVT` 记录（工步号同步量产工步号，程序名同步量产程序），并同步记录 `GC_RECIPE_PROGRAM_CONFIG_H` 历史表。详细操作规范参见交付文档：[LOT测试程序配置增加抓取量产Recipe功能](../../30-任务工作流与追踪/32-已交付需求(Done)/2026%20Q3/LOT测试程序配置增加抓取量产Recipe功能.md)。

---


## 4. 程序名与 BOM 版本校验卡控规则

### 4.1 校验规则矩阵

系统依据 BOM 版本前缀自动分为两组互斥校验规则（均不区分大小写，支持如 `PdCu`/`pdcu` 与 `Au`/`au`）：

| BOM 版本类型 | 判定条件 | Recipe程序（非空时） | Inline Recipe程序（非空时） | 校验未通过时提示 |
| :--- | :--- | :--- | :--- | :--- |
| **T 开头版本** | 以 `T` 或 `t` 开头（如 `T01`, `t1.0`） | 必须包含 **`PDCU`** | 必须包含 **`PDCU`** | BOM版本为T开头时，[Recipe/Inline]程序必须包含PDCU |
| **非 T 开头版本** | 不以 `T`/`t` 开头（如 `01`, `V1.0`） | 必须包含 **`AU`** | 必须包含 **`AU`** | BOM版本非T开头时，[Recipe/Inline]程序必须包含AU |

> 📌 **留空项处理**：若某工序为单机台、无串联设备，允许单项留空，但两者仍受“不能同时为空”的基础规则约束。未填写的留空项不触发关键字强制要求。

### 4.2 校验拦截场景示例矩阵

| 场景序号 | BOM版本 | Recipe程序 | Inline Recipe程序 | 校验结果 | 说明 |
| :---: | :---: | :---: | :---: | :---: | :--- |
| 1 | `T01` | `WB_PDCU_01` | *(留空)* | ✅ **允许保存** | T版本，主程序含 PDCU |
| 2 | `T01` | `WB_PdCu_01` | `INLINE_PDCU_01` | ✅ **允许保存** | T版本，两者均含 PDCU，兼容大小写 |
| 3 | `T01` | `WB_AU_01` | *(留空)* | ❌ **拦截报错** | T版本，Recipe程序未包含 PDCU |
| 4 | `T01` | `WB_PDCU_01` | `INLINE_AU_01` | ❌ **拦截报错** | T版本，Inline程序未包含 PDCU |
| 5 | `01` | `WB_AU_01` | *(留空)* | ✅ **允许保存** | 非T版本，主程序含 AU |
| 6 | `V1.0` | `WB_Au_01` | `INLINE_AU_01` | ✅ **允许保存** | 非T版本，两者均含 AU |
| 7 | `01` | `WB_PDCU_01` | *(留空)* | ❌ **拦截报错** | 非T版本，Recipe程序未包含 AU |
| 8 | `01` | `WB_AU_01` | `INLINE_TEST_01` | ❌ **拦截报错** | 非T版本，Inline程序未包含 AU |

---

## 5. 核心代码实现

### 5.1 前端交互拦截 (`recipeProgramConfig.js`)
在编辑保存 `saveEditor` 及批量修改 `updatePrograms` 中，在发起 Ajax 请求前执行前置卡控：
```javascript
function isTBomVersion(bomVersion) {
    if (Ext.isEmpty(bomVersion)) return false;
    var trimmed = Ext.String.trim(String(bomVersion));
    return trimmed.charAt(0) === 'T' || trimmed.charAt(0) === 't';
}

function containsKeyword(str, keyword) {
    return !Ext.isEmpty(str) && !Ext.isEmpty(keyword)
            && String(str).toUpperCase().indexOf(String(keyword).toUpperCase()) !== -1;
}

// saveEditor 保存前校验
if (isTBomVersion(values.bomVersion)) {
    if (!Ext.isEmpty(values.recipeProgram) && !containsKeyword(values.recipeProgram, 'PDCU')) {
        MyCim.notify.alert('BOM版本为T开头时，Recipe程序必须包含PDCU');
        return;
    }
    if (!Ext.isEmpty(values.recipeProgramInline) && !containsKeyword(values.recipeProgramInline, 'PDCU')) {
        MyCim.notify.alert('BOM版本为T开头时，Inline Recipe程序必须包含PDCU');
        return;
    }
} else {
    if (!Ext.isEmpty(values.recipeProgram) && !containsKeyword(values.recipeProgram, 'AU')) {
        MyCim.notify.alert('BOM版本非T开头时，Recipe程序必须包含AU');
        return;
    }
    if (!Ext.isEmpty(values.recipeProgramInline) && !containsKeyword(values.recipeProgramInline, 'AU')) {
        MyCim.notify.alert('BOM版本非T开头时，Inline Recipe程序必须包含AU');
        return;
    }
}
```

### 5.2 后端服务层强校验 (`PrpSetupServiceImpl.java`)
后端通过 `ensureRecipeProgramConfigValues` 与 `batchUpdateRecipeProgramConfigs` 提供兜底保障：
```java
private boolean isTBomVersion(String bomVersion) {
    if (StringUtils.isBlank(bomVersion)) return false;
    String trimmed = bomVersion.trim();
    return trimmed.startsWith("T") || trimmed.startsWith("t");
}

private boolean containsKeyword(String programName, String keyword) {
    return programName != null && keyword != null
            && programName.toUpperCase().contains(keyword.toUpperCase());
}

private void validateRecipeProgramsByBomVersion(String bomVersion, String recipeProgram, String recipeProgramInline) {
    if (StringUtils.isBlank(bomVersion)) return;
    if (isTBomVersion(bomVersion)) {
        if (StringUtils.isNotBlank(recipeProgram) && !containsKeyword(recipeProgram, "PDCU")) {
            throw new MyCimParameterException("BOM版本为T开头时，程序名必须包含PDCU");
        }
        if (StringUtils.isNotBlank(recipeProgramInline) && !containsKeyword(recipeProgramInline, "PDCU")) {
            throw new MyCimParameterException("BOM版本为T开头时，inline程序名必须包含PDCU");
        }
    } else {
        if (StringUtils.isNotBlank(recipeProgram) && !containsKeyword(recipeProgram, "AU")) {
            throw new MyCimParameterException("BOM版本非T开头时，程序名必须包含AU");
        }
        if (StringUtils.isNotBlank(recipeProgramInline) && !containsKeyword(recipeProgramInline, "AU")) {
            throw new MyCimParameterException("BOM版本非T开头时，inline程序名必须包含AU");
        }
    }
}
```

---

## 6. 常见问题与排查指南

### Q1: 为什么批量修改程序时提示“选中的配置中包含非T开头的BOM版本，Recipe程序必须包含AU”？
- **原因**：批量选中的记录中包含非 T 开头的量产 BOM 配置，但输入的替换程序名未包含 `AU`；
- **建议**：如需同时修改多种版本，请将量产（非T）与试产（T）记录分批选中后分别更新。

### Q2: 某些单机台工序没有 Inline 程序，是否可以只填 Recipe 程序？
- **可以**。系统规则为“至少填写一项”。若工序无 inline 串联设备，只需在 `Recipe程序` 中填写符合规则的程序名，`Inline Recipe程序` 保持留空即可顺利保存。

---

## 7. 详细更新记录

### [2026-09-10] BOM程序Recipe配置增加T开头PDCU及非T开头AU校验限制

- **需求人**：黄志平
- **所属代码分支**：`feature/bomRecipeConfig`（已合并至 `env/pirun`）
- **核心提交记录**：
  - `18bde2b7`：T*版本程序名必须包含PDCU
  - `dc052dff`：非T*版本程序名必须包含AU
  - `f28b92d2`：Merge branch 'feature/bomRecipeConfig' into env/pirun
- **需求 / 背景**：
  - 在 BOM Recipe 程序配置（`recipeProgramConfig`）业务中，规范不同工艺类型的机台程序名命名标准；
  - 业务规定：
    1. 添加或维护以 `T` 开头的 BOM 版本时，配置的 `Recipe程序` 与 `Inline Recipe程序` 必须包含关键字 **`PDCU`**（防范试产铜线工艺用错程序）；
    2. 添加或维护非 `T` 开头的 BOM 版本时，配置的 `Recipe程序` 与 `Inline Recipe程序` 必须包含关键字 **`AU`**（规范量产金线工艺程序命名）；
  - 需在前端界面交互弹窗（新增、编辑、批量修改程序）与后端核心服务层进行统一强校验卡控，覆盖全部配置维护场景。
- **核心代码改动**：
  - `PrpSetupServiceImpl.java`：
    - 新增通用辅助方法 `containsKeyword(String programName, String keyword)` 与分流校验方法 `validateRecipeProgramsByBomVersion(bomVersion, recipeProgram, recipeProgramInline)`；
    - 在 `ensureRecipeProgramConfigValues`（单条保存/更新）与 `batchUpdateRecipeProgramConfigs`（批量修改）中无条件对全部版本调用规则校验；
  - `recipeProgramConfig.js`：
    - 新增通用 `containsKeyword(str, keyword)` 前端校验函数；
    - 在配置编辑保存窗口（`saveEditor`）与批量修改窗口（`updatePrograms`）中增加 T 开头（PDCU）与非 T 开头（AU）的前置拦截弹窗提示。
- **数据库变动 (SQL)**：
  ```sql
  -- 本次无数据库表结构变更
  ```
- **配置与部署注意**：
  - 重新编译打包 `prp` 核心模块：执行 `build.cmd jar.prp` 成功生成 `prpClient.jar`；
  - 前端静态 JS（`recipeProgramConfig.js`）部署或刷新浏览器缓存后生效。
- **自测情况**：
  - 编译构建验证：执行 `build.cmd jar.prp` 编译通过（`BUILD SUCCESSFUL`）；
  - 单条新增/编辑：
    - T01 版本配置包含 PDCU 正常保存，不含 PDCU 弹窗拦截；
    - 01 版本配置包含 AU 正常保存，不含 AU 弹窗拦截；
  - 批量修改场景：针对选中的记录分别校验对应版本的关键字；
  - 留空项逻辑：单机台工序仅维护主程序名时，未填写的 inline 程序名不强制要求输入。

### [2026-09-20] LOT测试程序配置增加抓取量产Recipe功能

- **需求人**：刘姣
- **所属模块**：WIP / PRP
- **相关页面**：`/mycim2/lotTestPragramConfig.do`（`lotTestPragramConfigForm.js`）
- **需求 / 背景**：
  - 在 LOT 测试程序配置中，新增工单时提供【抓取量产Recipe】复选框；
  - 勾选后，系统解析工单型号（去掉最后一个 `-` 及其后尾缀）作为 BOM 编号，并读取工单自身的 BOM 版本，查询 `GC_RECIPE_PROGRAM_CONFIG` 表中的 MP 量产程序（`CONFIG_TYPE = 'MP'`）；
  - 查询当前工单已有的 PVT 记录（`CONFIG_TYPE = 'PVT'`），存在则按量产程序更新主程序与从程序名并记历史（`MODIFY`），不存在则同步量产工步号并新增 PVT 记录与记历史（`CREATE`）；未找到量产程序时进行阻断校验提示；未勾选则保持原逻辑不变。
- **核心代码改动**：
  - `lotTestPragramConfigForm.js`：添加复选框并在 Ajax 请求中传递 `syncRecipeProgram`；
  - `LotTestPragramConfig.java`：增加 `@Transient private Boolean syncRecipeProgram`；
  - `LotTestPragramConfigAction.java`：解析参数并调用重载服务，绑定线程操作人上下文；
  - `PrpSetupService.java` / `PrpSetupServiceImpl.java`：新增重载方法，执行 MP/PVT 匹配、新增/更新及历史写入。
- **验证与交付**：
  - `build.cmd jar.prp` 编译通过；Action 类编译通过。详细交付说明见：[LOT测试程序配置增加抓取量产Recipe功能](../../30-任务工作流与追踪/32-已交付需求(Done)/2026%20Q3/LOT测试程序配置增加抓取量产Recipe功能.md)。

