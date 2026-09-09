---
title: 修改打印标签规格
date: 2026-05-03
tags: [AI协作, MES]
---
# 修改打印标签规格

> **Workspace:** d:\data\MES\gc

---

## User
comProcessCardPrintForm.js 帮我看下这个文件打印功能，如果我想把打印的标签的产品规格去掉，改成一体机tray规格，是一个框，也是下拉的，封逃气孔的标签位修改为出货tray规格及tray颜色，也是下拉选，是一个框的，帮我看下怎么写，不要修改文件，给我修改建议我自己改

## User
comProcessCardPrintForm.js 帮我看下这个文件打印功能，如果我想把打印的标签的产品规格去掉，改成一体机tray规格，封逃气孔的标签位修改为出货tray规格及tray颜色，这两个值通过一张配置表获取，配置表是新的，通过配置BOM号，一体机tray，出货tray三个字段，工单获取对应的bom号，然后获取两个tray，bom号在配置表是唯一的，并且三个字段也不重复，帮我看下怎么写，不要修改文件，给我修改建议，我自己改

## Assistant
我来帮你分析这个文件的打印功能，并给出修改建议。首先让我查看一下这个文件的内容。



现在让我查看相关的Grid文件，了解完整的数据结构：



我已经查看完文件内容，给你详细的修改建议：

## 📋 修改建议

### 一、表单字段修改（第48-74行）

**将产品规格字段（第48-74行）替换为以下三个新字段：**

```javascript
// 字段1：一体机tray规格下拉框
{
    xtype: 'combo',
    editable: false,
    mode: 'local',
    columnWidth: 0.25,
    triggerAction: 'all',
    valueField: 'integratedTray',
    displayField: 'integratedTray',
    store: Ext.create('Ext.data.ArrayStore', {
        fields: ['integratedTray', 'shipmentTray', 'trayColor'],
        proxy: {
            queryMask: false,
            type: 'ajax',
            url: '/mycim2/comProcessCardPrint.do?action=getTrayConfig',
            reader: {
                root: 'msg.data'
            }
        }
    }),
    name: 'integratedTraySpec',
    id: 'integratedTraySpec',
    fieldLabel: '一体机tray规格' + i18n_fld_Red_Star,
    listeners: {
        change: function(combo, newValue) {
            // 选择一体机tray时，自动带出出货tray和tray颜色
            var record = combo.findRecordByValue(newValue);
            if (record) {
                Ext.getCmp('shipmentTraySpec').setValue(record.get('shipmentTray'));
                Ext.getCmp('trayColor').setValue(record.get('trayColor'));
            }
        }
    }
},
// 字段2：出货tray规格下拉框（与一体机tray在同一个框）
{
    xtype: 'combo',
    editable: false,
    mode: 'local',
    columnWidth: 0.25,
    triggerAction: 'all',
    valueField: 'shipmentTray',
    displayField: 'shipmentTray',
    store: Ext.create('Ext.data.ArrayStore', {
        fields: ['shipmentTray'],
        data: []  // 数据由一体机tray选择后动态加载
    }),
    name: 'shipmentTraySpec',
    id: 'shipmentTraySpec',
    fieldLabel: '出货tray规格' + i18n_fld_Red_Star
},
// 字段3：tray颜色下拉框
{
    xtype: 'combo',
    editable: false,
    mode: 'local',
    columnWidth: 0.25,
    triggerAction: 'all',
    valueField: 'trayColor',
    displayField: 'trayColor',
    store: Ext.create('Ext.data.ArrayStore', {
        fields: ['trayColor'],
        data: []
    }),
    name: 'trayColor',
    id: 'trayColor',
    fieldLabel: 'tray颜色'
}
```

### 二、重置功能修改（第108-113行）

**修改doClear函数：**

```javascript
Ext.doClear = function() {
    Ext.getCmp('outerOrderNo').setValue('');
    Ext.getCmp('lotId').setValue('');
    Ext.getCmp('integratedTraySpec').setValue('');
    Ext.getCmp('shipmentTraySpec').setValue('');
    Ext.getCmp('trayColor').setValue('');
    store1.removeAll();
};
```

### 三、打印功能修改（第167-193行）

**修改doPrintLabel函数：**

```javascript
Ext.doPrintLabel = function(){
    var integratedTraySpec = Ext.getCmp('integratedTraySpec').getValue();
    var shipmentTraySpec = Ext.getCmp('shipmentTraySpec').getValue();
    var trayColor = Ext.getCmp('trayColor').getValue();
    
    // 验证必填字段
    if(Ext.isEmpty(integratedTraySpec)){
        Ext.Msg.alert(i18n_msg_prompt, '请选择一体机tray规格');
        return;
    }
    if(Ext.isEmpty(shipmentTraySpec)){
        Ext.Msg.alert(i18n_msg_prompt, '请选择出货tray规格');
        return;
    }
    
    var records = Ext.getCmp("ComProcessCardPrintGrid").getSelectionModel().getSelection();
    if(records.length == 0){
        Ext.Msg.alert(i18n_msg_prompt, i18n_msg_PLEASE_CHECK_AT_LEAST_ONE_PIECE_OF_DATA);
        return;
    }
    
    var pageNum = 1;
    var pageNums = records.length;
    Ext.Array.each(records, function(record) {
        var printInfo = "lotId=" + record.data.lotId + 
            ";compomentMaterialId=" + record.data.compomentMaterialId + 
            ";lotQty=" + record.data.qty1 + 
            ";plannedQty=" + record.data.plannedQty + 
            ";productId=" + record.data.productId + 
            ";integratedTraySpec=" + integratedTraySpec +      // 替换productSpec
            ";shipmentTraySpec=" + shipmentTraySpec +           // 新增
            ";trayColor=" + trayColor +                         // 新增
            ";sealFlag=" + record.data.sealFlag + 
            ";sencondCode=" + record.data.levelTwoCode + 
            ";workOrderId=" + record.data.outerOrderNo + 
            ";waferLevel=" + record.data.waferLevel + 
            ";comments=" + record.data.comments + 
            ";pageNum=" + pageNum + 
            ";pageNums=" + pageNums;
        bartenderPrint(printInfo, record.data.printTemplateName, 1);
        pageNum++;
    });
}
```

### 四、工单号输入联动（第24-31行）

**修改工单号的keyup事件，输入工单号后自动加载tray配置：**

```javascript
listeners: {
    keyup: function(cmp, e){
        if (e.getKey() == 13) {
            var workOrderId = cmp.getValue();
            if (workOrderId) {
                // 先通过工单号获取BOM号
                Ext.Ajax.request({
                    url: '/mycim2/comProcessCardPrint.do?action=getBomByWorkOrder',
                    params: { workOrderId: workOrderId },
                    success: function(response) {
                        var result = Ext.JSON.decode(response.responseText);
                        if (result.success && result.msg.bomNo) {
                            // 根据BOM号加载tray配置
                            Ext.getCmp('integratedTraySpec').getStore().load({
                                params: { bomNo: result.msg.bomNo }
                            });
                        }
                    }
                });
            }
            Ext.doSearch();
        }
    }
}
```

### 五、后端接口实现

需要新增以下后端接口：

**1. 根据工单号获取BOM号：**
```java
@RequestMapping(params = "action=getBomByWorkOrder")
@ResponseBody
public Map<String, Object> getBomByWorkOrder(String workOrderId) {
    // 查询工单对应的BOM号
    String bomNo = workOrderService.getBomNoByWorkOrder(workOrderId);
    Map<String, Object> result = new HashMap<>();
    result.put("success", true);
    result.put("msg", Map.of("bomNo", bomNo));
    return result;
}
```

**2. 根据BOM号获取tray配置：**
```java
@RequestMapping(params = "action=getTrayConfig")
@ResponseBody
public Map<String, Object> getTrayConfig(String bomNo) {
    // 从配置表查询：SELECT integrated_tray, shipment_tray, tray_color 
    // FROM tray_config WHERE bom_no = #{bomNo}
    TrayConfig config = trayConfigService.getByBomNo(bomNo);
    
    Map<String, Object> result = new HashMap<>();
    result.put("success", true);
    
    List<Map<String, String>> data = new ArrayList<>();
    Map<String, String> item = new HashMap<>();
    item.put("integratedTray", config.getIntegratedTray());
    item.put("shipmentTray", config.getShipmentTray());
    item.put("trayColor", config.getTrayColor());
    data.add(item);
    
    result.put("msg", Map.of("data", data));
    return result;
}
```

### 六、配置表设计

```sql
CREATE TABLE tray_config (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    bom_no VARCHAR(100) NOT NULL COMMENT 'BOM号',
    integrated_tray VARCHAR(100) NOT NULL COMMENT '一体机tray规格',
    shipment_tray VARCHAR(100) NOT NULL COMMENT '出货tray规格',
    tray_color VARCHAR(50) COMMENT 'tray颜色',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT uk_bom_no UNIQUE (bom_no),
    CONSTRAINT uk_tray_combo UNIQUE (bom_no, integrated_tray, shipment_tray)
) COMMENT='Tray配置表';
```

### 七、注意事项

1. **数据联动逻辑**：选择一体机tray时，自动带出对应的出货tray和tray颜色
2. **必填校验**：一体机tray规格和出货tray规格为必填
3. **标签模板**：需要同步修改BarTender打印模板，将`productSpec`替换为新字段
4. **数据加载时机**：工单号回车后触发BOM号查询，再加载tray配置
5. **空值处理**：如果配置表没有对应BOM号的配置，需要提示用户

按照以上步骤修改即可实现你的需求。

## User
表单字段修改（第48-74行）可以都去掉了，产品规格不要了，然后另外两个是按照配置表来的

## Assistant
明白了，让我重新给你修改建议：

## 📋 修改建议（更新版）

### 一、表单字段修改（第48-74行）

**删除第48-74行的产品规格字段，替换为两个新字段：**

```javascript
// 替换原来的产品规格字段（第48-74行）
{
    xtype : 'combo',
    editable : false,
    mode : 'local',
    columnWidth : 0.25,
    triggerAction : 'all',
    valueField : 'integratedTray',
    displayField : 'integratedTray',
    store : Ext.create('Ext.data.ArrayStore', {
        fields : ['integratedTray', 'shipmentTray', 'trayColor'],
        data : []  // 数据通过工单号查询BOM后动态加载
    }),
    name : 'integratedTraySpec',
    id : 'integratedTraySpec',
    fieldLabel : '一体机tray规格' + i18n_fld_Red_Star,
    listeners : {
        change : function(combo, newValue) {
            var record = combo.findRecordByValue(newValue);
            if (record) {
                Ext.getCmp('shipmentTraySpec').setValue(record.get('shipmentTray'));
                Ext.getCmp('trayColor').setValue(record.get('trayColor'));
            }
        }
    }
},
{
    xtype : 'combo',
    editable : false,
    mode : 'local',
    columnWidth : 0.25,
    triggerAction : 'all',
    valueField : 'shipmentTray',
    displayField : 'shipmentTray',
    store : Ext.create('Ext.data.ArrayStore', {
        fields : ['shipmentTray'],
        data : []  // 数据通过工单号查询后动态加载
    }),
    name : 'shipmentTraySpec',
    id : 'shipmentTraySpec',
    fieldLabel : '出货tray规格' + i18n_fld_Red_Star
}
```

### 二、工单号联动加载tray配置（第24-31行）

**修改工单号输入框的keyup事件：**

```javascript
listeners : {
    keyup : function(cmp, e) {
        if (e.getKey() == 13) {
            var workOrderId = cmp.getValue();
            if (workOrderId) {
                Ext.Ajax.request({
                    url : '/mycim2/comProcessCardPrint.do?action=getTrayConfigByWorkOrder',
                    params : { workOrderId : workOrderId },
                    success : function(response) {
                        var result = Ext.JSON.decode(response.responseText);
                        if (result.success && result.msg.data) {
                            // 加载一体机tray规格下拉数据
                            var trayStore = Ext.getCmp('integratedTraySpec').getStore();
                            trayStore.loadData(result.msg.data);
                            
                            // 如果有多个配置，加载出货tray规格
                            var shipmentStore = Ext.getCmp('shipmentTraySpec').getStore();
                            shipmentStore.loadData(result.msg.shipmentTrayData);
                        }
                    }
                });
            }
            Ext.doSearch();
        }
    }
}
```

### 三、重置功能修改（第108-113行）

```javascript
Ext.doClear = function() {
    Ext.getCmp('outerOrderNo').setValue('');
    Ext.getCmp('lotId').setValue('');
    Ext.getCmp('integratedTraySpec').setValue('');
    Ext.getCmp('shipmentTraySpec').setValue('');
    // 清空store数据
    Ext.getCmp('integratedTraySpec').getStore().removeAll();
    Ext.getCmp('shipmentTraySpec').getStore().removeAll();
    store1.removeAll();
};
```

### 四、打印功能修改（第167-193行）

```javascript
Ext.doPrintLabel = function() {
    var integratedTraySpec = Ext.getCmp('integratedTraySpec').getValue();
    var shipmentTraySpec = Ext.getCmp('shipmentTraySpec').getValue();
    
    if (Ext.isEmpty(integratedTraySpec)) {
        Ext.Msg.alert(i18n_msg_prompt, '请先输入工单号加载tray配置');
        return;
    }
    if (Ext.isEmpty(shipmentTraySpec)) {
        Ext.Msg.alert(i18n_msg_prompt, '请选择出货tray规格');
        return;
    }
    
    var records = Ext.getCmp("ComProcessCardPrintGrid").getSelectionModel().getSelection();
    if (records.length == 0) {
        Ext.Msg.alert(i18n_msg_prompt, i18n_msg_PLEASE_CHECK_AT_LEAST_ONE_PIECE_OF_DATA);
        return;
    }
    
    var pageNum = 1;
    var pageNums = records.length;
    Ext.Array.each(records, function(record) {
        // 获取tray颜色（从一体机tray选中的record中获取）
        var trayColor = '';
        var trayRecord = Ext.getCmp('integratedTraySpec').findRecordByValue(integratedTraySpec);
        if (trayRecord) {
            trayColor = trayRecord.get('trayColor');
        }
        
        var printInfo = "lotId=" + record.data.lotId + 
            ";compomentMaterialId=" + record.data.compomentMaterialId + 
            ";lotQty=" + record.data.qty1 + 
            ";plannedQty=" + record.data.plannedQty + 
            ";productId=" + record.data.productId + 
            ";integratedTraySpec=" + integratedTraySpec + 
            ";shipmentTraySpec=" + shipmentTraySpec + 
            ";trayColor=" + trayColor + 
            ";sealFlag=" + record.data.sealFlag + 
            ";sencondCode=" + record.data.levelTwoCode + 
            ";workOrderId=" + record.data.outerOrderNo + 
            ";waferLevel=" + record.data.waferLevel + 
            ";comments=" + record.data.comments + 
            ";pageNum=" + pageNum + 
            ";pageNums=" + pageNums;
        bartenderPrint(printInfo, record.data.printTemplateName, 1);
        pageNum++;
    });
}
```

### 五、后端接口实现

**新增一个接口即可，根据工单号获取tray配置：**

```java
@RequestMapping(params = "action=getTrayConfigByWorkOrder")
@ResponseBody
public Map<String, Object> getTrayConfigByWorkOrder(String workOrderId) {
    // 1. 根据工单号获取BOM号
    String bomNo = workOrderService.getBomNoByWorkOrder(workOrderId);
    
    // 2. 根据BOM号从配置表获取tray配置
    TrayConfig config = trayConfigService.getByBomNo(bomNo);
    
    Map<String, Object> result = new HashMap<>();
    result.put("success", true);
    
    // 返回一体机tray数据
    List<Map<String, String>> trayData = new ArrayList<>();
    Map<String, String> item = new HashMap<>();
    item.put("integratedTray", config.getIntegratedTray());
    item.put("shipmentTray", config.getShipmentTray());
    item.put("trayColor", config.getTrayColor());
    trayData.add(item);
    
    // 如果出货tray有多个选项，返回多个；如果只有一个，和上面一样
    List<Map<String, String>> shipmentData = new ArrayList<>();
    Map<String, String> shipmentItem = new HashMap<>();
    shipmentItem.put("shipmentTray", config.getShipmentTray());
    shipmentData.add(shipmentItem);
    
    Map<String, Object> msg = new HashMap<>();
    msg.put("data", trayData);
    msg.put("shipmentTrayData", shipmentData);
    
    result.put("msg", msg);
    return result;
}
```

### 六、配置表SQL

```sql
CREATE TABLE tray_config (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    bom_no VARCHAR(100) NOT NULL UNIQUE COMMENT 'BOM号',
    integrated_tray VARCHAR(100) NOT NULL COMMENT '一体机tray规格',
    shipment_tray VARCHAR(100) NOT NULL COMMENT '出货tray规格',
    tray_color VARCHAR(50) COMMENT 'tray颜色',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) COMMENT='Tray配置表';
```

### 七、核心逻辑说明

1. **删除产品规格字段** - 不再需要
2. **工单号回车触发** - 自动查询BOM号→查询配置表→加载两个下拉框
3. **一体机tray选择联动** - 选择后自动带出出货tray规格和tray颜色
4. **打印参数传递** - 将`productSpec`替换为`integratedTraySpec`、`shipmentTraySpec`、`trayColor`三个新参数

按照这个方案修改即可！

## User
也不需要加载下拉框了，打印的时候带出到标签上就行

## Assistant
明白了，那更简单！只需要在打印时从配置表获取数据并传递到标签即可。

## 📋 修改建议（最终版）

### 一、表单字段修改（第48-74行）

**直接删除第48-74行的产品规格字段，整个items数组只保留前两个字段：**

```javascript
items : [{
    xtype : "textfield",
    fieldLabel : i18n_fld_WorkOrderId,
    id : 'outerOrderNo',
    name : 'outerOrderNo',
    targetIds : 'outerOrderNo',
    columnWidth : 0.24,
    enableKeyEvents : true,
    listeners : {
        keyup : function(cmp, e) {
            if (e.getKey() == 13) {
                Ext.doSearch();
            }
        }
    }
}, {
    xtype : "textfield",
    fieldLabel : i18n_msg_Lot_ID,
    id : 'lotId',
    name : 'lotId',
    targetIds : 'lotId',
    columnWidth : 0.24,
    enableKeyEvents : true,
    listeners : {
        keyup : function(cmp, e) {
            if (e.getKey() == 13) {
                Ext.doSearch();
            }
        }
    }
}]
```

### 二、重置功能修改（第108-113行）

```javascript
Ext.doClear = function() {
    Ext.getCmp('outerOrderNo').setValue('');
    Ext.getCmp('lotId').setValue('');
    store1.removeAll();
};
```

### 三、打印功能修改（第167-193行）

**修改doPrintLabel函数，在打印时查询配置表获取tray数据：**

```javascript
Ext.doPrintLabel = function() {
    var records = Ext.getCmp("ComProcessCardPrintGrid").getSelectionModel().getSelection();
    if (records.length == 0) {
        Ext.Msg.alert(i18n_msg_prompt, i18n_msg_PLEASE_CHECK_AT_LEAST_ONE_PIECE_OF_DATA);
        return;
    }
    
    var outerOrderNo = Ext.getCmp('outerOrderNo').getValue();
    if (Ext.isEmpty(outerOrderNo)) {
        Ext.Msg.alert(i18n_msg_prompt, '请先输入工单号');
        return;
    }
    
    // 打印前先获取tray配置
    Ext.Ajax.request({
        url : '/mycim2/comProcessCardPrint.do?action=getTrayConfigForPrint',
        params : { workOrderId : outerOrderNo },
        success : function(response) {
            var result = Ext.JSON.decode(response.responseText);
            if (result.success && result.msg) {
                var integratedTray = result.msg.integratedTray || '';
                var shipmentTray = result.msg.shipmentTray || '';
                var trayColor = result.msg.trayColor || '';
                
                var pageNum = 1;
                var pageNums = records.length;
                Ext.Array.each(records, function(record) {
                    var printInfo = "lotId=" + record.data.lotId + 
                        ";compomentMaterialId=" + record.data.compomentMaterialId + 
                        ";lotQty=" + record.data.qty1 + 
                        ";plannedQty=" + record.data.plannedQty + 
                        ";productId=" + record.data.productId + 
                        ";integratedTraySpec=" + integratedTray + 
                        ";shipmentTraySpec=" + shipmentTray + 
                        ";trayColor=" + trayColor + 
                        ";sealFlag=" + record.data.sealFlag + 
                        ";sencondCode=" + record.data.levelTwoCode + 
                        ";workOrderId=" + record.data.outerOrderNo + 
                        ";waferLevel=" + record.data.waferLevel + 
                        ";comments=" + record.data.comments + 
                        ";pageNum=" + pageNum + 
                        ";pageNums=" + pageNums;
                    bartenderPrint(printInfo, record.data.printTemplateName, 1);
                    pageNum++;
                });
            } else {
                Ext.Msg.alert(i18n_msg_prompt, '未找到该工单对应的tray配置');
            }
        },
        failure : function() {
            Ext.Msg.alert(i18n_msg_prompt, '获取tray配置失败');
        }
    });
}
```

### 四、后端接口实现

**新增一个接口：**

```java
@RequestMapping(params = "action=getTrayConfigForPrint")
@ResponseBody
public Map<String, Object> getTrayConfigForPrint(String workOrderId) {
    // 1. 根据工单号获取BOM号
    String bomNo = workOrderService.getBomNoByWorkOrder(workOrderId);
    
    // 2. 根据BOM号从配置表获取tray配置
    TrayConfig config = trayConfigService.getByBomNo(bomNo);
    
    Map<String, Object> result = new HashMap<>();
    
    if (config == null) {
        result.put("success", false);
        result.put("msg", "未找到tray配置");
    } else {
        result.put("success", true);
        Map<String, String> msg = new HashMap<>();
        msg.put("integratedTray", config.getIntegratedTray());
        msg.put("shipmentTray", config.getShipmentTray());
        msg.put("trayColor", config.getTrayColor());
        result.put("msg", msg);
    }
    
    return result;
}
```

### 五、核心逻辑

1. **表单简化** - 只保留工单号和批次号两个输入框，去掉产品规格
2. **打印时查询** - 点击打印按钮时，通过工单号查询BOM→查询配置表获取tray数据
3. **标签传参** - 将查询到的`integratedTray`、`shipmentTray`、`trayColor`三个值直接传入打印信息
4. **无需下拉框** - 用户不需要手动选择，系统自动从配置表获取

这样修改最简单，只需改前端打印函数和新增一个后端接口即可！

## User
你先帮我生成一个关于一体机tray和出货tray和BOM物料号之间的配置表的完整的一套代码吗

## Assistant
我来帮你生成一个完整的配置表代码，包括数据库表、实体类、DAO、Service和Controller。

首先让我了解一下项目的现有结构：



现在让我看看现有DAO和model的结构：



现在让我查看一下现有的model和数据库表结构，以便更好地创建新的配置表：



好的，我了解了项目结构。现在我来生成一个完整的Tray配置表的代码，包括数据库SQL、Model、DAO、Service和Controller。

## 完整代码生成

### 1. 数据库建表SQL

```sql
CREATE TABLE TRAY_CONFIG (
    CONFIG_RRN NUMBER(19) PRIMARY KEY,
    BOM_NO VARCHAR2(100) NOT NULL,
    INTEGRATED_TRAY VARCHAR2(100) NOT NULL,
    SHIPMENT_TRAY VARCHAR2(100) NOT NULL,
    TRAY_COLOR VARCHAR2(50),
    CREATE_USER VARCHAR2(50),
    CREATE_TIME TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UPDATE_USER VARCHAR2(50),
    UPDATE_TIME TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT UK_BOM_NO UNIQUE (BOM_NO)
);

COMMENT ON TABLE TRAY_CONFIG IS 'Tray配置表';
COMMENT ON COLUMN TRAY_CONFIG.CONFIG_RRN IS '配置主键';
COMMENT ON COLUMN TRAY_CONFIG.BOM_NO IS 'BOM物料号';
COMMENT ON COLUMN TRAY_CONFIG.INTEGRATED_TRAY IS '一体机tray规格';
COMMENT ON COLUMN TRAY_CONFIG.SHIPMENT_TRAY IS '出货tray规格';
COMMENT ON COLUMN TRAY_CONFIG.TRAY_COLOR IS 'tray颜色';

CREATE SEQUENCE TRAY_CONFIG_SEQ
START WITH 1
INCREMENT BY 1
NOCACHE
NOCYCLE;
```

### 2. Model实体类

文件路径：`d:\data\MES\gc\core\src\valueobject\src\com\mycim\prp\model\TrayConfig.java`

```java
package com.mycim.prp.model;

import java.io.Serializable;
import java.util.Date;

import javax.persistence.Column;
import javax.persistence.Id;
import javax.persistence.Table;

import lombok.Data;
import lombok.EqualsAndHashCode;

@Table(name = "TRAY_CONFIG")
@EqualsAndHashCode(callSuper = false)
@Data
public class TrayConfig implements Serializable {

    private static final long serialVersionUID = 1L;

    @Id
    @Column(name = "CONFIG_RRN")
    private Long configRrn;

    @Column(name = "BOM_NO")
    private String bomNo;

    @Column(name = "INTEGRATED_TRAY")
    private String integratedTray;

    @Column(name = "SHIPMENT_TRAY")
    private String shipmentTray;

    @Column(name = "TRAY_COLOR")
    private String trayColor;

    @Column(name = "CREATE_USER")
    private String createUser;

    @Column(name = "CREATE_TIME")
    private Date createTime;

    @Column(name = "UPDATE_USER")
    private String updateUser;

    @Column(name = "UPDATE_TIME")
    private Date updateTime;

    public TrayConfig() {
    }

    public TrayConfig(String bomNo, String integratedTray, String shipmentTray, String trayColor) {
        this.bomNo = bomNo;
        this.integratedTray = integratedTray;
        this.shipmentTray = shipmentTray;
        this.trayColor = trayColor;
    }
}
```

### 3. DAO层

**接口文件路径：** `d:\data\MES\gc\core\src\service\src\com\mycim\dao\prp\TrayConfigDao.java`

```java
package com.mycim.dao.prp;

import java.util.List;
import java.util.Map;

public interface TrayConfigDao {

    List<Map<String, Object>> getTrayConfigByBomNo(String bomNo);

    Map<String, Object> getTrayConfigForPrint(String workOrderId);

    int insertTrayConfig(Map<String, Object> config);

    int updateTrayConfig(Map<String, Object> config);

    int deleteTrayConfig(Long configRrn);

    List<Map<String, Object>> queryTrayConfigList(Map<String, Object> params);
}
```

**实现类文件路径：** `d:\data\MES\gc\core\src\ejb\prp\src\com\mycim\prp\dao\TrayConfigDaoImpl.java`

```java
package com.mycim.prp.dao;

import java.util.List;
import java.util.Map;

import org.springframework.stereotype.Repository;

import com.mycim.bas.dao.BaseDAOImpl;
import com.mycim.dao.prp.TrayConfigDao;

@Repository
public class TrayConfigDaoImpl extends BaseDAOImpl implements TrayConfigDao {

    @Override
    public List<Map<String, Object>> getTrayConfigByBomNo(String bomNo) {
        String sql = "SELECT CONFIG_RRN, BOM_NO, INTEGRATED_TRAY, SHIPMENT_TRAY, TRAY_COLOR "
                + "FROM TRAY_CONFIG WHERE BOM_NO = ?";
        Object[] args = { bomNo };
        return this.useSqlQuery(sql, Map.class, args);
    }

    @Override
    public Map<String, Object> getTrayConfigForPrint(String workOrderId) {
        String sql = "SELECT tc.BOM_NO, tc.INTEGRATED_TRAY, tc.SHIPMENT_TRAY, tc.TRAY_COLOR "
                + "FROM TRAY_CONFIG tc "
                + "INNER JOIN WORK_ORDER wo ON tc.BOM_NO = wo.BOM_NO "
                + "WHERE wo.WORK_ORDER_ID = ?";
        Object[] args = { workOrderId };
        List<Map<String, Object>> list = this.useSqlQuery(sql, Map.class, args);
        if (list != null && !list.isEmpty()) {
            return list.get(0);
        }
        return null;
    }

    @Override
    public int insertTrayConfig(Map<String, Object> config) {
        String sql = "INSERT INTO TRAY_CONFIG (CONFIG_RRN, BOM_NO, INTEGRATED_TRAY, SHIPMENT_TRAY, TRAY_COLOR, "
                + "CREATE_USER, CREATE_TIME, UPDATE_USER, UPDATE_TIME) "
                + "VALUES (TRAY_CONFIG_SEQ.NEXTVAL, ?, ?, ?, ?, ?, SYSDATE, ?, SYSDATE)";
        Object[] args = {
                config.get("bomNo"),
                config.get("integratedTray"),
                config.get("shipmentTray"),
                config.get("trayColor"),
                config.get("createUser"),
                config.get("updateUser")
        };
        return this.useSqlUpdate(sql, args);
    }

    @Override
    public int updateTrayConfig(Map<String, Object> config) {
        String sql = "UPDATE TRAY_CONFIG SET BOM_NO = ?, INTEGRATED_TRAY = ?, SHIPMENT_TRAY = ?, "
                + "TRAY_COLOR = ?, UPDATE_USER = ?, UPDATE_TIME = SYSDATE WHERE CONFIG_RRN = ?";
        Object[] args = {
                config.get("bomNo"),
                config.get("integratedTray"),
                config.get("shipmentTray"),
                config.get("trayColor"),
                config.get("updateUser"),
                config.get("configRrn")
        };
        return this.useSqlUpdate(sql, args);
    }

    @Override
    public int deleteTrayConfig(Long configRrn) {
        String sql = "DELETE FROM TRAY_CONFIG WHERE CONFIG_RRN = ?";
        Object[] args = { configRrn };
        return this.useSqlUpdate(sql, args);
    }

    @Override
    public List<Map<String, Object>> queryTrayConfigList(Map<String, Object> params) {
        StringBuilder sql = new StringBuilder();
        sql.append("SELECT CONFIG_RRN, BOM_NO, INTEGRATED_TRAY, SHIPMENT_TRAY, TRAY_COLOR, ");
        sql.append("CREATE_USER, CREATE_TIME, UPDATE_USER, UPDATE_TIME ");
        sql.append("FROM TRAY_CONFIG WHERE 1=1");

        if (params.get("bomNo") != null && !params.get("bomNo").toString().isEmpty()) {
            sql.append(" AND BOM_NO LIKE ?");
        }
        if (params.get("integratedTray") != null && !params.get("integratedTray").toString().isEmpty()) {
            sql.append(" AND INTEGRATED_TRAY LIKE ?");
        }

        sql.append(" ORDER BY CREATE_TIME DESC");

        return this.useSqlQuery(sql.toString(), Map.class, new Object[] {
                params.get("bomNo") != null ? "%" + params.get("bomNo") + "%" : null,
                params.get("integratedTray") != null ? "%" + params.get("integratedTray") + "%" : null
        });
    }
}
```

### 4. Service层

**接口文件路径：** `d:\data\MES\gc\core\src\service\src\com\mycim\service\prp\TrayConfigService.java`

```java
package com.mycim.service.prp;

import java.util.List;
import java.util.Map;

public interface TrayConfigService {

    Map<String, Object> getTrayConfigForPrint(String workOrderId);

    List<Map<String, Object>> getTrayConfigByBomNo(String bomNo);

    Map<String, Object> saveTrayConfig(Map<String, Object> config, String userName);

    Map<String, Object> updateTrayConfig(Map<String, Object> config, String userName);

    Map<String, Object> deleteTrayConfig(Long configRrn);

    List<Map<String, Object>> queryTrayConfigList(Map<String, Object> params);
}
```

**实现类文件路径：** `d:\data\MES\gc\core\src\ejb\prp\src\com\mycim\prp\service\impl\TrayConfigServiceImpl.java`

```java
package com.mycim.prp.service.impl;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.mycim.dao.prp.TrayConfigDao;
import com.mycim.service.prp.TrayConfigService;
import com.mycim.util.exception.MyCimException;

@Service
public class TrayConfigServiceImpl implements TrayConfigService {

    @Autowired
    private TrayConfigDao trayConfigDao;

    @Override
    public Map<String, Object> getTrayConfigForPrint(String workOrderId) {
        try {
            Map<String, Object> result = trayConfigDao.getTrayConfigForPrint(workOrderId);
            if (result == null) {
                throw new MyCimException("未找到该工单对应的BOM号或Tray配置");
            }
            return result;
        } catch (MyCimException e) {
            throw e;
        } catch (Exception e) {
            throw new MyCimException("获取Tray配置失败: " + e.getMessage());
        }
    }

    @Override
    public List<Map<String, Object>> getTrayConfigByBomNo(String bomNo) {
        try {
            return trayConfigDao.getTrayConfigByBomNo(bomNo);
        } catch (Exception e) {
            throw new MyCimException("查询Tray配置失败: " + e.getMessage());
        }
    }

    @Override
    public Map<String, Object> saveTrayConfig(Map<String, Object> config, String userName) {
        try {
            // 检查BOM号是否已存在
            List<Map<String, Object>> existList = trayConfigDao.getTrayConfigByBomNo(config.get("bomNo").toString());
            if (existList != null && !existList.isEmpty()) {
                throw new MyCimException("该BOM号的配置已存在，请使用更新功能");
            }

            config.put("createUser", userName);
            config.put("updateUser", userName);

            int result = trayConfigDao.insertTrayConfig(config);
            Map<String, Object> response = new HashMap<>();
            response.put("success", result > 0);
            response.put("msg", result > 0 ? "保存成功" : "保存失败");
            return response;
        } catch (MyCimException e) {
            throw e;
        } catch (Exception e) {
            throw new MyCimException("保存Tray配置失败: " + e.getMessage());
        }
    }

    @Override
    public Map<String, Object> updateTrayConfig(Map<String, Object> config, String userName) {
        try {
            if (config.get("configRrn") == null) {
                throw new MyCimException("配置主键不能为空");
            }

            config.put("updateUser", userName);

            int result = trayConfigDao.updateTrayConfig(config);
            Map<String, Object> response = new HashMap<>();
            response.put("success", result > 0);
            response.put("msg", result > 0 ? "更新成功" : "更新失败");
            return response;
        } catch (MyCimException e) {
            throw e;
        } catch (Exception e) {
            throw new MyCimException("更新Tray配置失败: " + e.getMessage());
        }
    }

    @Override
    public Map<String, Object> deleteTrayConfig(Long configRrn) {
        try {
            if (configRrn == null) {
                throw new MyCimException("配置主键不能为空");
            }

            int result = trayConfigDao.deleteTrayConfig(configRrn);
            Map<String, Object> response = new HashMap<>();
            response.put("success", result > 0);
            response.put("msg", result > 0 ? "删除成功" : "删除失败");
            return response;
        } catch (MyCimException e) {
            throw e;
        } catch (Exception e) {
            throw new MyCimException("删除Tray配置失败: " + e.getMessage());
        }
    }

    @Override
    public List<Map<String, Object>> queryTrayConfigList(Map<String, Object> params) {
        try {
            return trayConfigDao.queryTrayConfigList(params);
        } catch (Exception e) {
            throw new MyCimException("查询Tray配置列表失败: " + e.getMessage());
        }
    }
}
```

### 5. Controller层

**文件路径：** `d:\data\MES\gc\core\src\model\src\com\mycim\webapp\controllers\prp\TrayConfigController.java`

```java
package com.mycim.webapp.controllers.prp;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;

import com.mycim.service.prp.TrayConfigService;
import com.mycim.util.exception.MyCimException;

@Controller
@RequestMapping("/TrayConfigController")
public class TrayConfigController {

    protected Logger log = LoggerFactory.getLogger(this.getClass().getName());

    @Autowired
    private TrayConfigService trayConfigService;

    @RequestMapping(value = "/getTrayConfigForPrint", method = RequestMethod.POST, produces = "application/json; charset=utf-8")
    @ResponseBody
    public Map<String, Object> getTrayConfigForPrint(@RequestBody Map<String, String> params) {
        Map<String, Object> result = new HashMap<>();
        try {
            String workOrderId = params.get("workOrderId");
            if (workOrderId == null || workOrderId.trim().isEmpty()) {
                result.put("success", false);
                result.put("msg", "工单号不能为空");
                return result;
            }

            Map<String, Object> config = trayConfigService.getTrayConfigForPrint(workOrderId);
            result.put("success", true);
            result.put("msg", config);
        } catch (MyCimException e) {
            result.put("success", false);
            result.put("msg", e.getMessage());
        } catch (Exception e) {
            log.error("获取Tray配置失败", e);
            result.put("success", false);
            result.put("msg", "系统异常");
        }
        return result;
    }

    @RequestMapping(value = "/getTrayConfigByBomNo", method = RequestMethod.POST, produces = "application/json; charset=utf-8")
    @ResponseBody
    public Map<String, Object> getTrayConfigByBomNo(@RequestBody Map<String, String> params) {
        Map<String, Object> result = new HashMap<>();
        try {
            String bomNo = params.get("bomNo");
            if (bomNo == null || bomNo.trim().isEmpty()) {
                result.put("success", false);
                result.put("msg", "BOM号不能为空");
                return result;
            }

            List<Map<String, Object>> configList = trayConfigService.getTrayConfigByBomNo(bomNo);
            result.put("success", true);
            result.put("msg", configList);
        } catch (Exception e) {
            log.error("查询Tray配置失败", e);
            result.put("success", false);
            result.put("msg", "系统异常");
        }
        return result;
    }

    @RequestMapping(value = "/saveTrayConfig", method = RequestMethod.POST, produces = "application/json; charset=utf-8")
    @ResponseBody
    public Map<String, Object> saveTrayConfig(@RequestBody Map<String, Object> params) {
        Map<String, Object> result = new HashMap<>();
        try {
            String userName = params.get("userName") != null ? params.get("userName").toString() : "SYSTEM";
            result = trayConfigService.saveTrayConfig(params, userName);
        } catch (MyCimException e) {
            result.put("success", false);
            result.put("msg", e.getMessage());
        } catch (Exception e) {
            log.error("保存Tray配置失败", e);
            result.put("success", false);
            result.put("msg", "系统异常");
        }
        return result;
    }

    @RequestMapping(value = "/updateTrayConfig", method = RequestMethod.POST, produces = "application/json; charset=utf-8")
    @ResponseBody
    public Map<String, Object> updateTrayConfig(@RequestBody Map<String, Object> params) {
        Map<String, Object> result = new HashMap<>();
        try {
            String userName = params.get("userName") != null ? params.get("userName").toString() : "SYSTEM";
            result = trayConfigService.updateTrayConfig(params, userName);
        } catch (MyCimException e) {
            result.put("success", false);
            result.put("msg", e.getMessage());
        } catch (Exception e) {
            log.error("更新Tray配置失败", e);
            result.put("success", false);
            result.put("msg", "系统异常");
        }
        return result;
    }

    @RequestMapping(value = "/deleteTrayConfig", method = RequestMethod.POST, produces = "application/json; charset=utf-8")
    @ResponseBody
    public Map<String, Object> deleteTrayConfig(@RequestBody Map<String, Object> params) {
        Map<String, Object> result = new HashMap<>();
        try {
            Long configRrn = params.get("configRrn") != null ? Long.valueOf(params.get("configRrn").toString()) : null;
            result = trayConfigService.deleteTrayConfig(configRrn);
        } catch (MyCimException e) {
            result.put("success", false);
            result.put("msg", e.getMessage());
        } catch (Exception e) {
            log.error("删除Tray配置失败", e);
            result.put("success", false);
            result.put("msg", "系统异常");
        }
        return result;
    }

    @RequestMapping(value = "/queryTrayConfigList", method = RequestMethod.POST, produces = "application/json; charset=utf-8")
    @ResponseBody
    public Map<String, Object> queryTrayConfigList(@RequestBody Map<String, Object> params) {
        Map<String, Object> result = new HashMap<>();
        try {
            List<Map<String, Object>> list = trayConfigService.queryTrayConfigList(params);
            result.put("success", true);
            result.put("msg", list);
        } catch (Exception e) {
            log.error("查询Tray配置列表失败", e);
            result.put("success", false);
            result.put("msg", "系统异常");
        }
        return result;
    }
}
```

---

## 📝 使用说明

### 文件清单：
1. **数据库SQL** - 创建表和序列
2. **TrayConfig.java** - Model实体类（放在`valueobject/src/com/mycim/prp/model/`）
3. **TrayConfigDao.java** - DAO接口（放在`service/src/com/mycim/dao/prp/`）
4. **TrayConfigDaoImpl.java** - DAO实现（放在`ejb/prp/src/com/mycim/prp/dao/`）
5. **TrayConfigService.java** - Service接口（放在`service/src/com/mycim/service/prp/`）
6. **TrayConfigServiceImpl.java** - Service实现（放在`ejb/prp/src/com/mycim/prp/service/impl/`）
7. **TrayConfigController.java** - Controller控制器（放在`model/src/com/mycim/webapp/controllers/prp/`）

### 提供的接口：
- `/TrayConfigController/getTrayConfigForPrint` - 打印时获取tray配置（通过工单号）
- `/TrayConfigController/getTrayConfigByBomNo` - 根据BOM号查询配置
- `/TrayConfigController/saveTrayConfig` - 新增配置
- `/TrayConfigController/updateTrayConfig` - 更新配置
- `/TrayConfigController/deleteTrayConfig` - 删除配置
- `/TrayConfigController/queryTrayConfigList` - 查询配置列表

### 前端调用示例（打印功能）：
```javascript
Ext.Ajax.request({
    url: '/mycim2/TrayConfigController/getTrayConfigForPrint',
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    jsonData: { workOrderId: outerOrderNo },
    success: function(response) {
        var result = Ext.JSON.decode(response.responseText);
        if (result.success) {
            var integratedTray = result.msg.INTEGRATED_TRAY;
            var shipmentTray = result.msg.SHIPMENT_TRAY;
            var trayColor = result.msg.TRAY_COLOR;
            // 使用这些值进行打印
        }
    }
});
```

这就是完整的配置表代码了！

## User
帮我生成详细的代码

## Assistant
我来帮你生成完整的详细代码文件。让我先检查一下项目结构和现有代码风格，然后创建所有必要的文件。



我来为你生成所有完整的代码文件。

## 1. 数据库建表SQL

```sql
CREATE TABLE TRAY_CONFIG (
    CONFIG_RRN NUMBER(19) PRIMARY KEY,
    BOM_NO VARCHAR2(100) NOT NULL,
    INTEGRATED_TRAY VARCHAR2(100) NOT NULL,
    SHIPMENT_TRAY VARCHAR2(100) NOT NULL,
    TRAY_COLOR VARCHAR2(50),
    CREATE_USER VARCHAR2(50),
    CREATE_TIME TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UPDATE_USER VARCHAR2(50),
    UPDATE_TIME TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT UK_BOM_NO UNIQUE (BOM_NO)
);

COMMENT ON TABLE TRAY_CONFIG IS 'Tray配置表';
COMMENT ON COLUMN TRAY_CONFIG.CONFIG_RRN IS '配置主键';
COMMENT ON COLUMN TRAY_CONFIG.BOM_NO IS 'BOM物料号';
COMMENT ON COLUMN TRAY_CONFIG.INTEGRATED_TRAY IS '一体机tray规格';
COMMENT ON COLUMN TRAY_CONFIG.SHIPMENT_TRAY IS '出货tray规格';
COMMENT ON COLUMN TRAY_CONFIG.TRAY_COLOR IS 'tray颜色';
COMMENT ON COLUMN TRAY_CONFIG.CREATE_USER IS '创建人';
COMMENT ON COLUMN TRAY_CONFIG.CREATE_TIME IS '创建时间';
COMMENT ON COLUMN TRAY_CONFIG.UPDATE_USER IS '更新人';
COMMENT ON COLUMN TRAY_CONFIG.UPDATE_TIME IS '更新时间';

CREATE SEQUENCE TRAY_CONFIG_SEQ
START WITH 1
INCREMENT BY 1
NOCACHE
NOCYCLE;
```

## 2. Model实体类

文件路径：`d:\data\MES\gc\core\src\valueobject\src\com\mycim\prp\model\TrayConfig.java`

```java
package com.mycim.prp.model;

import java.io.Serializable;
import java.util.Date;

import javax.persistence.Column;
import javax.persistence.Id;
import javax.persistence.Table;

import lombok.Data;
import lombok.EqualsAndHashCode;

@Table(name = "TRAY_CONFIG")
@EqualsAndHashCode(callSuper = false)
@Data
public class TrayConfig implements Serializable {

    private static final long serialVersionUID = 1L;

    @Id
    @Column(name = "CONFIG_RRN")
    private Long configRrn;

    @Column(name = "BOM_NO")
    private String bomNo;

    @Column(name = "INTEGRATED_TRAY")
    private String integratedTray;

    @Column(name = "SHIPMENT_TRAY")
    private String shipmentTray;

    @Column(name = "TRAY_COLOR")
    private String trayColor;

    @Column(name = "CREATE_USER")
    private String createUser;

    @Column(name = "CREATE_TIME")
    private Date createTime;

    @Column(name = "UPDATE_USER")
    private String updateUser;

    @Column(name = "UPDATE_TIME")
    private Date updateTime;

    public TrayConfig() {
    }

    public TrayConfig(String bomNo, String integratedTray, String shipmentTray, String trayColor) {
        this.bomNo = bomNo;
        this.integratedTray = integratedTray;
        this.shipmentTray = shipmentTray;
        this.trayColor = trayColor;
    }
}
```

## 3. DAO接口

文件路径：`d:\data\MES\gc\core\src\service\src\com\mycim\dao\prp\TrayConfigDao.java`

```java
package com.mycim.dao.prp;

import java.util.List;
import java.util.Map;

public interface TrayConfigDao {

    List<Map<String, Object>> getTrayConfigByBomNo(String bomNo);

    Map<String, Object> getTrayConfigForPrint(String workOrderId);

    int insertTrayConfig(Map<String, Object> config);

    int updateTrayConfig(Map<String, Object> config);

    int deleteTrayConfig(Long configRrn);

    List<Map<String, Object>> queryTrayConfigList(Map<String, Object> params);
}
```

## 4. DAO实现类

文件路径：`d:\data\MES\gc\core\src\ejb\prp\src\com\mycim\prp\dao\TrayConfigDaoImpl.java`

```java
package com.mycim.prp.dao;

import java.util.List;
import java.util.Map;

import com.mycim.bas.dao.BaseDAOImpl;
import com.mycim.core.factory.AbstractFactory;
import com.mycim.core.jdbc.core.JdbcTemplate;
import com.mycim.core.repository.AbstractRepositoryImpl;
import com.mycim.core.repository.IRepository;
import com.mycim.core.util.CollectionUtils;
import com.mycim.dao.prp.TrayConfigDao;
import com.mycim.util.exception.ExceptionHandler;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class TrayConfigDaoImpl extends BaseDAOImpl implements TrayConfigDao {

    private static final Logger logger = LoggerFactory.getLogger(TrayConfigDaoImpl.class);

    private AbstractRepositoryImpl iRepository = (AbstractRepositoryImpl) AbstractFactory
            .getRepository(IRepository.class.getName());

    private JdbcTemplate jdbcTemplate;

    public TrayConfigDaoImpl() {
        jdbcTemplate = new JdbcTemplate(iRepository.getJdbcTemplate().getDataSource());
    }

    @Override
    public List<Map<String, Object>> getTrayConfigByBomNo(String bomNo) {
        try {
            String sql = "SELECT CONFIG_RRN, BOM_NO, INTEGRATED_TRAY, SHIPMENT_TRAY, TRAY_COLOR "
                    + "FROM TRAY_CONFIG WHERE BOM_NO = ?";
            Object[] args = { bomNo };
            return iRepository.useSqlQuery(sql, Map.class, args);
        } catch (Exception e) {
            throw ExceptionHandler.handlerException(e, logger);
        }
    }

    @Override
    public Map<String, Object> getTrayConfigForPrint(String workOrderId) {
        try {
            String sql = "SELECT tc.CONFIG_RRN, tc.BOM_NO, tc.INTEGRATED_TRAY, tc.SHIPMENT_TRAY, tc.TRAY_COLOR "
                    + "FROM TRAY_CONFIG tc "
                    + "INNER JOIN WORK_ORDER wo ON tc.BOM_NO = wo.BOM_NO "
                    + "WHERE wo.WORK_ORDER_ID = ?";
            Object[] args = { workOrderId };
            List<Map<String, Object>> list = iRepository.useSqlQuery(sql, Map.class, args);
            if (CollectionUtils.isNotEmpty(list)) {
                return list.get(0);
            }
            return null;
        } catch (Exception e) {
            throw ExceptionHandler.handlerException(e, logger);
        }
    }

    @Override
    public int insertTrayConfig(Map<String, Object> config) {
        try {
            String sql = "INSERT INTO TRAY_CONFIG (CONFIG_RRN, BOM_NO, INTEGRATED_TRAY, SHIPMENT_TRAY, TRAY_COLOR, "
                    + "CREATE_USER, CREATE_TIME, UPDATE_USER, UPDATE_TIME) "
                    + "VALUES (TRAY_CONFIG_SEQ.NEXTVAL, ?, ?, ?, ?, ?, SYSDATE, ?, SYSDATE)";
            Object[] args = {
                    config.get("bomNo"),
                    config.get("integratedTray"),
                    config.get("shipmentTray"),
                    config.get("trayColor"),
                    config.get("createUser"),
                    config.get("updateUser")
            };
            return jdbcTemplate.update(sql, args);
        } catch (Exception e) {
            throw ExceptionHandler.handlerException(e, logger);
        }
    }

    @Override
    public int updateTrayConfig(Map<String, Object> config) {
        try {
            String sql = "UPDATE TRAY_CONFIG SET BOM_NO = ?, INTEGRATED_TRAY = ?, SHIPMENT_TRAY = ?, "
                    + "TRAY_COLOR = ?, UPDATE_USER = ?, UPDATE_TIME = SYSDATE WHERE CONFIG_RRN = ?";
            Object[] args = {
                    config.get("bomNo"),
                    config.get("integratedTray"),
                    config.get("shipmentTray"),
                    config.get("trayColor"),
                    config.get("updateUser"),
                    config.get("configRrn")
            };
            return jdbcTemplate.update(sql, args);
        } catch (Exception e) {
            throw ExceptionHandler.handlerException(e, logger);
        }
    }

    @Override
    public int deleteTrayConfig(Long configRrn) {
        try {
            String sql = "DELETE FROM TRAY_CONFIG WHERE CONFIG_RRN = ?";
            Object[] args = { configRrn };
            return jdbcTemplate.update(sql, args);
        } catch (Exception e) {
            throw ExceptionHandler.handlerException(e, logger);
        }
    }

    @Override
    public List<Map<String, Object>> queryTrayConfigList(Map<String, Object> params) {
        try {
            StringBuilder sql = new StringBuilder();
            sql.append("SELECT CONFIG_RRN, BOM_NO, INTEGRATED_TRAY, SHIPMENT_TRAY, TRAY_COLOR, ");
            sql.append("CREATE_USER, TO_CHAR(CREATE_TIME, 'YYYY-MM-DD HH24:MI:SS') AS CREATE_TIME_STR, ");
            sql.append("UPDATE_USER, TO_CHAR(UPDATE_TIME, 'YYYY-MM-DD HH24:MI:SS') AS UPDATE_TIME_STR ");
            sql.append("FROM TRAY_CONFIG WHERE 1=1");

            if (params != null && params.get("bomNo") != null && !params.get("bomNo").toString().isEmpty()) {
                sql.append(" AND BOM_NO LIKE ?");
            }
            if (params != null && params.get("integratedTray") != null
                    && !params.get("integratedTray").toString().isEmpty()) {
                sql.append(" AND INTEGRATED_TRAY LIKE ?");
            }
            if (params != null && params.get("shipmentTray") != null
                    && !params.get("shipmentTray").toString().isEmpty()) {
                sql.append(" AND SHIPMENT_TRAY LIKE ?");
            }

            sql.append(" ORDER BY CREATE_TIME DESC");

            Object[] args = null;
            if (params != null && params.get("bomNo") != null && !params.get("bomNo").toString().isEmpty()) {
                if (params.get("integratedTray") != null && !params.get("integratedTray").toString().isEmpty()) {
                    if (params.get("shipmentTray") != null && !params.get("shipmentTray").toString().isEmpty()) {
                        args = new Object[] {
                                "%" + params.get("bomNo") + "%",
                                "%" + params.get("integratedTray") + "%",
                                "%" + params.get("shipmentTray") + "%"
                        };
                    } else {
                        args = new Object[] {
                                "%" + params.get("bomNo") + "%",
                                "%" + params.get("integratedTray") + "%"
                        };
                    }
                } else if (params.get("shipmentTray") != null && !params.get("shipmentTray").toString().isEmpty()) {
                    args = new Object[] {
                            "%" + params.get("bomNo") + "%",
                            "%" + params.get("shipmentTray") + "%"
                    };
                } else {
                    args = new Object[] { "%" + params.get("bomNo") + "%" };
                }
            } else if (params != null && params.get("integratedTray") != null
                    && !params.get("integratedTray").toString().isEmpty()) {
                if (params.get("shipmentTray") != null && !params.get("shipmentTray").toString().isEmpty()) {
                    args = new Object[] {
                            "%" + params.get("integratedTray") + "%",
                            "%" + params.get("shipmentTray") + "%"
                    };
                } else {
                    args = new Object[] { "%" + params.get("integratedTray") + "%" };
                }
            } else if (params != null && params.get("shipmentTray") != null
                    && !params.get("shipmentTray").toString().isEmpty()) {
                args = new Object[] { "%" + params.get("shipmentTray") + "%" };
            }

            if (args != null) {
                return iRepository.useSqlQuery(sql.toString(), Map.class, args);
            } else {
                return iRepository.useSqlQuery(sql.toString(), Map.class);
            }
        } catch (Exception e) {
            throw ExceptionHandler.handlerException(e, logger);
        }
    }
}
```

## 5. Service接口

文件路径：`d:\data\MES\gc\core\src\service\src\com\mycim\service\prp\TrayConfigService.java`

```java
package com.mycim.service.prp;

import java.util.List;
import java.util.Map;

public interface TrayConfigService {

    Map<String, Object> getTrayConfigForPrint(String workOrderId);

    List<Map<String, Object>> getTrayConfigByBomNo(String bomNo);

    Map<String, Object> saveTrayConfig(Map<String, Object> config, String userName);

    Map<String, Object> updateTrayConfig(Map<String, Object> config, String userName);

    Map<String, Object> deleteTrayConfig(Long configRrn);

    List<Map<String, Object>> queryTrayConfigList(Map<String, Object> params);
}
```

## 6. Service实现类

文件路径：`d:\data\MES\gc\core\src\ejb\prp\src\com\mycim\prp\service\impl\TrayConfigServiceImpl.java`

```java
package com.mycim.prp.service.impl;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import com.mycim.core.util.CollectionUtils;
import com.mycim.dao.prp.TrayConfigDao;
import com.mycim.service.prp.TrayConfigService;
import com.mycim.util.exception.MyCimException;
import com.mycim.util.exception.MyCimParameterException;

public class TrayConfigServiceImpl implements TrayConfigService {

    private TrayConfigDao trayConfigDao;

    public void setTrayConfigDao(TrayConfigDao trayConfigDao) {
        this.trayConfigDao = trayConfigDao;
    }

    @Override
    public Map<String, Object> getTrayConfigForPrint(String workOrderId) {
        if (workOrderId == null || workOrderId.trim().isEmpty()) {
            throw new MyCimParameterException("workOrderId", "工单号不能为空");
        }

        try {
            Map<String, Object> result = trayConfigDao.getTrayConfigForPrint(workOrderId);
            if (result == null) {
                throw new MyCimException("未找到该工单对应的BOM号或Tray配置");
            }
            return result;
        } catch (MyCimException e) {
            throw e;
        } catch (Exception e) {
            throw new MyCimException("获取Tray配置失败: " + e.getMessage());
        }
    }

    @Override
    public List<Map<String, Object>> getTrayConfigByBomNo(String bomNo) {
        if (bomNo == null || bomNo.trim().isEmpty()) {
            throw new MyCimParameterException("bomNo", "BOM号不能为空");
        }

        try {
            return trayConfigDao.getTrayConfigByBomNo(bomNo);
        } catch (Exception e) {
            throw new MyCimException("查询Tray配置失败: " + e.getMessage());
        }
    }

    @Override
    public Map<String, Object> saveTrayConfig(Map<String, Object> config, String userName) {
        if (config == null) {
            throw new MyCimParameterException("config", "配置信息不能为空");
        }

        String bomNo = (String) config.get("bomNo");
        String integratedTray = (String) config.get("integratedTray");
        String shipmentTray = (String) config.get("shipmentTray");

        if (bomNo == null || bomNo.trim().isEmpty()) {
            throw new MyCimParameterException("bomNo", "BOM号不能为空");
        }
        if (integratedTray == null || integratedTray.trim().isEmpty()) {
            throw new MyCimParameterException("integratedTray", "一体机tray规格不能为空");
        }
        if (shipmentTray == null || shipmentTray.trim().isEmpty()) {
            throw new MyCimParameterException("shipmentTray", "出货tray规格不能为空");
        }

        try {
            List<Map<String, Object>> existList = trayConfigDao.getTrayConfigByBomNo(bomNo);
            if (CollectionUtils.isNotEmpty(existList)) {
                throw new MyCimException("该BOM号的配置已存在，请使用更新功能");
            }

            config.put("createUser", userName);
            config.put("updateUser", userName);

            int result = trayConfigDao.insertTrayConfig(config);

            Map<String, Object> response = new HashMap<>();
            response.put("success", result > 0);
            response.put("msg", result > 0 ? "保存成功" : "保存失败");
            return response;
        } catch (MyCimException e) {
            throw e;
        } catch (Exception e) {
            throw new MyCimException("保存Tray配置失败: " + e.getMessage());
        }
    }

    @Override
    public Map<String, Object> updateTrayConfig(Map<String, Object> config, String userName) {
        if (config == null) {
            throw new MyCimParameterException("config", "配置信息不能为空");
        }

        Object configRrnObj = config.get("configRrn");
        if (configRrnObj == null) {
            throw new MyCimParameterException("configRrn", "配置主键不能为空");
        }

        String bomNo = (String) config.get("bomNo");
        String integratedTray = (String) config.get("integratedTray");
        String shipmentTray = (String) config.get("shipmentTray");

        if (bomNo == null || bomNo.trim().isEmpty()) {
            throw new MyCimParameterException("bomNo", "BOM号不能为空");
        }
        if (integratedTray == null || integratedTray.trim().isEmpty()) {
            throw new MyCimParameterException("integratedTray", "一体机tray规格不能为空");
        }
        if (shipmentTray == null || shipmentTray.trim().isEmpty()) {
            throw new MyCimParameterException("shipmentTray", "出货tray规格不能为空");
        }

        try {
            config.put("updateUser", userName);

            int result = trayConfigDao.updateTrayConfig(config);

            Map<String, Object> response = new HashMap<>();
            response.put("success", result > 0);
            response.put("msg", result > 0 ? "更新成功" : "更新失败");
            return response;
        } catch (MyCimException e) {
            throw e;
        } catch (Exception e) {
            throw new MyCimException("更新Tray配置失败: " + e.getMessage());
        }
    }

    @Override
    public Map<String, Object> deleteTrayConfig(Long configRrn) {
        if (configRrn == null) {
            throw new MyCimParameterException("configRrn", "配置主键不能为空");
        }

        try {
            int result = trayConfigDao.deleteTrayConfig(configRrn);

            Map<String, Object> response = new HashMap<>();
            response.put("success", result > 0);
            response.put("msg", result > 0 ? "删除成功" : "删除失败");
            return response;
        } catch (MyCimException e) {
            throw e;
        } catch (Exception e) {
            throw new MyCimException("删除Tray配置失败: " + e.getMessage());
        }
    }

    @Override
    public List<Map<String, Object>> queryTrayConfigList(Map<String, Object> params) {
        try {
            return trayConfigDao.queryTrayConfigList(params);
        } catch (Exception e) {
            throw new MyCimException("查询Tray配置列表失败: " + e.getMessage());
        }
    }
}
```

## 7. Controller层

文件路径：`d:\data\MES\gc\core\src\model\src\com\mycim\webapp\controllers\prp\TrayConfigController.java`

```java
package com.mycim.webapp.controllers.prp;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import com.mycim.service.prp.TrayConfigService;
import com.mycim.service.support.ServiceFactory;
import com.mycim.sys.model.User;
import com.mycim.util.exception.MyCimException;
import com.mycim.webapp.actions.AbstractAction;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
@RequestMapping("/TrayConfigController")
public class TrayConfigController extends AbstractAction {

    protected final Logger log = LoggerFactory.getLogger(this.getClass().getName());

    protected final TrayConfigService trayConfigService = (TrayConfigService) ServiceFactory
            .getService(TrayConfigService.class);

    @RequestMapping(value = "/getTrayConfigForPrint", method = RequestMethod.POST, produces = "application/json; charset=utf-8")
    @ResponseBody
    public Map<String, Object> getTrayConfigForPrint(@RequestBody Map<String, String> params) {
        Map<String, Object> result = new HashMap<>();
        try {
            String workOrderId = params.get("workOrderId");
            if (workOrderId == null || workOrderId.trim().isEmpty()) {
                result.put("success", false);
                result.put("msg", "工单号不能为空");
                return result;
            }

            Map<String, Object> config = trayConfigService.getTrayConfigForPrint(workOrderId);
            result.put("success", true);
            result.put("msg", config);
        } catch (MyCimException e) {
            log.error("获取Tray配置失败: {}", e.getMessage());
            result.put("success", false);
            result.put("msg", e.getMessage());
        } catch (Exception e) {
            log.error("获取Tray配置系统异常", e);
            result.put("success", false);
            result.put("msg", "系统异常: " + e.getMessage());
        }
        return result;
    }

    @RequestMapping(value = "/getTrayConfigByBomNo", method = RequestMethod.POST, produces = "application/json; charset=utf-8")
    @ResponseBody
    public Map<String, Object> getTrayConfigByBomNo(@RequestBody Map<String, String> params) {
        Map<String, Object> result = new HashMap<>();
        try {
            String bomNo = params.get("bomNo");
            if (bomNo == null || bomNo.trim().isEmpty()) {
                result.put("success", false);
                result.put("msg", "BOM号不能为空");
                return result;
            }

            List<Map<String, Object>> configList = trayConfigService.getTrayConfigByBomNo(bomNo);
            result.put("success", true);
            result.put("msg", configList);
        } catch (MyCimException e) {
            log.error("查询Tray配置失败: {}", e.getMessage());
            result.put("success", false);
            result.put("msg", e.getMessage());
        } catch (Exception e) {
            log.error("查询Tray配置系统异常", e);
            result.put("success", false);
            result.put("msg", "系统异常: " + e.getMessage());
        }
        return result;
    }

    @RequestMapping(value = "/saveTrayConfig", method = RequestMethod.POST, produces = "application/json; charset=utf-8")
    @ResponseBody
    public Map<String, Object> saveTrayConfig(@RequestBody Map<String, Object> params, HttpServletRequest request) {
        Map<String, Object> result = new HashMap<>();
        try {
            User user = getUser(request);
            String userName = user != null ? user.getUserId() : "SYSTEM";

            result = trayConfigService.saveTrayConfig(params, userName);
        } catch (MyCimException e) {
            log.error("保存Tray配置失败: {}", e.getMessage());
            result.put("success", false);
            result.put("msg", e.getMessage());
        } catch (Exception e) {
            log.error("保存Tray配置系统异常", e);
            result.put("success", false);
            result.put("msg", "系统异常: " + e.getMessage());
        }
        return result;
    }

    @RequestMapping(value = "/updateTrayConfig", method = RequestMethod.POST, produces = "application/json; charset=utf-8")
    @ResponseBody
    public Map<String, Object> updateTrayConfig(@RequestBody Map<String, Object> params, HttpServletRequest request) {
        Map<String, Object> result = new HashMap<>();
        try {
            User user = getUser(request);
            String userName = user != null ? user.getUserId() : "SYSTEM";

            result = trayConfigService.updateTrayConfig(params, userName);
        } catch (MyCimException e) {
            log.error("更新Tray配置失败: {}", e.getMessage());
            result.put("success", false);
            result.put("msg", e.getMessage());
        } catch (Exception e) {
            log.error("更新Tray配置系统异常", e);
            result.put("success", false);
            result.put("msg", "系统异常: " + e.getMessage());
        }
        return result;
    }

    @RequestMapping(value = "/deleteTrayConfig", method = RequestMethod.POST, produces = "application/json; charset=utf-8")
    @ResponseBody
    public Map<String, Object> deleteTrayConfig(@RequestBody Map<String, Object> params) {
        Map<String, Object> result = new HashMap<>();
        try {
            Long configRrn = null;
            if (params.get("configRrn") != null) {
                configRrn = Long.valueOf(params.get("configRrn").toString());
            }

            result = trayConfigService.deleteTrayConfig(configRrn);
        } catch (MyCimException e) {
            log.error("删除Tray配置失败: {}", e.getMessage());
            result.put("success", false);
            result.put("msg", e.getMessage());
        } catch (Exception e) {
            log.error("删除Tray配置系统异常", e);
            result.put("success", false);
            result.put("msg", "系统异常: " + e.getMessage());
        }
        return result;
    }

    @RequestMapping(value = "/queryTrayConfigList", method = RequestMethod.POST, produces = "application/json; charset=utf-8")
    @ResponseBody
    public Map<String, Object> queryTrayConfigList(@RequestBody Map<String, Object> params) {
        Map<String, Object> result = new HashMap<>();
        try {
            List<Map<String, Object>> list = trayConfigService.queryTrayConfigList(params);
            result.put("success", true);
            result.put("msg", list);
            result.put("total", list.size());
        } catch (MyCimException e) {
            log.error("查询Tray配置列表失败: {}", e.getMessage());
            result.put("success", false);
            result.put("msg", e.getMessage());
        } catch (Exception e) {
            log.error("查询Tray配置列表系统异常", e);
            result.put("success", false);
            result.put("msg", "系统异常: " + e.getMessage());
        }
        return result;
    }
}
```

## 8. Service配置文件

需要在Spring配置文件中注册Service Bean。文件路径根据项目实际情况调整：

```xml
<!-- 在 applicationContext.xml 或 prp-service.xml 中添加 -->
<bean id="trayConfigDao" class="com.mycim.prp.dao.TrayConfigDaoImpl" />

<bean id="trayConfigService" class="com.mycim.prp.service.impl.TrayConfigServiceImpl">
    <property name="trayConfigDao" ref="trayConfigDao" />
</bean>
```

---

## 文件清单汇总

| 序号 | 文件名 | 路径 | 说明 |
|------|--------|------|------|
| 1 | 建表SQL | 数据库执行 | 创建TRAY_CONFIG表和序列 |
| 2 | TrayConfig.java | `core/src/valueobject/src/com/mycim/prp/model/` | 实体类 |
| 3 | TrayConfigDao.java | `core/src/service/src/com/mycim/dao/prp/` | DAO接口 |
| 4 | TrayConfigDaoImpl.java | `core/src/ejb/prp/src/com/mycim/prp/dao/` | DAO实现 |
| 5 | TrayConfigService.java | `core/src/service/src/com/mycim/service/prp/` | Service接口 |
| 6 | TrayConfigServiceImpl.java | `core/src/ejb/prp/src/com/mycim/prp/service/impl/` | Service实现 |
| 7 | TrayConfigController.java | `core/src/model/src/com/mycim/webapp/controllers/prp/` | Controller |
| 8 | Spring配置 | 项目Spring配置文件 | Bean注册 |

## 提供的接口清单

| 接口路径 | 方法 | 说明 | 请求参数 |
|---------|------|------|---------|
| `/TrayConfigController/getTrayConfigForPrint` | POST | 打印时获取tray配置 | `{"workOrderId": "工单号"}` |
| `/TrayConfigController/getTrayConfigByBomNo` | POST | 根据BOM号查询 | `{"bomNo": "BOM号"}` |
| `/TrayConfigController/saveTrayConfig` | POST | 新增配置 | `{"bomNo": "", "integratedTray": "", "shipmentTray": "", "trayColor": ""}` |
| `/TrayConfigController/updateTrayConfig` | POST | 更新配置 | `{"configRrn": 1, "bomNo": "", "integratedTray": "", "shipmentTray": "", "trayColor": ""}` |
| `/TrayConfigController/deleteTrayConfig` | POST | 删除配置 | `{"configRrn": 1}` |
| `/TrayConfigController/queryTrayConfigList` | POST | 查询列表 | `{"bomNo": "", "integratedTray": "", "shipmentTray": ""}` |

## 前端调用示例

```javascript
Ext.Ajax.request({
    url: '/mycim2/TrayConfigController/getTrayConfigForPrint',
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    jsonData: { workOrderId: outerOrderNo },
    success: function(response) {
        var result = Ext.JSON.decode(response.responseText);
        if (result.success) {
            var integratedTray = result.msg.INTEGRATED_TRAY;
            var shipmentTray = result.msg.SHIPMENT_TRAY;
            var trayColor = result.msg.TRAY_COLOR;
        } else {
            Ext.Msg.alert('提示', result.msg);
        }
    },
    failure: function() {
        Ext.Msg.alert('提示', '系统异常');
    }
});
```

所有代码已生成完毕！

## User
直接帮我生成文件代码

## Assistant
我来直接生成所有代码文件。



我来直接为你生成所有代码文件。



所有代码文件已生成完毕！

## 已创建的文件清单

| 序号 | 文件路径 | 说明 |
|------|---------|------|
| 1 | [TrayConfig.java](file:///d:/data/MES/gc/core/src/valueobject/src/com/mycim/prp/model/TrayConfig.java) | Model实体类 |
| 2 | [TrayConfigDao.java](file:///d:/data/MES/gc/core/src/service/src/com/mycim/dao/prp/TrayConfigDao.java) | DAO接口 |
| 3 | [TrayConfigDaoImpl.java](file:///d:/data/MES/gc/core/src/ejb/prp/src/com/mycim/prp/dao/TrayConfigDaoImpl.java) | DAO实现类 |
| 4 | [TrayConfigService.java](file:///d:/data/MES/gc/core/src/service/src/com/mycim/service/prp/TrayConfigService.java) | Service接口 |
| 5 | [TrayConfigServiceImpl.java](file:///d:/data/MES/gc/core/src/ejb/prp/src/com/mycim/prp/service/impl/TrayConfigServiceImpl.java) | Service实现类 |
| 6 | [TrayConfigController.java](file:///d:/data/MES/gc/core/src/model/src/com/mycim/webapp/controllers/prp/TrayConfigController.java) | Controller控制器 |

## 数据库建表SQL

请在Oracle数据库中执行以下SQL：

```sql
CREATE TABLE TRAY_CONFIG (
    CONFIG_RRN NUMBER(19) PRIMARY KEY,
    BOM_NO VARCHAR2(100) NOT NULL,
    INTEGRATED_TRAY VARCHAR2(100) NOT NULL,
    SHIPMENT_TRAY VARCHAR2(100) NOT NULL,
    TRAY_COLOR VARCHAR2(50),
    CREATE_USER VARCHAR2(50),
    CREATE_TIME TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UPDATE_USER VARCHAR2(50),
    UPDATE_TIME TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT UK_BOM_NO UNIQUE (BOM_NO)
);

COMMENT ON TABLE TRAY_CONFIG IS 'Tray配置表';
COMMENT ON COLUMN TRAY_CONFIG.CONFIG_RRN IS '配置主键';
COMMENT ON COLUMN TRAY_CONFIG.BOM_NO IS 'BOM物料号';
COMMENT ON COLUMN TRAY_CONFIG.INTEGRATED_TRAY IS '一体机tray规格';
COMMENT ON COLUMN TRAY_CONFIG.SHIPMENT_TRAY IS '出货tray规格';
COMMENT ON COLUMN TRAY_CONFIG.TRAY_COLOR IS 'tray颜色';
COMMENT ON COLUMN TRAY_CONFIG.CREATE_USER IS '创建人';
COMMENT ON COLUMN TRAY_CONFIG.CREATE_TIME IS '创建时间';
COMMENT ON COLUMN TRAY_CONFIG.UPDATE_USER IS '更新人';
COMMENT ON COLUMN TRAY_CONFIG.UPDATE_TIME IS '更新时间';

CREATE SEQUENCE TRAY_CONFIG_SEQ
START WITH 1
INCREMENT BY 1
NOCACHE
NOCYCLE;
```

## Spring配置

需要在你的Spring配置文件中添加以下Bean配置：

```xml
<bean id="trayConfigDao" class="com.mycim.prp.dao.TrayConfigDaoImpl" />

<bean id="trayConfigService" class="com.mycim.prp.service.impl.TrayConfigServiceImpl">
    <property name="trayConfigDao" ref="trayConfigDao" />
</bean>
```

## 提供的接口清单

| 接口 | 说明 |
|------|------|
| `/TrayConfigController/getTrayConfigForPrint` | 打印时通过工单号获取tray配置 |
| `/TrayConfigController/getTrayConfigByBomNo` | 根据BOM号查询配置 |
| `/TrayConfigController/saveTrayConfig` | 新增配置 |
| `/TrayConfigController/updateTrayConfig` | 更新配置 |
| `/TrayConfigController/deleteTrayConfig` | 删除配置 |
| `/TrayConfigController/queryTrayConfigList` | 查询配置列表 |

## User
前端页面的代码也要生成啊，而且你的sql脚本不需要，直接用项目的liquibase的方式

## Assistant
好的，我来生成前端页面代码和Liquibase数据库变更文件。先让我看看项目中Liquibase的使用方式和现有的前端页面结构。



让我查看更多Liquibase文件示例和前端页面结构：



现在我来生成Liquibase文件和前端页面代码：



所有代码已生成完毕！以下是完整的文件清单：

## 📋 已创建/修改的文件

### 后端代码
| 文件 | 路径 | 状态 |
|------|------|------|
| [TrayConfig.java](file:///d:/data/MES/gc/core/src/valueobject/src/com/mycim/prp/model/TrayConfig.java) | Model实体类 | ✅ 已创建 |
| [TrayConfigDao.java](file:///d:/data/MES/gc/core/src/service/src/com/mycim/dao/prp/TrayConfigDao.java) | DAO接口 | ✅ 已创建 |
| [TrayConfigDaoImpl.java](file:///d:/data/MES/gc/core/src/ejb/prp/src/com/mycim/prp/dao/TrayConfigDaoImpl.java) | DAO实现 | ✅ 已创建 |
| [TrayConfigService.java](file:///d:/data/MES/gc/core/src/service/src/com/mycim/service/prp/TrayConfigService.java) | Service接口 | ✅ 已创建 |
| [TrayConfigServiceImpl.java](file:///d:/data/MES/gc/core/src/ejb/prp/src/com/mycim/prp/service/impl/TrayConfigServiceImpl.java) | Service实现 | ✅ 已创建 |
| [TrayConfigController.java](file:///d:/data/MES/gc/core/src/model/src/com/mycim/webapp/controllers/prp/TrayConfigController.java) | Controller | ✅ 已创建 |

### 数据库变更
| 文件 | 路径 | 状态 |
|------|------|------|
| [db.changelog-0.0.1.tray_config.yaml](file:///d:/data/MES/gc/core/web/WEB-INF/config/db/changelog/db.changelog-0.0.1.tray_config.yaml) | Liquibase变更文件 | ✅ 已创建 |

**注意：** 需要在master changelog文件中添加引用，找到项目的主changelog文件（如`db.changelog-master.yaml`），添加：
```yaml
  - include:
      file: db.changelog-0.0.1.tray_config.yaml
      relativeToChangelogFile: true
```

### 前端代码
| 文件 | 路径 | 状态 |
|------|------|------|
| [comProcessCardPrintForm.js](file:///d:/data/MES/gc/core/web/prp/comProcessCardPrint/comProcessCardPrintForm.js) | 打印表单 | ✅ 已修改 |

## 📝 修改说明

### 前端打印功能修改（comProcessCardPrintForm.js）
1. **删除了产品规格字段**（原第48-74行的combo）
2. **修改了doClear函数** - 移除了productSpec的清空
3. **修改了doPrintLabel函数** - 打印时通过AJAX调用`/mycim2/TrayConfigController/getTrayConfigForPrint`接口，根据工单号获取tray配置，然后将`integratedTraySpec`、`shipmentTraySpec`、`trayColor`三个参数传入打印信息

### 打印参数变化
**旧参数：**
```
productSpec=产品规格
```

**新参数：**
```
integratedTraySpec=一体机tray规格
shipmentTraySpec=出货tray规格
trayColor=tray颜色
```

## ⚠️ 后续步骤

1. **注册Spring Bean** - 在Spring配置文件中添加：
```xml
<bean id="trayConfigDao" class="com.mycim.prp.dao.TrayConfigDaoImpl" />
<bean id="trayConfigService" class="com.mycim.prp.service.impl.TrayConfigServiceImpl">
    <property name="trayConfigDao" ref="trayConfigDao" />
</bean>
```

2. **添加Liquibase引用** - 在master changelog中包含新的yaml文件

3. **修改BarTender打印模板** - 将模板中的`productSpec`变量替换为新的三个变量

## User
你好像没有听懂我在说什么，我需要你完整的生成所有tray配置表需要的代码，包括前端后端，能直接使用的代码，还有配置表怎么能没有历史表啊

## Assistant
抱歉，我前面理解偏成“打印改造”了，没有按“完整可维护的配置表功能”来做。现在我按你的意思重新补齐：**Tray配置表完整功能 = 主表 + 历史表 + Liquibase + 后端增删改查/历史记录 + 前端维护页面 + 打印查询接口**，并把代码生成到项目里。

