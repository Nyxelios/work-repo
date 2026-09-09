---
title: 优化-WorkOrderProduce-文件夹代码
date: 2026-05-03
tags: [AI协作, MES]
---
# 优化 WorkOrderProduce 文件夹代码

> **Workspace:** d:\data\MES\gc

---

## User
core/web/prp/WorkOrderProduce  帮我看下这个文件夹里的代码，我总感觉自己写的很乱，你看能有什么优化的空间吗

## Assistant
我来帮你看看这个文件夹里的代码，分析一下是否有优化的空间。

首先让我查看一下这个文件夹里的文件：



让我读取这些主要文件来分析代码结构：



现在让我再看看子目录下的文件来了解更多上下文：



我已经详细阅读了整个 `WorkOrderProduce` 文件夹的代码。作为一个基于 ExtJS 的 MES 工单生产模块，代码整体功能完整，但确实存在不少可以优化的地方。让我为你详细分析：

## 📋 主要问题分析

### 1. **架构层面问题**

#### ① 全局变量污染严重
在 [common.js](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/common.js) 和 [app.js](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/app.js) 中，大量使用全局变量：
- `cpAdvanceRwFlag`、`pageCategory` 等配置变量
- 7个 Combo Store（`targetGradeComboStore`、`routeIdComboStore` 等）
- `WmsMLotStore`、`MesMLotStore`

**问题**：容易造成命名冲突，难以维护，不利于测试。

#### ② 污染 Ext 命名空间
在 [common.js](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/common.js) 中直接挂载方法到 `Ext` 对象：
```javascript
Ext.queryWorkOrder = function() {...}
Ext.createWorkOrder = function() {...}
Ext.createComboListeners = function() {...}
```

**问题**：`Ext` 是第三方库的命名空间，不应该被业务代码污染。

#### ③ 各车间 JSP 高度重复
对比 [COM/WorkOrderProduce.jsp](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/COM/WorkOrderProduce.jsp)、[FT/WorkOrderProduce.jsp](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/FT/WorkOrderProduce.jsp)、[RC/WorkOrderProduce.jsp](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/RC/WorkOrderProduce.jsp)，几乎完全相同，只有 `pageCategory` 和 `<title>` 不同。

---

### 2. **代码质量问题**

#### ① `initPage` 函数违反开闭原则
[app.js#L9-L56](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/app.js#L9-L56) 中大量 if-else：

```javascript
if (pageCategory === 'COM') {
    // ...
} else if (pageCategory === 'OIS') {
    // ...
} else if (pageCategory === '回收清洗组') {
    // ...
} else if (pageCategory === 'FT') {
    // ...
} else if (pageCategory === 'RW') {
    // ...
}
```

**问题**：每增加一个新车间，就需要修改这个函数，容易引入 bug。

#### ② `createComboListeners` 函数过长
[common.js#L224-L411](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/common.js#L224-L411) 将近 200 行，包含大量 if-else 分支，职责不清。

#### ③ 测试代码未清理
[WorkOrderConstant.js#L18-L30](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/WorkOrderConstant.js#L18-L30)：
```javascript
{
    text: "1111",
    handler: function () {
        return ajaxPromise({
            url: '/mycim2/WorkOrderProduce.do?action=1234',
        })
    }
}
```

---

### 3. **性能和维护性问题**

#### ① 频繁使用 `Ext.getCmp`
全局查找组件效率低，应该保存组件引用或使用 `down()` / `up()` 相对查找。

#### ② 不合理的分页配置
[MesMLotGrid.js#L17](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/MesMLotGrid.js#L17) 和 [WmsMLotGrid.js#L24](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/WmsMLotGrid.js#L24)：
```javascript
var orPageSize = 99999;
var obPageSize = 99999;
```

**问题**：一次性加载 99999 条数据会严重影响性能。

#### ④ 混合使用回调和 Promise
[common.js](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/common.js) 中 `ajaxPromise` 和 `doSuccessCallback` 混用，异步处理不一致。

---

## 🔧 优化建议

### 高优先级（强烈建议）

#### 1. 使用命名空间封装，避免全局污染

```javascript
// 创建统一的命名空间
var WorkOrderProduce = WorkOrderProduce || {};

// 将所有 Store 封装
WorkOrderProduce.stores = {
    targetGrade: Ext.create('Ext.data.Store', {...}),
    routeId: Ext.create('Ext.data.Store', {...}),
    // ...
};

// 将方法封装
WorkOrderProduce.actions = {
    queryWorkOrder: function(workorderId) {...},
    createWorkOrder: function(params) {...},
    // ...
};
```

#### 2. 使用配置对象重构 `initPage`

```javascript
// 将各车间的配置抽离为配置对象
var workshopConfig = {
    'COM': {
        waferSource: '16',
        hideComponents: ['currentSubQty'],
        hideGrids: ['WmsMLotGrid.currentSubQty']
    },
    'OIS': {
        waferSource: '22',
        hideComponents: ['waferLevel', 'waferSource'],
        columnConfigs: { storageModel: 0.22 }
    },
    'FT': {
        showComponents: ['targetGrade'],
        hideComponents: ['waferLevel']
    },
    'RW': {
        showComponents: ['checkRemark', 'btCpRw', 'pendingQuantity']
    }
};

function initPage() {
    var config = workshopConfig[pageCategory];
    if (!config) return;
    
    // 统一处理逻辑
    if (config.waferSource) {
        Ext.getCmp("waferSource").setValue(config.waferSource);
    }
    // 隐藏组件
    (config.hideComponents || []).forEach(id => {
        var comp = Ext.getCmp(id);
        if (comp) comp.setVisible(false);
    });
    // ...
}
```

#### 3. 创建 Store 工厂函数，消除重复代码

```javascript
// 替换 7 个重复的 Store 定义
function createComboStore(action) {
    return Ext.create('Ext.data.Store', {
        fields: ['key', 'value'],
        proxy: {
            type: 'ajax',
            actionMethods: {read: 'GET'},
            url: '/mycim2/WorkOrderProduce.do?action=' + action,
            queryMask: false,
        },
        autoLoad: false
    });
}

// 使用
var targetGradeComboStore = createComboStore('targetGradeComboBox');
var routeIdComboStore = createComboStore('routeIdComboBox');
var productIdComboStore = createComboStore('productIdComboBox');
```

#### 4. 拆分 `createComboListeners`

```javascript
// 将每个 method 的处理逻辑拆成独立函数
var comboHandlers = {
    storageModel: function(params) {
        if (Ext.isEmpty(Ext.getCmp('waferSource').getValue())) {
            MyCim.notify.alert("wafer属性不能为空");
            return;
        }
        // ...
    },
    bomCategory: function(params) {
        // ...
    },
    bomVersion: function(params) {
        // ...
    }
};

// 主函数只需路由
Ext.createComboListeners = function(params) {
    return {
        focus: function(combo) {
            var handler = comboHandlers[params.method];
            if (handler) handler(params);
        },
        select: function(combo, value) {
            // ...
        }
    };
};
```

---

### 中优先级（建议优化）

#### 5. 删除测试代码
删除 [WorkOrderConstant.js](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/WorkOrderConstant.js#L18-L30) 中的 "1111" 按钮。

#### 6. 优化分页大小
```javascript
// 改为合理的分页大小
var DEFAULT_PAGE_SIZE = 50; // 或 100
```

#### 7. 减少 `Ext.getCmp` 使用
```javascript
// 不好
var workorderId = Ext.getCmp('workorderId').getValue();

// 好：在需要的地方缓存引用
var formPanel = Ext.getCmp('WorkOrderProduceForm');
var values = formPanel.getForm().getValues();
```

#### 8. 统一异步处理风格
将 `doSuccessCallback` 改为 Promise 风格，或使用 Store 的事件监听。

---

### 低优先级（可选优化）

#### 9. 合并子目录的 JSP 文件
通过 URL 参数传递 `pageCategory`，使用同一个 JSP 文件。

#### 10. 提取常量
```javascript
var ACTIONS = {
    QUERY_WORK_ORDER: 'ReadWorkOrder',
    CREATE_WORK_ORDER: 'CreateWorkOrder',
    UPDATE_WORK_ORDER: 'UpdateWorkOrder',
    // ...
};

var COMPONENT_IDS = {
    WORK_ORDER_ID: 'workorderId',
    STATUS: 'status',
    PLANNED_QTY: 'plannedQty',
    // ...
};
```

#### 11. 添加 JSDoc 注释
为复杂函数添加类型注解和参数说明，提高可维护性。

---

## 📊 优先级总结

| 优先级 | 优化项 | 影响范围 | 实施难度 |
|--------|--------|----------|----------|
| 🔴 高 | 命名空间封装 | 全局 | 中 |
| 🔴 高 | 重构 initPage | app.js | 低 |
| 🔴 高 | Store 工厂函数 | common.js | 低 |
| 🔴 高 | 拆分 createComboListeners | common.js | 中 |
| 🟡 中 | 删除测试代码 | WorkOrderConstant.js | 低 |
| 🟡 中 | 优化分页大小 | Grid 文件 | 低 |
| 🟢 低 | 合并 JSP | 子目录 | 中 |

---

你希望我帮你实施哪些优化？我建议先从**高优先级**的项开始，可以显著提升代码质量。

## User
列一下每项的优化具体措施，但是不要改动代码

## Assistant
好的，我来为每个优化项列出**具体的实施措施**，方便你评估和后续实施。

---

## 🔴 高优先级优化措施

### 1. **命名空间封装，避免全局污染**

**涉及文件：**
- [common.js](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/common.js)
- [app.js](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/app.js)
- [WorkOrderConstant.js](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/WorkOrderConstant.js)

**具体措施：**

**步骤 1：** 在 `common.js` 顶部创建命名空间结构
```javascript
// 原有代码：
var targetGradeComboStore = Ext.create('Ext.data.Store', {...});
var routeIdComboStore = Ext.create('Ext.data.Store', {...});
// ... 共 7 个 Store

// 改造后：
var WorkOrderProduce = WorkOrderProduce || {};
WorkOrderProduce.stores = {};
WorkOrderProduce.stores.targetGrade = Ext.create('Ext.data.Store', {...});
WorkOrderProduce.stores.routeId = Ext.create('Ext.data.Store', {...});
// ...
```

**步骤 2：** 将 `Ext` 上的方法迁移到命名空间
```javascript
// 原有代码：
Ext.queryWorkOrder = function(workorderId) {...}
Ext.createWorkOrder = function(params) {...}
Ext.updateWorkOrder = function(params) {...}

// 改造后：
WorkOrderProduce.actions = {};
WorkOrderProduce.actions.queryWorkOrder = function(workorderId) {...}
WorkOrderProduce.actions.createWorkOrder = function(params) {...}
WorkOrderProduce.actions.updateWorkOrder = function(params) {...}
```

**步骤 3：** 全局替换所有引用
- 搜索 `targetGradeComboStore` → 替换为 `WorkOrderProduce.stores.targetGrade`
- 搜索 `Ext.queryWorkOrder` → 替换为 `WorkOrderProduce.actions.queryWorkOrder`
- 依此类推...

**步骤 4：** 将 `cpAdvanceRwFlag` 等配置变量放入命名空间
```javascript
WorkOrderProduce.config = {
    cpAdvanceRwFlag: 'N',
    pageSize: 50  // 同时解决分页问题
};
```

**预期效果：**
- 全局变量从 15+ 个减少到 1 个（`WorkOrderProduce`）
- 避免与其他模块命名冲突
- 便于单元测试和代码重构

---

### 2. **重构 initPage 函数（配置驱动）**

**涉及文件：**
- [app.js#L9-L56](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/app.js#L9-L56)

**具体措施：**

**步骤 1：** 在 `app.js` 顶部定义配置对象
```javascript
var workshopConfig = {
    'COM': {
        setValue: {
            'waferSource': '16'
        },
        setVisible: {
            'currentSubQty': false  // WmsMLotGrid 下的组件
        }
    },
    'OIS': {
        setColumnWidth: {
            'storageModel': 0.22
        },
        setVisible: {
            'waferLevel': false,
            'waferSource': false,
            'currentSubQty': false,
            'unitId': false,
            'lotId': false,
            'mesObjectId': false,
            'mesLotId': false
        },
        setValue: {
            'waferSource': '22'
        }
    },
    '回收清洗组': {
        setValue: {
            'waferSource': '31'
        },
        setVisible: {
            'waferSource': false
        }
    },
    'FT': {
        setVisible: {
            'targetGrade': true,
            'waferLevel': false,
            'unitId': false,
            'lotId': false,
            'currentSubQty': false,
            'mesObjectId': false,
            'mesLotId': false
        }
    },
    'RW': {
        setVisible: {
            'checkRemark': true,
            'btCpRw': true,      // WmsMLotForm 下的组件
            'pendingQuantity': true  // MesMLotGrid 下的组件
        }
    }
};
```

**步骤 2：** 重写 `initPage` 函数
```javascript
function initPage() {
    console.log('页面加载完成，开始设置默认值');
    console.log('cpAdvanceRwFlag：' + WorkOrderProduce.config.cpAdvanceRwFlag);
    console.log('pageCategory：' + pageCategory);

    // 设置 workshop 值
    Ext.getCmp("workshop").setValue(pageCategory);

    // 获取当前车间配置
    var config = workshopConfig[pageCategory];
    if (!config) {
        console.log('未找到车间配置：' + pageCategory);
        return;
    }

    // 获取组件引用
    var MesMLotGrid = Ext.getCmp("MesMLotGrid");
    var WmsMLotGrid = Ext.getCmp("WmsMLotGrid");
    var WmsMLotForm = Ext.getCmp("WmsMLotForm");

    // 处理 setValue
    if (config.setValue) {
        Object.keys(config.setValue).forEach(function(id) {
            var comp = Ext.getCmp(id);
            if (comp) comp.setValue(config.setValue[id]);
        });
    }

    // 处理 setVisible
    if (config.setVisible) {
        Object.keys(config.setVisible).forEach(function(id) {
            var visible = config.setVisible[id];
            // 优先从 Grid/Form 中查找
            var comp = MesMLotGrid.down('#' + id) || 
                       WmsMLotGrid.down('#' + id) || 
                       WmsMLotForm.down('#' + id) ||
                       Ext.getCmp(id);
            if (comp) comp.setVisible(visible);
        });
    }

    // 处理 setColumnWidth
    if (config.setColumnWidth) {
        Object.keys(config.setColumnWidth).forEach(function(id) {
            var comp = Ext.getCmp(id);
            if (comp) comp.columnWidth = config.setColumnWidth[id];
        });
    }

    console.log('默认值设置完成');
}
```

**步骤 3：** 新增车间时只需添加配置
```javascript
// 例如新增 'FT-AE' 车间
'FT-AE': {
    setValue: { 'waferSource': '25' },
    setVisible: { 'targetGrade': true }
}
```

**预期效果：**
- 消除 50+ 行 if-else 代码
- 新增车间无需修改逻辑代码
- 配置清晰，易于维护

---

### 3. **创建 Store 工厂函数**

**涉及文件：**
- [common.js#L9-L75](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/common.js#L9-L75)

**具体措施：**

**步骤 1：** 定义工厂函数
```javascript
// 替换原有的 7 个 Store 定义
WorkOrderProduce.createStore = function(action, autoLoad) {
    return Ext.create('Ext.data.Store', {
        fields: ['key', 'value'],
        proxy: {
            type: 'ajax',
            actionMethods: {read: 'GET'},
            url: '/mycim2/WorkOrderProduce.do?action=' + action,
            queryMask: false,
        },
        autoLoad: autoLoad || false
    });
};
```

**步骤 2：** 使用工厂函数创建 Store
```javascript
// 原有代码（75 行）：
var targetGradeComboStore = Ext.create('Ext.data.Store', {
    fields: ['key', 'value'],
    proxy: {
        type: 'ajax',
        actionMethods: {read: 'GET'},
        url: '/mycim2/WorkOrderProduce.do?action=targetGradeComboBox',
        queryMask: false,
    },
    autoLoad: false
});
var routeIdComboStore = Ext.create('Ext.data.Store', {...});
var productIdComboStore = Ext.create('Ext.data.Store', {...});
// ... 重复 4 次

// 改造后（7 行）：
WorkOrderProduce.stores = {
    targetGrade: WorkOrderProduce.createStore('targetGradeComboBox'),
    routeId: WorkOrderProduce.createStore('routeIdComboBox'),
    productId: WorkOrderProduce.createStore('productIdComboBox'),
    storageModel: WorkOrderProduce.createStore('finishProductIdComboBox'),
    putProductId: WorkOrderProduce.createStore('putProductIdComboBox'),
    bomVersion: WorkOrderProduce.createStore('queryBomInfo'),
    bomCategory: WorkOrderProduce.createStore('queryBomInfo')
};
```

**步骤 3：** 更新所有 Store 引用
- `targetGradeComboStore` → `WorkOrderProduce.stores.targetGrade`
- `routeIdComboStore` → `WorkOrderProduce.stores.routeId`
- 依此类推...

**预期效果：**
- 代码量从 75 行减少到 15 行（减少 80%）
- 新增 Store 只需一行代码
- 统一 Store 创建逻辑，便于添加全局配置

---

### 4. **拆分 createComboListeners 函数**

**涉及文件：**
- [common.js#L224-L411](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/common.js#L224-L411)

**具体措施：**

**步骤 1：** 提取通用辅助函数
```javascript
// 新增：获取表单值的辅助函数
WorkOrderProduce.getFormValues = function() {
    return {
        waferSource: Ext.getCmp('waferSource').getValue(),
        waferSourceDesc: Ext.getCmp('waferSource').getRawValue(),
        storageModel: Ext.getCmp('storageModel').getValue(),
        bomCategory: Ext.getCmp('bomCategory').getValue(),
        workOrderType: Ext.getCmp('workOrderType').getValue(),
        bomVersion: Ext.getCmp('bomVersion').getValue(),
        putProductId: Ext.getCmp('putProductId').getValue(),
        productId: Ext.getCmp('productId').getValue(),
        route: Ext.getCmp('route').getValue(),
        levelTwoCode: Ext.getCmp('levelTwoCode').getValue()
    };
};

// 新增：验证并加载 Store 的通用函数
WorkOrderProduce.validateAndLoad = function(requiredFields, pStore, extraParams) {
    var values = WorkOrderProduce.getFormValues();
    
    // 验证必填字段
    for (var i = 0; i < requiredFields.length; i++) {
        var field = requiredFields[i];
        if (Ext.isEmpty(values[field])) {
            MyCim.notify.alert(field + '不能为空');
            return false;
        }
    }
    
    // 设置参数并加载
    var proxy = pStore.getProxy();
    Object.keys(extraParams).forEach(function(key) {
        proxy.extraParams[key] = extraParams[key];
    });
    pStore.load();
    return true;
};
```

**步骤 2：** 拆分每个 method 的处理逻辑
```javascript
WorkOrderProduce.comboHandlers = {
    storageModel: function(params) {
        var values = WorkOrderProduce.getFormValues();
        return WorkOrderProduce.validateAndLoad(
            ['waferSource'],
            params.pStore,
            {
                waferSource: values.waferSource,
                waferSourceDesc: values.waferSourceDesc
            }
        );
    },
    
    bomCategory: function(params) {
        var values = WorkOrderProduce.getFormValues();
        return WorkOrderProduce.validateAndLoad(
            ['storageModel'],
            params.pStore,
            {
                bomType: 'bomCategory',
                storageModel: values.storageModel
            }
        );
    },
    
    bomVersion: function(params) {
        var values = WorkOrderProduce.getFormValues();
        return WorkOrderProduce.validateAndLoad(
            ['storageModel', 'bomCategory'],
            params.pStore,
            {
                bomType: 'bomVersion',
                storageModel: values.storageModel,
                bomCategory: values.bomCategory
            }
        );
    },
    
    putProductId: function(params) {
        var values = WorkOrderProduce.getFormValues();
        var requiredFields = ['storageModel'];
        
        // 根据工单类型动态添加必填字段
        var workOrderType = values.workOrderType;
        if (workOrderType != '3' && workOrderType != '5' && 
            workOrderType != '8' && workOrderType != '9') {
            requiredFields.push('bomCategory', 'bomVersion', 'waferSource');
        }
        
        return WorkOrderProduce.validateAndLoad(
            requiredFields,
            params.pStore,
            {
                storageModel: values.storageModel,
                bomVersion: values.bomVersion,
                bomCategory: values.bomCategory,
                waferSource: values.waferSource,
                waferSourceDesc: values.waferSourceDesc
            }
        );
    },
    
    productId: function(params) {
        var values = WorkOrderProduce.getFormValues();
        return WorkOrderProduce.validateAndLoad(
            ['waferSource', 'putProductId'],
            params.pStore,
            {
                waferSource: values.waferSource,
                waferSourceDesc: values.waferSourceDesc,
                putProductId: values.putProductId,
                workOrderType: values.workOrderType
            }
        );
    },
    
    route: function(params) {
        var values = WorkOrderProduce.getFormValues();
        
        // 计算 levelTwoCode
        var levelTwoCode = values.waferSource == '16' 
            ? values.bomVersion 
            : values.levelTwoCode;
        
        return WorkOrderProduce.validateAndLoad(
            ['waferSource', 'productId', 'workOrderType'],
            params.pStore,
            {
                waferSource: values.waferSource,
                waferSourceDesc: values.waferSourceDesc,
                productId: values.productId,
                levelTwoCode: levelTwoCode,
                orderType: values.workOrderType
            }
        );
    },
    
    targetGrade: function(params) {
        var values = WorkOrderProduce.getFormValues();
        return WorkOrderProduce.validateAndLoad(
            ['waferSource', 'productId', 'route'],
            params.pStore,
            {
                productId: values.productId,
                routeId: values.route,
                waferSource: values.waferSource,
                waferSourceDesc: values.waferSourceDesc
            }
        );
    }
};
```

**步骤 3：** 重写 `createComboListeners` 为路由函数
```javascript
Ext.createComboListeners = function(params) {
    return {
        focus: function(combo) {
            var handler = WorkOrderProduce.comboHandlers[params.method];
            if (handler) {
                handler(params);
            }
        },
        select: function(combo, value) {
            if (!Ext.isEmpty(value)) {
                // 清空字段
                Ext.Array.each(params.clearFields, function(field) {
                    Ext.getCmp(field).setValue('');
                });
                Ext.Array.each(params.storeToClear, function(storeClear) {
                    storeClear.removeAll();
                });
                
                // 处理特定方法的选择逻辑
                if (params.method === 'bondedProperty') {
                    WorkOrderProduce.comboHandlers.bondedPropertySelect(combo);
                } else if (params.method === 'putProductId') {
                    WorkOrderProduce.comboHandlers.putProductIdSelect(combo);
                }
            }
        }
    };
};

// 拆分 select 逻辑
WorkOrderProduce.comboHandlers.bondedPropertySelect = function() {
    var bondedProperty = Ext.getCmp('bondedProperty').getValue();
    if (bondedProperty === 'SH') {
        Ext.getCmp('processMethod').setValue('ST');
    } else if (bondedProperty === 'ZSH') {
        Ext.getCmp('processMethod').setValue('ZC');
    }
};

WorkOrderProduce.comboHandlers.putProductIdSelect = function(combo) {
    var values = WorkOrderProduce.getFormValues();
    var storageModel = values.storageModel;
    
    if (values.waferSource === '16') {
        var store = WorkOrderProduce.stores.productId;
        store.getProxy().extraParams.waferSource = values.waferSource;
        store.getProxy().extraParams.waferSourceDesc = values.waferSourceDesc;
        store.getProxy().extraParams.putProductId = values.putProductId;
        store.getProxy().extraParams.workOrderType = values.workOrderType;
        
        store.load({
            callback: function(records, operation, success) {
                if (success) {
                    records.forEach(function(record) {
                        if (storageModel == record.data.value) {
                            Ext.getCmp('productId').setValue(storageModel);
                        }
                    });
                }
            }
        });
    }
};
```

**预期效果：**
- 单个函数从 190 行减少到 30 行
- 每个 handler 职责单一，便于测试
- 新增 combo 类型只需添加一个 handler

---

## 🟡 中优先级优化措施

### 5. **删除测试代码**

**涉及文件：**
- [WorkOrderConstant.js#L17-L30](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/WorkOrderConstant.js#L17-L30)

**具体措施：**

**步骤 1：** 删除测试按钮配置
```javascript
// 删除这段代码：
{
    text: "1111",
    region: 'center',
    handler: function () {
        MyCim.notify.confirm("是否确认操作", function (button) {
            if (button === "yes" || button === "ok") {
                return ajaxPromise({
                    url: '/mycim2/WorkOrderProduce.do?action=1234',
                }).then(responseData => {

                })
            }
        });
    }
},
```

**步骤 2：** 检查是否有其他测试代码
- 搜索 `console.log`（保留必要的调试日志）
- 搜索 `debugger`
- 搜索注释掉的代码块

---

### 6. **优化分页大小**

**涉及文件：**
- [MesMLotGrid.js#L17](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/MesMLotGrid.js#L17)
- [WmsMLotGrid.js#L24](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/WmsMLotGrid.js#L24)

**具体措施：**

**步骤 1：** 在命名空间中定义合理分页大小
```javascript
WorkOrderProduce.config = {
    cpAdvanceRwFlag: 'N',
    pageSize: 50,  // 从 99999 改为 50
    // ... 其他配置
};
```

**步骤 2：** 修改 MesMLotGrid.js
```javascript
// 原代码：
var orPageSize = 99999;
Ext.define('WorkOrderProduceViewer.MesMLotStore', {
    extend: 'Ext.data.Store',
    model: 'MesMLotModel',
    autoLoad: false,
    pageSize: orPageSize,
    proxy: {
        params: {
            limit: orPageSize
        },
        // ...
    }
});

// 改造后：
Ext.define('WorkOrderProduceViewer.MesMLotStore', {
    extend: 'Ext.data.Store',
    model: 'MesMLotModel',
    autoLoad: false,
    pageSize: WorkOrderProduce.config.pageSize,
    proxy: {
        params: {
            limit: WorkOrderProduce.config.pageSize
        },
        // ...
    }
});
```

**步骤 3：** 同样修改 WmsMLotGrid.js

**预期效果：**
- 首次加载数据量从 99999 条减少到 50 条
- 页面加载速度提升 90%+
- 减少内存占用

---

### 7. **减少 Ext.getCmp 使用**

**涉及文件：**
- 所有 JS 文件

**具体措施：**

**步骤 1：** 在 `WorkOrderProduceViewer.js` 中缓存组件引用
```javascript
// 在 App 类中添加组件缓存
Ext.define('WorkOrderProduceViewer.App', {
    extend: 'Ext.container.Viewport',
    
    // 新增：组件缓存
    componentCache: {},
    
    // 新增：获取组件的方法
    getComponent: function(id) {
        if (!this.componentCache[id]) {
            this.componentCache[id] = Ext.getCmp(id);
        }
        return this.componentCache[id];
    },
    
    // 新增：将组件注册到缓存
    registerComponent: function(id, component) {
        this.componentCache[id] = component;
    },
    
    // ... 其他代码
});
```

**步骤 2：** 在组件创建时注册到缓存
```javascript
createWorkOrderProduceTab: function () {
    var me = this;

    var WorkOrderProduceForm = Ext.create('WorkOrderProduceViewer.WorkOrderProduceForm', {
        id: 'WorkOrderProduceForm',
        region: 'north',
    });
    me.registerComponent('WorkOrderProduceForm', WorkOrderProduceForm);

    var MesMLotGrid = Ext.create('WorkOrderProduceViewer.MesMLotGrid', {
        id: 'MesMLotGrid',
        region: 'west',
        width: '50%',
        store: WorkOrderProduce.stores.mesMLot
    });
    me.registerComponent('MesMLotGrid', MesMLotGrid);

    // ... 其他组件也注册
}
```

**步骤 3：** 在 `common.js` 中使用缓存
```javascript
// 原代码：
Ext.queryWorkOrder = function (workorderId) {
    var workorderId = Ext.getCmp('workorderId').getValue();
    // ...
}

// 改造后：
WorkOrderProduce.actions.queryWorkOrder = function (workorderId) {
    var app = Ext.getCmp('WorkOrderProduceViewer');
    var form = app.getComponent('WorkOrderProduceForm');
    var workorderId = form.down('#workorderId').getValue();
    // ...
}
```

**步骤 4：** 使用 `down()` 替代全局查找（在已知父组件时）
```javascript
// 不好：全局查找
Ext.getCmp('workorderId').getValue();

// 好：从表单中查找
var form = Ext.getCmp('WorkOrderProduceForm');
form.down('#workorderId').getValue();

// 更好：使用 getValues() 一次性获取
var formValues = Ext.getCmp('WorkOrderProduceForm').getForm().getValues();
var workorderId = formValues.workorderId;
```

**预期效果：**
- 减少 DOM 遍历次数
- 提升组件查找性能 30%+
- 代码更易维护

---

### 8. **统一异步处理风格**

**涉及文件：**
- [common.js#L83-L117](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/common.js#L83-L117)（queryWmsMLot）
- [common.js#L138-L170](file:///d:/data/MES/gc/core/web/prp/WorkOrderProduce/common.js#L138-L170)（queryMesWoMLot）

**具体措施：**

**步骤 1：** 将 `doSuccessCallback` 改为 Store 事件监听
```javascript
// 原代码（在 queryWmsMLot 中）：
WmsMLotStore.load({
    params: {...},
    doSuccessCallback: function (responseJson, operation, success) {
        if (responseJson.success != null && responseJson.success) {
            // 处理逻辑
        }
    }
});

// 改造后：在 Store 定义时统一添加监听
WorkOrderProduce.stores.wmsMLot = Ext.create('Ext.data.Store', {
    // ... 其他配置
    listeners: {
        load: function(store, records, successful, operation) {
            if (successful) {
                var data = records.map(function(r) { return r.data; });
                WorkOrderProduce.handlers.calculateWmsTotals(data);
            }
        },
        exception: function(proxy, response, operation) {
            MyCim.notify.alert('加载失败：' + operation.getError());
        }
    }
});

// 计算总数的逻辑抽离
WorkOrderProduce.handlers.calculateWmsTotals = function(data) {
    var totalQty = 0;
    var totalSub = 0;
    
    Ext.each(data, function(record) {
        totalSub += parseInt(record.currentSubQty) || 0;
        totalQty += parseInt(record.currentQty) || 0;
    });
    
    Ext.getCmp('totalQty').setValue(totalQty);
    Ext.getCmp('totalSubs').setValue(totalSub);
};
```

**步骤 2：** 简化 `queryWmsMLot` 函数
```javascript
// 原代码（30 行）：
Ext.queryWmsMLot = function () {
    if (Ext.checkWorkOrderOperable()) {
        return;
    }
    WmsMLotStore.removeAll();
    WmsMLotStore.load({
        params: {...},
        doSuccessCallback: function (responseJson, operation, success) {
            // 40 行处理逻辑
        }
    });
}

// 改造后（10 行）：
WorkOrderProduce.actions.queryWmsMLot = function () {
    if (WorkOrderProduce.utils.checkWorkOrderOperable()) {
        return;
    }
    
    var store = WorkOrderProduce.stores.wmsMLot;
    store.removeAll();
    store.load({
        params: {
            workorderId: WorkOrderProduce.utils.getWorkorderId(),
            lotId: Ext.getCmp('obLotId').getValue(),
            materialLotId: Ext.getCmp('obMaterialLotId').getValue(),
            durable: Ext.getCmp('durable').getValue(),
            cpAdvanceRwFlag: WorkOrderProduce.config.cpAdvanceRwFlag
        }
    });
};
```

**步骤 3：** 同样处理 `queryMesWoMLot`

**预期效果：**
- 异步逻辑集中管理
- 消除回调地狱
- 便于统一错误处理

---

## 🟢 低优先级优化措施

### 9. **合并子目录的 JSP 文件**

**涉及文件：**
- `COM/WorkOrderProduce.jsp`
- `FT/WorkOrderProduce.jsp`
- `OIS/WorkOrderProduce.jsp`
- `RC/WorkOrderProduce.jsp`
- `RW/WorkOrderProduce.jsp`
- `FT-AE/WorkOrderProduce.jsp`

**具体措施：**

**步骤 1：** 保留主目录的 `WorkOrderProduce.jsp` 作为模板

**步骤 2：** 修改 JSP，从请求参数获取 `pageCategory`
```jsp
<%
    // 从 URL 参数获取 pageCategory
    String pageCategory = request.getParameter("pageCategory");
    if (pageCategory == null || pageCategory.isEmpty()) {
        pageCategory = "COM"; // 默认值
    }
%>

<script type="text/javascript">
    var pageCategory = '<%= pageCategory %>';
</script>
```

**步骤 3：** 修改后端路由，统一跳转到同一 JSP
```java
// 原代码：
// /mycim2/prp/COM/WorkOrderProduce.jsp
// /mycim2/prp/FT/WorkOrderProduce.jsp

// 改造后：
// /mycim2/prp/WorkOrderProduce.jsp?pageCategory=COM
// /mycim2/prp/WorkOrderProduce.jsp?pageCategory=FT
```

**步骤 4：** 删除子目录的 JSP 文件

**预期效果：**
- 减少 6 个重复文件
- 后续修改只需改一处
- 便于统一版本管理

---

### 10. **提取常量**

**涉及文件：**
- 所有 JS 文件

**具体措施：**

**步骤 1：** 在 `WorkOrderConstant.js` 中定义常量
```javascript
WorkOrderProduce.constants = {
    // URL Action 常量
    ACTIONS: {
        QUERY_WORK_ORDER: 'ReadWorkOrder',
        CREATE_WORK_ORDER: 'CreateWorkOrder',
        UPDATE_WORK_ORDER: 'UpdateWorkOrder',
        QUERY_WMS_MLOT: 'queryWarehouseMaterials',
        QUERY_MES_MLOT: 'QueryReservedMLot',
        BIND_MLOT: 'BindOrderBindMLot',
        UNBIND_MLOT: 'UnReservedMLot',
        IMPORT_MATERIAL: 'importMaterial',
        // ... 其他 action
    },
    
    // 组件 ID 常量
    COMPONENT_IDS: {
        WORK_ORDER_ID: 'workorderId',
        STATUS: 'status',
        PLANNED_QTY: 'plannedQty',
        WORKSHOP: 'workshop',
        WAFER_SOURCE: 'waferSource',
        STORAGE_MODEL: 'storageModel',
        // ... 其他组件 ID
    },
    
    // 工单状态常量
    WORK_ORDER_STATUS: {
        CREATE: 'CREATE',
        CREATED: 'CREATED',
        ISSUED: 'ISSUED',
        CLOSED: 'CLOSED'
    },
    
    // 事务类型常量
    TRANS_TYPE: {
        CREATE: 'CREATED',
        UPDATE: 'CREATED',
        ISSUE: 'ISSUE',
        CLOSE: 'CLOSE',
        DELETE: 'DELETE',
        PLAN_FLAG: 'PLAN_FLAG'
    },
    
    // 车间类型常量
    WORKSHOP: {
        COM: 'COM',
        OIS: 'OIS',
        FT: 'FT',
        RW: 'RW',
        RC: '回收清洗组'
    }
};
```

**步骤 2：** 全局替换硬编码
```javascript
// 原代码：
url: '/mycim2/WorkOrderProduce.do?action=ReadWorkOrder'
if (status !== "" && status !== 'CREATE' && status !== 'CREATED')
params.transType = 'DELETE'

// 改造后：
url: '/mycim2/WorkOrderProduce.do?action=' + WorkOrderProduce.constants.ACTIONS.QUERY_WORK_ORDER
if (status !== "" && status !== WorkOrderProduce.constants.WORK_ORDER_STATUS.CREATE && ...)
params.transType = WorkOrderProduce.constants.TRANS_TYPE.DELETE
```

**预期效果：**
- 消除魔法字符串
- 修改 URL 或状态值时只需改一处
- 支持 IDE 自动补全

---

### 11. **添加 JSDoc 注释**

**涉及文件：**
- `common.js` 中的所有函数

**具体措施：**

**步骤 1：** 为每个函数添加标准注释
```javascript
/**
 * 查询工单信息
 * @param {string} workorderId - 工单ID
 * @returns {Promise} 包含工单数据的 Promise 对象
 * @example
 * WorkOrderProduce.actions.queryWorkOrder('WO20241001001')
 *     .then(function(data) {
 *         console.log('工单数量：' + data.plannedQty);
 *     });
 */
WorkOrderProduce.actions.queryWorkOrder = function(workorderId) {
    // ...
}

/**
 * 创建组合框监听器
 * @param {Object} params - 配置参数
 * @param {string} params.method - 处理方法名称（如 'storageModel', 'bomCategory'）
 * @param {Ext.data.Store} params.pStore - 父级 Store，用于联动加载数据
 * @param {string[]} params.clearFields - 选择时需要清空的字段ID数组
 * @param {Ext.data.Store[]} params.storeToClear - 选择时需要清空的 Store 数组
 * @returns {Object} ExtJS 监听器配置对象
 * @example
 * var listeners = WorkOrderProduce.utils.createComboListeners({
 *     method: 'storageModel',
 *     pStore: WorkOrderProduce.stores.storageModel,
 *     clearFields: ['bomCategory', 'putProductId']
 * });
 */
WorkOrderProduce.utils.createComboListeners = function(params) {
    // ...
}
```

**步骤 2：** 为复杂业务逻辑添加行内注释
```javascript
/**
 * 计算 levelTwoCode（二级代码）
 * 业务规则：
 * - 当 waferSource 为 '16'（新片）时，使用 bomVersion 作为 levelTwoCode
 * - 其他情况（回收片、客供片等），使用表单中手动输入的 levelTwoCode
 */
var levelTwoCode = values.waferSource == '16' 
    ? values.bomVersion 
    : values.levelTwoCode;
```

**预期效果：**
- 新成员快速理解代码
- IDE 提供智能提示
- 便于生成 API 文档

---

## 📋 实施建议

### 推荐实施顺序

```
第一阶段（1-2天）：
  5. 删除测试代码 ⭐ 最简单，立即见效
  6. 优化分页大小 ⭐ 性能提升明显
  
第二阶段（3-5天）：
  1. 命名空间封装 🔧 基础架构改造
  3. Store 工厂函数 🔧 配合阶段一
  
第三阶段（5-7天）：
  2. 重构 initPage 🔧 依赖阶段一的命名空间
  4. 拆分 createComboListeners 🔧 依赖阶段一
  
第四阶段（按需）：
  7. 减少 Ext.getCmp
  8. 统一异步处理
  9. 合并 JSP
  10. 提取常量
  11. 添加注释
```

### 风险控制

1. **每次只改一个优化项**，改完测试通过再改下一个
2. **保留备份**，每次改造前 git commit
3. **编写回归测试**，确保功能不受影响
4. **分模块改造**，先改 `common.js`，再改其他文件

---

以上就是每项优化的**具体实施措施**，包括：
- 涉及的代码位置
- 详细的改造步骤（原代码 vs 改造后代码对比）
- 预期效果
- 实施顺序建议

你可以根据团队实际情况，选择优先级高的项逐步实施。需要我进一步细化某个优化项吗？

