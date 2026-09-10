---
title: BOM程序Recipe配置与版本校验规则
date: 2026-09-10
author: MES Team
status: completed
tags: [BOM管理, Recipe配置, PDCU, 校验规则, PRP, SOP]
---

# BOM程序Recipe配置与版本校验规则

## 1. 业务背景与概述

在半导体芯片封装与测试（Packaging & Testing / FT）生产中，不同产品（BOM）、不同工艺阶段版本（BOM Version）、各道工序（Operation）以及特定工单对应的机台程序（Recipe）存在着严格的映射约束。

为规范机台 Recipe 程序的集中维护与生命周期管理，系统提供了 **Recipe程序配置（Recipe Program Configuration）** 功能模块（`/mycim2/recipeProgramConfig.do`），统一管理产品各站点所使用的机台主程序（`Recipe程序`）与串联设备从程序（`Inline Recipe程序`）。

针对研发测试、工程打样或特定试产工艺（如镀钯铜丝 PdCu 键合工艺、车规级 PDCU 动力域控制器测试等），其对应的 BOM 版本通常以 **`T`**（或小写 `t`）开头。为防范因程序选型或手工输入失误导致的严重工艺质量事故，系统建立了强校验机制：**维护 T 开头的 BOM 版本时，配置的程序名与 inline 程序名必须包含 `PDCU` 关键字**。

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
4. 确认提交后，系统将统一执行校验并批量更新选中的全部记录。

### 3.4 删除配置
- 选中一条或多条配置记录，点击 **【删除】**，二次确认后执行物理删除并写入历史变更表（`DELETE` 类型事务）。

---

## 4. T 开头 BOM 版本的 PDCU 校验卡控规则

### 4.1 业务卡控要求
- **适用对象**：所有以 `T` 或 `t` 开头的 BOM 版本（例如 `T01`, `T1.0`, `TA`, `t02` 等）；
- **卡控目标**：该配置下的机台程序名与 inline 程序名必须包含关键字 **`PDCU`**；
- **全场景覆盖**：单条新增保存、编辑修改、批量修改程序以及勾选抓取量产程序场景均严格受到卡控。

### 4.2 校验卡控逻辑细则

| 检验项目 | 判定规则 | 业务说明 |
| :--- | :--- | :--- |
| **BOM版本判定** | `bomVersion.trim().toUpperCase().startsWith("T")` | 不区分大小写，`T` 与 `t` 开头均进入卡控分支 |
| **关键字匹配** | `program.toUpperCase().contains("PDCU")` | 不区分大小写匹配，兼容 `PDCU`、`PdCu`、`pdcu` |
| **Recipe程序** | 若非空，必须包含 `PDCU` | 不符合时拦截并提示：`BOM版本为T开头时，Recipe程序必须包含PDCU` |
| **Inline Recipe程序** | 若非空，必须包含 `PDCU` | 不符合时拦截并提示：`BOM版本为T开头时，Inline Recipe程序必须包含PDCU` |
| **留空项处理** | 允许单项留空 | 封测产线多数单机台工序无 inline 程序，未填写的项不强制输入，仍维持“两者不能同时为空”的原规则 |

### 4.3 校验拦截场景示例矩阵

| 场景序号 | BOM版本 | Recipe程序 | Inline Recipe程序 | 校验结果 | 说明 |
| :---: | :---: | :---: | :---: | :---: | :--- |
| 1 | `T01` | `WB_PDCU_01` | *(留空)* | ✅ **允许保存** | 只有主程序，且主程序含 PDCU |
| 2 | `T01` | `WB_PdCu_01` | `INLINE_PDCU_01` | ✅ **允许保存** | 两者均含 PDCU，兼容大小写 |
| 3 | `T01` | `WB_TEST_01` | *(留空)* | ❌ **拦截报错** | Recipe程序未包含 PDCU |
| 4 | `T01` | `WB_PDCU_01` | `INLINE_TEST_01` | ❌ **拦截报错** | Inline Recipe程序未包含 PDCU |
| 5 | `01` / `A` | `WB_TEST_01` | *(留空)* | ✅ **允许保存** | 非 T 开头版本，不受 PDCU 规则约束 |

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

function containsPdcu(str) {
    return !Ext.isEmpty(str) && String(str).toUpperCase().indexOf('PDCU') !== -1;
}

// saveEditor 保存前校验
if (isTBomVersion(values.bomVersion)) {
    if (!Ext.isEmpty(values.recipeProgram) && !containsPdcu(values.recipeProgram)) {
        MyCim.notify.alert('BOM版本为T开头时，Recipe程序必须包含PDCU');
        return;
    }
    if (!Ext.isEmpty(values.recipeProgramInline) && !containsPdcu(values.recipeProgramInline)) {
        MyCim.notify.alert('BOM版本为T开头时，Inline Recipe程序必须包含PDCU');
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

private boolean containsPdcu(String programName) {
    return programName != null && programName.toUpperCase().contains("PDCU");
}

private void validatePdcuRecipePrograms(String bomVersion, String recipeProgram, String recipeProgramInline) {
    if (!isTBomVersion(bomVersion)) return;
    if (StringUtils.isNotBlank(recipeProgram) && !containsPdcu(recipeProgram)) {
        throw new MyCimParameterException("BOM版本为T开头时，程序名必须包含PDCU");
    }
    if (StringUtils.isNotBlank(recipeProgramInline) && !containsPdcu(recipeProgramInline)) {
        throw new MyCimParameterException("BOM版本为T开头时，inline程序名必须包含PDCU");
    }
}
```

---

## 6. 常见问题与排查指南

### Q1: 为什么批量修改程序时提示“选中的配置中包含T开头的BOM版本，Recipe程序必须包含PDCU”？
- **原因**：批量选中的记录中包含至少一条 BOM 版本以 T 开头的配置；
- **解决方式**：
  1. 若该批次属于研发试产，请输入符合规则包含 `PDCU` 的程序名；
  2. 若批量修改针对量产程序，请将量产记录与 T 开头试产记录分批选择并分别修改。

### Q2: 某些单机台工序没有 Inline 程序，是否可以只填 Recipe 程序？
- **可以**。系统规则为“至少填写一项”。如果工序只有单台机台、无 inline 串联设备，只需在 `Recipe程序` 中填写包含 `PDCU` 的程序名，`Inline Recipe程序` 保持留空即可顺利保存。

---

## 7. 详细更新记录

### [2026-09-10] BOM程序Recipe配置增加T开头版本PDCU校验限制

- **需求人**：黄志平
- **需求 / 背景**：
  - 在 BOM Recipe 程序配置（`recipeProgramConfig`）业务中，针对研发测试及特定试产工艺的 BOM 版本（以 `T` 或 `t` 开头），规范其机台程序名命名标准；
  - 业务规定：添加或维护以 `T` 开头的 BOM 版本时，配置的程序名（`Recipe程序`）和 `inline程序名`（`Inline Recipe程序`）中必须包含关键字 `PDCU`，防止研发试产批次因调错程序导致工艺事故；
  - 需在前端界面交互弹窗（新增、编辑、批量修改程序）与后端核心服务层进行统一强校验卡控，覆盖全部配置维护场景。
- **核心代码改动**：
  - `PrpSetupServiceImpl.java`：
    - 新增私有辅助方法 `isTBomVersion(String bomVersion)`、`containsPdcu(String programName)` 与 `validatePdcuRecipePrograms(String bomVersion, String recipeProgram, String recipeProgramInline)`；
    - 在 `ensureRecipeProgramConfigValues`（单条保存/更新）中增加卡控：当 BOM 版本以 `T`/`t` 开头时，对非空的 `recipeProgram` 及 `recipeProgramInline` 强制校验必须包含 `PDCU`（不区分大小写），若不包含则抛出 `MyCimParameterException` 拦截；
    - 在 `batchUpdateRecipeProgramConfigs`（批量修改程序）中增加卡控：当选中的配置属于 T 开头的 BOM 版本时，检查替换后的最终程序名与 inline 程序名，若不含 `PDCU` 则拒绝更新；
  - `recipeProgramConfig.js`：
    - 增加 `isTBomVersion(bomVersion)` 和 `containsPdcu(str)` 前端校验函数；
    - 在配置编辑保存窗口（`saveEditor`）中增加前置校验：当 BOM 版本以 T 开头时，填写的 Recipe 程序或 Inline Recipe 程序若不含 `PDCU`，弹窗友好提示并阻止发起 Ajax 请求；
    - 在批量修改窗口（`updatePrograms`）中增加前置校验：若选中的记录中包含 T 开头的 BOM 版本，检查填写的程序名，不符合规则时即时拦截提示。
- **数据库变动 (SQL)**：
  ```sql
  -- 本次无数据库表结构变更
  ```
- **配置与部署注意**：
  - 重新编译打包 `prp` 核心模块：执行 `ant jar.prp` 成功生成 `prpClient.jar`；
  - 前端静态 JS（`recipeProgramConfig.js`）部署或刷新浏览器缓存后生效。
- **自测情况**：
  - 编译构建验证：执行 `ant jar.core` 生成 `valueobject.jar`，执行 `ant jar.prp` 编译通过（`BUILD SUCCESSFUL`）；
  - 单条新增/编辑：BOM版本为 `T01` 时，若程序名不含 `PDCU`，前端弹窗提示并拦截；程序名包含 `PDCU` 时正常保存；
  - 留空项逻辑：单机台工序仅维护主程序名，未填写的 inline 程序名不强制要求输入（符合原业务“不能同时为空”的生产实际）；
  - 非 T 开头版本不受任何限制；
  - 批量修改场景对 T 开头版本正常拦截。

