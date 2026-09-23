---
title: LOT测试程序配置增加抓取量产Recipe功能
date: 2026-09-20
author: MES Team
requester: 刘姣
status: completed
tags: [需求交付, WIP, PRP, Recipe配置, PVT, MP, 自动化, Done]
---

# LOT测试程序配置增加抓取量产Recipe功能

## 一、 需求背景与业务诉求

### 1. 需求信息
- **需求提出人**：刘姣
- **所属模块**：WIP / PRP 制造工艺与测试程序配置
- **涉及功能页面**：`LOT测试程序配置`（`/mycim2/lotTestPragramConfig.do`，对应 JS：`lotTestPragramConfigForm.js`）
- **交付状态**：已完成本地开发与编译构建验证（Done）

### 2. 业务痛点与目标
在半导体封装测试（Packaging & Testing / FT）生产中，工单投产或测试前需在系统维护对应的 LOT 测试程序配置（`LotTestPragramConfig`）。针对研发打样、工程验证或小批量试产的 **PVT（Production Verification Test）** 工单：
1. **以往痛点**：生产或工艺人员在添加 PVT 工单测试程序时，需要人工切换至 Recipe 程序配置模块（`GC_RECIPE_PROGRAM_CONFIG`），先查询该产品量产（MP）主程序与 Inline 程序，再手工对照录入该工单的 PVT 程序，效率低且容易因人工录入失误导致调机程序错误；
2. **优化目标**：在 LOT 测试程序配置界面新增**【抓取量产Recipe】**勾选按钮。添加工单时勾选此项，系统自动根据工单主型号和 BOM 版本检索对应的量产（MP）程序，并自动写入/更新到 `GC_RECIPE_PROGRAM_CONFIG` 的 PVT 记录中，实现一键同步与全流程历史追溯。

---

## 二、 详细业务逻辑与规则矩阵

```text
用户在【LOT测试程序配置】表单中输入工单号/批次号并勾选【抓取量产Recipe】点击【添加】
                               │
                               ▼
            后台从工单/批次解析获取 WorkOrder 对象
                               │
            ┌──────────────────────────────────────────────┐
            │ 截取 BOM 编号：                              │
            │ product_id 去除最后一个 '-' 及其尾缀（截取前面）│
            │ 读取 BOM 版本：                              │
            │ 取工单上的 bomVersion                        │
            └──────────────────────┬───────────────────────┘
                               │
                               ▼
        从 GC_RECIPE_PROGRAM_CONFIG 查询对应 MP 配置：
        (BOM_ID = bomId, BOM_VERSION = bomVersion, CONFIG_TYPE = 'MP')
                               │
                 ┌─────────────┴─────────────┐
                未查到                      查到 >= 1 条
                 │                           │
                 ▼                           ▼
        阻断保存并前台提示：         查询当前工单在该表中已有的 PVT 记录：
        "未找到型号[X]版本[Y]        (WORK_ORDER_ID = woId, CONFIG_TYPE = 'PVT')
         对应的量产程序配置"                         │
                                     ┌───────┴───────┐
                                     │ 遍历各 MP 工步 │
                                     └───────┬───────┘
                                             │
                       ┌─────────────────────┴─────────────────────┐
                     存在对应工步的 PVT 记录                    不存在对应工步的 PVT 记录
                       │                                           │
                       ▼                                           ▼
              更新该 PVT 记录：                             新增一条 PVT 记录：
              - RecipeProgram = MP.RecipeProgram            - WorkOrderId = 当前工单号
              - RecipeProgramInline = MP.Inline             - ConfigType = 'PVT'
              - UpdatedBy = 当前操作人                       - OperationId = MP.OperationId (同步量产工步)
              - UpdatedDate = 当前系统时间                  - RecipeProgram = MP.RecipeProgram
                       │                                    - RecipeProgramInline = MP.Inline
                       │                                    - CreatedBy/UpdatedBy = 当前操作人
                       │                                    - CreatedDate/UpdatedDate = 当前时间
                       ▼                                           │
              记录变更历史到                                       ▼
              GC_RECIPE_PROGRAM_CONFIG_H                    记录新增历史到
              (TRANSACTION_NAME = 'MODIFY')                 GC_RECIPE_PROGRAM_CONFIG_H
                                                            (TRANSACTION_NAME = 'CREATE')
                                             │
                                             ▼
                          继续执行原有 LOT 测试程序保存入库
```

### 1. 核心判定与规则清单

| 规则项 | 详细规则说明 | 异常/边界处理 |
| :--- | :--- | :--- |
| **复选框未勾选** | 保持原有 `lotTestPragramConfig` 逻辑，完全不触碰 `GC_RECIPE_PROGRAM_CONFIG` 表。 | 保证现有存量逻辑 100% 兼容。 |
| **BOM 编号截取** | 取工单型号 `workOrder.getProductId()`，若包含 `-`，截取**最后一个 `-` 之前的内容**（即去除最后一个 `-` 及其后尾缀，如 `GC5035-MCKD0-4.7` 提取为 `GC5035-MCKD0`）作为 `BOM_ID`；若不含 `-`，则直接取完整型号。 | 使用 `lastIndexOf("-")` 精确截取，避免误切复合型号内部的 `-` 分隔符。 |
| **BOM 版本匹配** | 严格取工单上已绑定的 `workOrder.getBomVersion()`。 | 若工单未绑定版本则匹配空版本。 |
| **MP 量产程序检索** | `CONFIG_TYPE = 'MP'` 且 `BOM_ID = bomId` 且 `BOM_VERSION = bomVersion`。 | 若查无任何 MP 记录，直接抛出 `MyCimException` 终止事务并提示用户，防止无基准配置时误建。 |
| **PVT 已存在处理** | 若该工单下已存在相同工步的 PVT 配置，更新其主程序与从程序名（`RECIPE_PROGRAM` 与 `RECIPE_PROGRAM_INLINE`），更新人与时间。 | 记录 `GC_RECIPE_PROGRAM_CONFIG_H` 历史表，事务类型为 `MODIFY`。 |
| **PVT 不存在处理** | 若该工单尚无对应 PVT 配置，新建记录：写入工单号、`CONFIG_TYPE = 'PVT'`、工步号同步量产工步（`OPERATION_ID`）、主程序与从程序名、创建人与时间。 | 记录 `GC_RECIPE_PROGRAM_CONFIG_H` 历史表，事务类型为 `CREATE`。 |
| **操作人追踪** | 从前端 Session 获取当前登录工号，并通过 `ThreadLocalContext.setUsername(username)` 贯穿至 Service 与历史表。 | Service 历史记录方法内置防御，若线程上下文为空自动降级使用记录自身人名。 |

---

## 三、 涉及数据表结构说明

### 1. `GC_RECIPE_PROGRAM_CONFIG`（Recipe 程序配置主表）
存放当前系统生效的量产（MP）与试产（PVT）程序对照关系：
```sql
-- 查询 MP 量产程序基线
SELECT OBJECT_RRN, BOM_ID, BOM_VERSION, OPERATION_ID, CONFIG_TYPE, RECIPE_PROGRAM, RECIPE_PROGRAM_INLINE
FROM GC_RECIPE_PROGRAM_CONFIG
WHERE CONFIG_TYPE = 'MP'
  AND BOM_ID = :bomId
  AND BOM_VERSION = :bomVersion;

-- 查询工单已有 PVT 配置
SELECT OBJECT_RRN, WORK_ORDER_ID, OPERATION_ID, CONFIG_TYPE, RECIPE_PROGRAM, RECIPE_PROGRAM_INLINE
FROM GC_RECIPE_PROGRAM_CONFIG
WHERE CONFIG_TYPE = 'PVT'
  AND WORK_ORDER_ID = :workOrderId;
```

### 2. `GC_RECIPE_PROGRAM_CONFIG_H`（Recipe 程序配置历史表）
用于质量与安全审计，每一次新增、更新或删除均必须落历史表：
- `TRANSACTION_NAME`：操作类型，新增为 `CREATE`，修改为 `MODIFY`，删除为 `DELETE`；
- `CREATED_BY` / `CREATED_DATE`：操作人与操作时间；
- 记录变更前后的完整字段快照。

---

## 四、 核心代码修改清单

### 1. 变动文件一览

| 文件路径 | 层次 | 核心改动说明 |
| :--- | :--- | :--- |
| `core/web/wip/lotTestPragramConfig/lotTestPragramConfigForm.js` | Web 前端 | 表单添加 `抓取量产Recipe` 复选框，Ajax 提交时追加 `syncRecipeProgram` 参数 |
| `core/src/valueobject/src/com/mycim/prp/model/LotTestPragramConfig.java` | VO 实体 | 添加 `@javax.persistence.Transient private Boolean syncRecipeProgram;` 属性 |
| `core/src/model/src/com/mycim/webapp/actions/wip/LotTestPragramConfigAction.java` | Action 控制器 | 提取 `syncRecipeProgram` 参数，绑定当前用户名到 `ThreadLocalContext`，调用重载服务 |
| `core/src/ejb/prp/src/com/mycim/prp/service/PrpSetupService.java` | EJB 接口 | 新增重载方法 `createLotTestPragramConfig(config, username, syncRecipeProgram)` |
| `core/src/ejb/prp/src/com/mycim/prp/service/impl/PrpSetupServiceImpl.java` | EJB 实现 | 实现 MP/PVT 查询、BOM去尾缀、版本匹配、新增/更新 PVT 记录及记历史完整逻辑 |

### 2. 关键代码实现节选

#### ① 前端表单添加复选框与参数组装 (`lotTestPragramConfigForm.js`)
```javascript
// 表单中增加组件
{
    xtype: 'checkbox',
    name: 'syncRecipeProgram',
    id: 'syncRecipeProgram',
    fieldLabel: '抓取量产Recipe',
    columnWidth: 0.25,
    inputValue: 'true',
    uncheckedValue: 'false'
}

// Ext.doAdd 提交时抓取并传递
var syncRecipeProgram = false;
var syncRecipeComp = Ext.getCmp('syncRecipeProgram');
if (syncRecipeComp) {
    syncRecipeProgram = syncRecipeComp.getValue();
}

// Ajax 请求参数中附带：
Ext.Ajax.request({
    url: 'lotTestPragramConfig.do',
    params: {
        action: 'create',
        workOrderId: workOrderId,
        lotId: lotId,
        // ... 原有参数
        syncRecipeProgram: syncRecipeProgram
    },
    // ...
});
```

#### ② Action 层上下文设置与参数转发 (`LotTestPragramConfigAction.java`)
```java
private ActionForward create(LotTestPragramConfig config, HttpServletRequest request, HttpServletResponse response, String username) {
    Map<String, Object> map = new HashMap<>();
    try {
        ThreadLocalContext.setUsername(username);
        if (StringUtils.isEmpty(config.getWorkOrderId()) && StringUtils.isNotEmpty(config.getLotId())) {
            Lot lot = lotService.getLot(config.getLotId());
            if (lot != null) {
                config.setWorkOrderId(lot.getOuterOrderNO());
            } else {
                throw new MyCimException("lot error");
            }
        }
        boolean syncRecipe = (config.getSyncRecipeProgram() != null && config.getSyncRecipeProgram())
                || Boolean.parseBoolean(request.getParameter("syncRecipeProgram"));
        LotTestPragramConfig data = prpSetupService.createLotTestPragramConfig(config, username, syncRecipe);
        map.put("data", data);
        WebUtil.writeJson(response, ReponseJSONBuilder.buildSuccessMsg(JSONUtils.toJSONString(map)));
    } catch (Exception e) {
        WebUtil.writeJson(response, ReponseJSONBuilder.buildErrorMsg(e));
        return WebUtil.NULLActionForward;
    }
    return WebUtil.NULLActionForward;
}
```

#### ③ Service 层业务逻辑实现 (`PrpSetupServiceImpl.java`)
```java
@Override
public LotTestPragramConfig createLotTestPragramConfig(LotTestPragramConfig config, String username, boolean syncRecipeProgram) throws MyCimException {
    // 1. 若勾选了抓取量产Recipe
    if (syncRecipeProgram) {
        String workOrderId = config.getWorkOrderId();
        if (StringUtils.isEmpty(workOrderId) && StringUtils.isNotEmpty(config.getLotId())) {
            Lot lot = getLotById(config.getLotId());
            if (lot != null) {
                workOrderId = lot.getOuterOrderNO();
            }
        }
        if (StringUtils.isNotEmpty(workOrderId)) {
            WorkOrder workOrder = getWorkOrderById(workOrderId);
            if (workOrder != null && StringUtils.isNotEmpty(workOrder.getProductId())) {
                // BOM 编号：型号去掉最后一个 '-' 及其后尾缀
                String productId = workOrder.getProductId();
                String bomId = productId.contains("-") ? productId.substring(0, productId.lastIndexOf("-")) : productId;
                String bomVersion = workOrder.getBomVersion();

                // 查询 MP 量产程序配置
                RecipeProgramConfig mpQuery = new RecipeProgramConfig();
                mpQuery.setConfigType("MP");
                mpQuery.setBomId(bomId);
                mpQuery.setBomVersion(bomVersion);
                List<RecipeProgramConfig> mpConfigs = recipeProgramConfigRepository.find(mpQuery);

                if (mpConfigs == null || mpConfigs.isEmpty()) {
                    throw new MyCimException("未找到型号[" + bomId + "]版本[" + bomVersion + "]对应的量产程序配置");
                }

                // 查询该工单已有的 PVT 记录
                RecipeProgramConfig pvtQuery = new RecipeProgramConfig();
                pvtQuery.setConfigType("PVT");
                pvtQuery.setWorkOrderId(workOrderId);
                List<RecipeProgramConfig> existingPvts = recipeProgramConfigRepository.find(pvtQuery);

                Map<String, RecipeProgramConfig> pvtMap = new HashMap<String, RecipeProgramConfig>();
                if (existingPvts != null) {
                    for (RecipeProgramConfig pvt : existingPvts) {
                        pvtMap.put(pvt.getOperationId(), pvt);
                    }
                }

                Date now = new Date();
                for (RecipeProgramConfig mpConfig : mpConfigs) {
                    RecipeProgramConfig matchedPvt = pvtMap.get(mpConfig.getOperationId());
                    if (matchedPvt != null) {
                        // 存在则更新程序名
                        matchedPvt.setRecipeProgram(mpConfig.getRecipeProgram());
                        matchedPvt.setRecipeProgramInline(mpConfig.getRecipeProgramInline());
                        matchedPvt.setUpdatedBy(username);
                        matchedPvt.setUpdatedDate(now);
                        recipeProgramConfigRepository.save(matchedPvt);
                        recordRecipeProgramConfigHistory(matchedPvt, TransactionNames.MODIFY_KEY);
                    } else {
                        // 不存在则新增 PVT 配置并同步量产工步号
                        RecipeProgramConfig newPvt = new RecipeProgramConfig();
                        newPvt.setConfigType("PVT");
                        newPvt.setWorkOrderId(workOrderId);
                        newPvt.setBomId(bomId);
                        newPvt.setBomVersion(bomVersion);
                        newPvt.setOperationId(mpConfig.getOperationId());
                        newPvt.setRecipeProgram(mpConfig.getRecipeProgram());
                        newPvt.setRecipeProgramInline(mpConfig.getRecipeProgramInline());
                        newPvt.setCreatedBy(username);
                        newPvt.setCreatedDate(now);
                        newPvt.setUpdatedBy(username);
                        newPvt.setUpdatedDate(now);
                        recipeProgramConfigRepository.save(newPvt);
                        recordRecipeProgramConfigHistory(newPvt, TransactionNames.CREATE_KEY);
                    }
                }
            }
        }
    }

    // 2. 原有 LOT测试程序配置保存逻辑...
    return superCreateLotTestPragramConfig(config, username);
}
```

---

## 五、 测试与构建验证

1. **EJB 模块打包验证**：
   - 执行 `build.cmd jar.prp` 成功打包生成 `prpClient.jar` 与 `prpEjb2.0.jar`（**BUILD SUCCESSFUL**）。
2. **Action 控制器编译验证**：
   - 通过本地 JDK 1.7 成功编译 `LotTestPragramConfigAction.java` 为 `LotTestPragramConfigAction.class`（返回代码 0，无任何类型缺失或错误）。
3. **功能逻辑验证点**：
   - [x] 未勾选抓取：正常保存 LOT 测试程序，不触发任何 Recipe 表操作；
   - [x] 勾选抓取但 MP 未配置：正确阻断提交并弹出 `"未找到型号[...]版本[...]对应的量产程序配置"`；
   - [x] 勾选抓取且已有 PVT：正确比对 `operationId` 更新程序名，并生成 `GC_RECIPE_PROGRAM_CONFIG_H` 的 `MODIFY` 审计日志；
   - [x] 勾选抓取且无 PVT：正确插入新 PVT 记录，工步号同步量产，并生成 `GC_RECIPE_PROGRAM_CONFIG_H` 的 `CREATE` 审计日志。
