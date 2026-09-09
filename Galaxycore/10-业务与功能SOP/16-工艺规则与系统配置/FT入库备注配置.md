---
title: FT入库备注配置
date: 2024-10-21
author: MES Team
status: completed
tags: [工艺, FT, 功能]
---

# FT真空包入库备注管

🎣 关于表格选中统计计数的任务，我放弃了，这前端的玩意太难了。不知道代码从何下手。这周继续搞一下FT入库备注

## 会议记录

![20241017134116.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241017134116.png)

🎣 需要显示的主要是这三个列，增加型号、等级、时效性字

## 数据表结

```yaml
databaseChangeLog:
  - changeSet:
      id: GC_FT_STORAGEREMARK_CONFIG_addColumn_0.0.3
      author: Natsume_wang
      comment: GC_FT_STORAGEREMARK_CONFIG add column
      changes:
        - addColumn:
            tableName: GC_FT_STORAGEREMARK_CONFIG
            columns:
              - column:
                  name: PRODUCT_MODEL
                  type: VARCHAR2(32)
                  remarks: 产品型号
              - column:
                  name: GRADE
                  type: VARCHAR2(16)
                  remarks: 等级
              - column:
                  name: TIMELY
                  type: VARCHAR2(16)
                  remarks: 时效性（天）
              - column:
                  name: IS_VALID
                  type: NUMBER(1)
                  remarks: 是否有效
              - column:
                  name: CREATE_TIME
                  type: DATE
                  remarks: 创建时间
```

## 实体

```java
@Column(name="PRODUCT_MODEL")
private String productModel;

@Column(name="GRADE")
private String grade;

@Column(name="TIMELY")
private String timely;

@Column(name="IS_VALID")
private int isValid;

@Column(name="CREATE_TIME")
private Date createTime;
```

## JS

```java
//18_CN.js
i18n_fld_timely = "时效性（day;
i18n_fld_isValid = "是否有效";
i18n_fld_createTime = "创建时间";

//18_EN.js
i18n_fld_timely = "timely（day;
i18n_fld_isValid = "isValid";
i18n_fld_createTime = "createTime";
```

```jsx
//主要是添加三个文本框，不过添加搜索的方法中也要修改如下细节
{
   xtype: 'mycim.textfield',
   typeAhead: true,
   allowBlank : false,
   name: 'productModel',
   id:'productModel',
   fieldLabel: i18n_fld_productModel,
   columnWidth: 0.20,
},{
   xtype: 'mycim.textfield',
   typeAhead: true,
   allowBlank : false,
   name: 'grade',
   id:'grade',
   fieldLabel: i18n_fld_grade,
   columnWidth: 0.15,
},{
   xtype: 'mycim.textfield',
   typeAhead: true,
   allowBlank : false,
   name: 'timely',
   id:'timely',
   fieldLabel: i18n_fld_timely,
   columnWidth: 0.15,
}
```

```jsx
{
   name : 'productModel'
},{
   name : 'grade'
},{
   name : 'timely'
},{
   name : 'isValid'
},{
   name : 'createTime'
}
//......
{
   xtype : 'gridcolumn',
   dataIndex : 'productModel',
   header : i18n_fld_productModel,
   sortable : true,
   align : 'center',
   width : 130,
},{
   xtype : 'gridcolumn',
   dataIndex : 'grade',
   header : i18n_fld_grade,
   sortable : true,
   align : 'center',
   width : 130,
},{
   xtype : 'gridcolumn',
   dataIndex : 'timely',
   header : i18n_fld_timely,
   sortable : true,
   align : 'center',
   width : 130,
},{
   xtype : 'gridcolumn',
   dataIndex : 'isValid',
   header : i18n_fld_isValid,
   sortable : true,
   align : 'center',
   width : 130,
   hidden: true
},{
   xtype : 'gridcolumn',
   dataIndex : 'createTime',
   header : i18n_fld_createTime,
   sortable : true,
   align : 'center',
   width : 130,
   hidden: true
}
```
## JAVA
![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241017134826.png)
![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241017134903.png)
![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241017134920.png)
![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241017134941.png)
## 阶段1结束：疑

- [x] 时效性是下拉选择还是自己输（自己输，整形数字或者ALL
- [x] 等级是下拉还是（自己输）
- [x] 产品型号下来还是（下拉，在产品型号池中新增一个ALL
- [x] 修改可以修改哪些属性（新增的三个都可以修改
- [x] 下拉出现可用的产品型号是在哪里（李志威发我）

## 核心的调用方

> 由于每个页面的需求都是根据页面的产品型号和等级，去入库备注表**`GC_FT_STORAGEREMARK_CONFIG`**中查找对应的有效的入库备

💡 一共有四种情况。产品型号和等级的关系：1.一对一.一对多.多对一.多对

```java
//FtService.java
...
//传入所需的产品型号和等级，获取所有有效的入库备注
List<GcFtStorageRemarkConfig> getValidGcFtStorageRemarkConfigByProductModelAndGrade(String productModel, String grade)throws MyCimException;
...

//FtServiceImpl.java
//传入所需的产品型号和等级，获取所有有效的入库备注
@Override
public List<GcFtStorageRemarkConfig> getValidGcFtStorageRemarkConfigByProductModelAndGrade(String productModel, String grade)throws MyCimException{
	try{
		return gcFtStorageRemarkConfigRepository.getValidGcFtStorageRemarkConfigByProductModelAndGrade(productModel, grade);
	}catch(Exception e) {
		throw ExceptionHandler.handlerException(e, log);
	}
}

//GcFtStorageRemarkConfigRepository.java
List<GcFtStorageRemarkConfig> getValidGcFtStorageRemarkConfigByProductModelAndGrade(String productModel, String grade)throws MyCimException;

//GcFtStorageRemarkConfigRepositoryImpl.java
@Override
	public List<GcFtStorageRemarkConfig> getValidGcFtStorageRemarkConfigByProductModelAndGrade(String productModel, String grade) throws MyCimException {
		try{
			final String ALL = "ALL";
			List<GcFtStorageRemarkConfig> list = new ArrayList<>();
			if (!ALL.equals(productModel) && !ALL.equals(grade)){
				SqlBuilder sqlBuilder1 = getSqlBuilder().select(columnNames).from(tableName).where("1=1");
				sqlBuilder1.and("PRODUCT_MODEL").eq(productModel)
						.and("GRADE").eq(grade).and("IS_VALID").eq(1);
				SqlBuilder sqlBuilder2 = getSqlBuilder().select(columnNames).from(tableName).where("1=1");
				sqlBuilder2.and("PRODUCT_MODEL").eq("ALL")
						.and("GRADE").eq(grade).and("IS_VALID").eq(1);
				SqlBuilder sqlBuilder3 = getSqlBuilder().select(columnNames).from(tableName).where("1=1");
				sqlBuilder3.and("PRODUCT_MODEL").eq(productModel)
						.and("GRADE").eq("ALL").and("IS_VALID").eq(1);
				list.addAll(sqlBuilder1.queryList(GcFtStorageRemarkConfig.class));
				list.addAll(sqlBuilder2.queryList(GcFtStorageRemarkConfig.class));
				list.addAll(sqlBuilder3.queryList(GcFtStorageRemarkConfig.class));
			}else if (ALL.equals(productModel) && !ALL.equals(grade)){
				SqlBuilder sqlBuilder2 = getSqlBuilder().select(columnNames).from(tableName).where("1=1");
				sqlBuilder2.and("PRODUCT_MODEL").eq("ALL")
						.and("GRADE").eq(grade).and("IS_VALID").eq(1);
				list.addAll(sqlBuilder2.queryList(GcFtStorageRemarkConfig.class));
			}else if (!ALL.equals(productModel)){
				SqlBuilder sqlBuilder3 = getSqlBuilder().select(columnNames).from(tableName).where("1=1");
				sqlBuilder3.and("PRODUCT_MODEL").eq(productModel)
						.and("GRADE").eq("ALL").and("IS_VALID").eq(1);
				list.addAll(sqlBuilder3.queryList(GcFtStorageRemarkConfig.class));
			}
			SqlBuilder sqlBuilder4 = getSqlBuilder().select(columnNames).from(tableName).where("1=1");
			sqlBuilder4.and("PRODUCT_MODEL").eq("ALL")
					.and("GRADE").eq("ALL").and("IS_VALID").eq(1);
			list.addAll(sqlBuilder4.queryList(GcFtStorageRemarkConfig.class));
			return list;
		}catch(Exception e){
			throw ExceptionHandler.handlerException(e, log);
		}
	}

```

### 增加产品型号类型

💡 在添加入库备注的页面，产品类型的增加查找的下拉框中增加一个ALL的栏

```java
//FtProgramRelationshipMaintenanceAction.java
private ActionForward queryProduct(Long facilityRrn, HttpServletRequest request, HttpServletResponse response) {
		try {
			...
			//增加产品型号ALL类型
			Map<String, Object> m = new HashMap<>();
			m.put("key", "ALL");
			m.put("value", "ALL");
			dataMapList.add(m);
			...
	}
```

### 入库备注的查询与修改

```java
//GcFtStorageRemarkConfigAction.java
//更新入库备注
private ActionForward update(HttpServletRequest request, HttpServletResponse response) {
		Map<String, Object> map = new HashMap<>();
		try{
			...
			String productModel = WebUtils.getParameter("productModel", request);
			String grade = WebUtils.getParameter("grade", request);
			String timely = WebUtils.getParameter("timely", request);
			GcFtStorageRemarkConfig gcFtStorageRemarkConfig = ftService.getGcFtStorageRemarkConfigByRrn(objectRrn);
			if(gcFtStorageRemarkConfig != null){
				gcFtStorageRemarkConfig.setRemarkCode(remarkCode);
				gcFtStorageRemarkConfig.setRemarkDescribe(remarkDescribe);
				gcFtStorageRemarkConfig.setShipmentType(shipmentType);
				gcFtStorageRemarkConfig.setProductModel(productModel);
				gcFtStorageRemarkConfig.setGrade(grade);
				gcFtStorageRemarkConfig.setTimely(timely);

				//更新有效
				gcFtStorageRemarkConfig.setIsValid(updateIsValid(new Date(), timely, gcFtStorageRemarkConfig.getCreateTime()));
				...
			}
			...
	}

//更新入库备注的有效
private int updateIsValid(Date currentDate, String timely, Date createTime) throws ParseException {
		if("ALL".equals(timely)){
			return 1;
		}else {
			int i = Integer.parseInt(timely);
			Calendar c = Calendar.getInstance();
			c.setTime(createTime);
			c.add(Calendar.DATE, i);
			Date newDate = c.getTime();//有效期的最后一
			return newDate.after(currentDate) ? 1 : 0;
		}
	}

//保存入库备注
private ActionForward save(HttpServletRequest request, HttpServletResponse response) {
		Map<String, Object> map = new HashMap<>();
		try{
			...
			String productModel = WebUtils.getParameter("productModel", request);
			String grade = WebUtils.getParameter("grade", request);
			String timely = WebUtils.getParameter("timely", request);

			...
			gcFtStorageRemarkConfig.setTimely(timely);
			gcFtStorageRemarkConfig.setProductModel(productModel);
			gcFtStorageRemarkConfig.setGrade(grade);

			gcFtStorageRemarkConfig.setCreateTime(new Date());
			int isValid = updateIsValid(new Date(), timely, gcFtStorageRemarkConfig.getCreateTime());
			gcFtStorageRemarkConfig.setIsValid(isValid);

			...
		}catch(Exception e){
			WebUtil.writeJson(response, ReponseJSONBuilder.buildErrorMsg(e));
		}
		return WebUtils.NULLActionForward;
	}

//查询入库备注并更
private ActionForward search(HttpServletRequest request, HttpServletResponse response) {
		Map<String, Object> map = new HashMap<>();
		try{
			String productType = WebUtils.getParameter("productClassifyForm", request);
			
			List<GcFtStorageRemarkConfig> gcFtStorageRemarkList = ftService.getByProductClassifyAndRemarkCode(productType,null);

			for (GcFtStorageRemarkConfig c: gcFtStorageRemarkList) {
				if (updateIsValid(new Date(), c.getTimely(), c.getCreateTime()) != 1){
					c.setIsValid(0);
					ftService.updateGcFtStorageRemarkConfig(c);
					gcFtStorageRemarkList.remove(c);//移除过期的备
				}
			}

			map.put("data", gcFtStorageRemarkListToJsonArray(gcFtStorageRemarkList));
			
			WebUtil.writeJson(response, ReponseJSONBuilder.buildSuccessMsg(JSONUtils.toJSONString(map)));
		}catch(Exception e){
			WebUtil.writeJson(response, ReponseJSONBuilder.buildErrorMsg(e));
		}
		return WebUtils.NULLActionForward;
	}

```

### 入库备注表单增加栏位

```java
//gcFtStorageRemarkConfigForm.js
...
var orignalModelIdComboStore = Ext.create('Ext.data.Store', {
	fields: ['key', 'value'],
	proxy: {
		type: 'ajax',
		actionMethods: { read: 'GET' },
		url: '/mycim2/ftProgramRelationshipMaintenance.do?action=queryProduct',
		queryMask: false,
	},
	autoLoad: true
});
...

{
    xtype: 'combo',
    displayField: 'key',
    valueField: 'value',
    typeAhead: false,
    editable: false,
    selectOnFocus: true,
    store: orignalModelIdComboStore,
    queryMode: 'local',
    enableKeyEvents: true,
    name: 'productModel',
    id: 'productModel',
    fieldLabel: i18n_fld_productModel + i18n_fld_Red_Star,
    columnWidth: 0.25,
}, {
    xtype: 'mycim.textfield',
    typeAhead: true,
    allowBlank: false,
    name: 'grade',
    id: 'grade',
    fieldLabel: i18n_fld_grade,
    columnWidth: 0.25,
}, {
    xtype: 'mycim.textfield',
    typeAhead: true,
    allowBlank: false,
    name: 'timely',
    id: 'timely',
    fieldLabel: i18n_fld_timely,
    columnWidth: 0.25,
}

//gcFtStorageRemarkConfigGrid.js
{
    xtype: 'gridcolumn',
    dataIndex: 'productModel',
    header: i18n_fld_productModel,
    sortable: true,
    align: 'center',
    width: 130,
}, {
    xtype: 'gridcolumn',
    dataIndex: 'grade',
    header: i18n_fld_grade,
    sortable: true,
    align: 'center',
    width: 130,
}, {
    xtype: 'gridcolumn',
    dataIndex: 'timely',
    header: i18n_fld_timely,
    sortable: true,
    align: 'center',
    width: 130,
}, {
    xtype: 'gridcolumn',
    dataIndex: 'isValid',
    header: i18n_fld_isValid,
    sortable: true,
    align: 'center',
    width: 130,
    hidden: true
}, {
    xtype: 'gridcolumn',
    dataIndex: 'createTime',
    header: i18n_fld_createTime,
    sortable: true,
    align: 'center',
    width: 130,
    hidden: true
}
```

### 入库备注修改的弹

```java
//gcFtStorageRemarkAltering.jsp
...
var productModel = '<%=request.getParameter("productModel")%>';
var grade = '<%=request.getParameter("grade")%>';
var timely = '<%=request.getParameter("timely")%>';
...

//gcFtStorageRemarkAlteringForm.js
{
    xtype: 'combobox',
    editable: false,
    allowBlank: false,
    fieldLabel: i18n_fld_productModel,
    name: 'productModel',
    id: 'productModel',
    columnWidth: 0.5,
    enableKeyEvents: true,
    typeAhead: true,
    displayField: 'key',
    valueField: 'value',
    store: Ext.create('Ext.data.Store', {
        fields: ['key', 'value'],
        proxy: {
            type: "ajax",
            actionMethods: {
                read: 'GET'
            },
            url: '/mycim2/ftProgramRelationshipMaintenance.do?action=queryProduct',
            queryMask: false
        },
        autoLoad: true
    }),
    queryMode: 'local',
}
xtype: 'mycim.textfield',
    name: 'grade',
    allowBlank: false,
    id: 'grade',
    fieldLabel: i18n_fld_grade,
    width: 350,
    value: '',
    enableKeyEvents: true,
    targetIds: 'grade',
}, {
    xtype: 'mycim.textfield',
    name: 'timely',
    allowBlank: false,
    id: 'timely',
    fieldLabel: i18n_fld_timely,
    width: 350,
    value: '',
    enableKeyEvents: true,
    targetIds: 'timely'
}

```

## 七个有关入库备注的修

💡 ~~这次的作业，上一部分是蛮容易的，但是下半部分这个需要修改七个部分，而且每个部分都不太一样，就离谱，而且让我修改也只是把修改的地方给我指了出来，并没有和我详细说一些细节，关于我如何去调试，怎么操作的流程，都得自己问了才知道_-||~~

### 制造管》包装操》FT_FQC-》结

```java
//FtFqcAction.java
private ActionForward queryShipNote(HttpServletRequest request, HttpServletResponse response) {
			...
			String productId = WebUtils.getParameter("productId", request);
			String grade = WebUtils.getParameter("grade", request);
      List<GcFtStorageRemarkConfig> FtStorageRemarkConfigs = ftService.getValidGcFtStorageRemarkConfigByProductModelAndGrade(productId, grade);
      ...
}

//ftFqcEndForm.js
var shipNoteComboStore = Ext.create('Ext.data.Store', {
	fields: ['key', 'value'],
	proxy: {
		type: 'ajax',
		actionMethods: { read: 'GET' },
		url: '/mycim2/ftFqc.do?action=queryShipNote',
		queryMask: false,
	},
	autoLoad: true
});

Ext.doEndSearch = function() {
    ...
    Ext.Ajax.request({
        url: '/mycim2/ftFqc.do?action=queryEndVBox',
        params: {
            vboxId: vboxId,
        },
        success: function(response) {
			          ...
                shipNoteComboStore.getProxy().extraParams.productId = data.productId;
                shipNoteComboStore.getProxy().extraParams.grade = data.grade;
                shipNoteComboStore.load();
            } else {
                MyCim.notify.alert(responseJson.msg);
            }
        }
    })
};
```

### 制造管FT功能管理-FT批次入库备注修改

```java
//FtLotStorageRemarkModifyAction.java
private ActionForward remarkStroe(HttpServletRequest request, HttpServletResponse response) {
		//List<String> storageRemarkList = ftService.getStorageRemark(Lot.STATUS_FT);
		//List<String> _storageRemarkList = ftService.getStorageRemark(Lot.STATUS_WLFT);
		List<String> storageRemarkList = new ArrayList<>();
		String productId = request.getParameter("productId");
		String grade = request.getParameter("binDesc");
		List<GcFtStorageRemarkConfig> FtStorageRemarkConfigs = ftService.getValidGcFtStorageRemarkConfigByProductModelAndGrade(productId, grade);
		for(GcFtStorageRemarkConfig gcFtStorageRemarkConfig : FtStorageRemarkConfigs){
			storageRemarkList.add(gcFtStorageRemarkConfig.getRemarkDescribe()+"_"+gcFtStorageRemarkConfig.getShipmentType());
		}
		Map keyValueMap = new HashMap<>();
		keyValueMap.put("key", ");
		keyValueMap.put("value", "");
		keyValueMapList.add(keyValueMap);
		//storageRemarkList.addAll(_storageRemarkList);
}

//lotStorageRemarkModifyForm.js
function selectedLine() {
	var records = Ext.getCmp('LotStorageRemarkModifyGrid').getSelectionModel().getSelection();
	if (records.length < 1){
		return;
	}
	var flagProductId = 1;
	var flagGrade = 1;
	for (var i = 1; i < records.length; i++) {
		if (records[i].data.productId !== records[i-1].data.productId){
			flagProductId = 0;
			break;
		}
	}
	for (var j = 1; j < records.length; j++) {
		if (records[j].data.binDesc !== records[j-1].data.binDesc
			|| records[j].data.binDesc.split(',').length > 1
			|| records[j-1].data.binDesc.split(',').length > 1){
			flagGrade = 0;
			break;
		}
	}
	if (flagProductId === 1){
		remarkStroe.getProxy().extraParams.productId = records[0].data.productId;
	}else {
		remarkStroe.getProxy().extraParams.productId = 'ALL';
	}
	if (flagGrade === 1){
		remarkStroe.getProxy().extraParams.binDesc = records[0].data.binDesc.split(',')[0];
	}else {
		remarkStroe.getProxy().extraParams.binDesc = 'ALL';
	}
	//binDesc:"MA,",之后根据一个产品型号对应多个等级的解决，来处理
	remarkStroe.load();
}

//lotStorageRemarkModifyGrid.js
Ext.define('LotStorageRemarkModifyViewer.LotStorageRemarkModifyGrid', {
	...
	listeners : {
		itemclick : function(sm) {
			selectedLine();
		}
	},
	...
}
```

### 制造管》包装操》FT盒包》FT盒包

```java
//FtPackageLotAction.java
private ActionForward shipNote(ActionMapping mapping, HttpServletRequest request, HttpServletResponse response,
			Long facilityRrn) {
		...
		try{
		    String type = request.getParameter("group");
		    String jsonArrayStr = com.mycim.core.util.StringUtils.EMPTY;
		    if(StringUtils.equals(type, "SPECIAL")||StringUtils.equals(type, "AUTO")){
						..
						String productId = request.getParameter("productId");
						String grade = request.getParameter("grade");
						List<GcFtStorageRemarkConfig> FtStorageRemarkConfigs = ftService.getValidGcFtStorageRemarkConfigByProductModelAndGrade(productId, grade);
						...
				}
		}
}

//FtPackageLotForm.js
var shipNoteComboStore = Ext.create('Ext.data.Store', {
	fields: ['key', 'value'],
	proxy: {
		type: 'ajax',
		actionMethods: { read: 'GET' },
		url: '/mycim2/ftPackageLot.do?action=shipNote',
		queryMask: false,
		reader : {
			root : 'msg.data'
		}
	}
});

listeners: {
    blur: function(combo, value) {
        if (!Ext.isEmpty(combo.rawValue)) {
            ...
            shipNoteComboStore.getProxy().extraParams.productId = Ext.getCmp('productId').getValue();
            shipNoteComboStore.getProxy().extraParams.grade = Ext.getCmp('grade').getValue();
            shipNoteComboStore.getProxy().extraParams.group = combo.rawValue;
            shipNoteComboStore.load();
        } else {
            ...
        }
    }
}
```

### 制造管》包装操》FT真空包包》FT真空包包

```java
//FtVboxPackageAction.java
private ActionForward shipNote(HttpServletRequest request, HttpServletResponse response) {
		  ...
			//List<GcFtStorageRemarkConfig> FtStorageRemarkConfigs = ftService.getGcFtStorageRemarkConfigAll();
			String productId = WebUtils.getParameter("productId", request);
			String grade = WebUtils.getParameter("grade", request);
			List<GcFtStorageRemarkConfig> FtStorageRemarkConfigs = ftService.getValidGcFtStorageRemarkConfigByProductModelAndGrade(productId, grade);
			...
}

//FtVboxPackageTBoxForm.js
Ext.Ajax.request({
    	url: '/mycim2/ftVboxPackage.do?action=queryPackage',
    	...
    	success: function(response) {
			...
    	if(noReReplaceFlags.length > 1){
    			replaceFlags.splice(-1,1);
    			MyCim.notify.alert(i18n_msg_replace_Flag_is_differ);
    			return;
    	}
			...
			getFtVboxPackageShipNote(data.productId, data.grade);
    	...
}

//FtVboxPackageTBoxGrid.js
function getFtVboxPackageShipNote(productId, grade) {
	productIdComboStore.getProxy().extraParams.productId = productId;
	productIdComboStore.getProxy().extraParams.grade = grade;
	productIdComboStore.load();
}
```

### F600-ATE-并批/合批 待出时的出入组操

```java
//LotGroupOperationOperatorAction.java
private ActionForward comboValueQuery(ActionMapping mapping, HttpServletRequest request,
                                          HttpServletResponse response) {
		...    
		String lotBinString = WebUtils.getParameter("lotBinString", request);
    String productId = WebUtils.getParameter("productId", request);
    List<GcFtStorageRemarkConfig> FtStorageRemarkConfigs = ftService.getValidGcFtStorageRemarkConfigByProductModelAndGrade(productId, lotBinString);
		...
}

//lotGroupOperationCommon.js
function doGroupOperation() {
	...
	if(opercationId == "FPQC" && dieGroupFlag == "待出){
		showSavePackedPropertyWindow();
		selectedLine(records);
	} else {
		doLotGroupOperation();
	}
}

function selectedLine(records) {
	if (records.length < 1){
		return;
	}
	var flagProductId = 1;
	var flagGrade = 1;
	for (var i = 1; i < records.length; i++) {
		if (records[i].data.productId !== records[i-1].data.productId){
			flagProductId = 0;
			break;
		}
	}
	for (var j = 1; j < records.length; j++) {
		if (records[j].data.lotBinString !== records[j-1].data.lotBinString
			|| records[j].data.lotBinString.split(' ').length > 1
			|| records[j-1].data.lotBinString.split(' ').length > 1){
			flagGrade = 0;
			break;
		}
	}
	if (flagProductId === 1){
		lotGroupOperationShipNoteStore.getProxy().extraParams.productId = records[0].data.productId;
	}else {
		lotGroupOperationShipNoteStore.getProxy().extraParams.productId = 'ALL';
	}
	if (flagGrade === 1){
		lotGroupOperationShipNoteStore.getProxy().extraParams.lotBinString = records[0].data.lotBinString.split(':')[0];
	}else {
		lotGroupOperationShipNoteStore.getProxy().extraParams.lotBinString = 'ALL';
	}
	//binDesc:"MA,",之后根据一个产品型号对应多个等级的解决，来处理
	lotGroupOperationShipNoteStore.load();
}

//LotSavePackedPropertyWindow.js
var lotGroupOperationShipNoteStore = Ext.create('Ext.data.ArrayStore', {
	fields : [ 'key1Value', 'data1Value' ],
	proxy : {
		queryMask: false,
		type : 'ajax',
		url : '/mycim2/lotGroupOperationOperator.do?action=comboValueQuery',
		params : {
			test : 'test'
		},
		reader : {
			root : 'msg.data'
		}
	}
});

```

### 制造管》包装操》FT_PKG-》入库单打印

```java
//WltInStorageAction.java
private ActionForward wltInStorageNoteCombo(ActionMapping mapping, ActionForm form, HttpServletRequest request,
			HttpServletResponse response, String userName, Long facilityRrn) {
		...
		String productId = request.getParameter("productId");
		String grade = request.getParameter("grade");
		List<GcFtStorageRemarkConfig> FtStorageRemarkConfigs = ftService.getValidGcFtStorageRemarkConfigByProductModelAndGrade(productId, grade);	
		...
}

//ftInStorageForm.js
function inStorageShipNoteStore() {
	var records = Ext.getCmp('FtInStorageGrid').getSelectionModel().getSelection();
	shipNoteStore.getProxy().extraParams.productId = records[0].data.productId;
	shipNoteStore.getProxy().extraParams.grade = records[0].data.grade;
	shipNoteStore.load();
}

Ext.doInStorageSearch = function(flag) {
    ...
    Ext.Ajax.request({
        ...
        success: function(response) {
            var responseJson = Ext.JSON.decode(response.responseText);
            if (responseJson.success != null && responseJson.success) {
                ...
                if (inStorageStore.data.length == 1) {
                    inStorageStore.each(function(record, index) {
                        ...
                        inStorageShipNoteStore();
                    });
                }
                ...
                Ext.getCmp('inStorageVboxId').focus(false, 100);
            } else {
                MyCim.notify.alert(responseJson.msg);
            }
        }
    })
};

//ftInStorageGrid.js
Ext.define('FtPkgViewer.FtInStorageGrid', {
  ...
  listeners:{
	  itemclick : function(sm) {
		  inStorageShipNoteStore();
	  }
  },
	...
}

var shipNoteStore = Ext.create('Ext.data.ArrayStore', {
	fields : [ 'key1Value', 'data1Value' ],
	proxy : {
		queryMask: false,
		type : 'ajax',
		url : '/mycim2/wltInStorage.do?action=comboValueQuery',
		reader : {
			root : 'msg.data'
		},
		actionMethods: { read: 'GET' },
	}
});
```

### 制造管》包装操》FT零头合批

```java
//ftOddMergeForm.js
{
		...
    fieldLabel: i18n_fld_package_property + i18n_fld_Red_Star,
    columnWidth: 0.35,
    listeners: {
        blur: function(combo, value) {
            if (!Ext.isEmpty(combo.rawValue)) {
                var shipNote = Ext.getCmp('shipNote');
                shipNote.setValue('');
                shipNoteComboStore.getProxy().extraParams.productId = Ext.getCmp('productId').getValue();
                shipNoteComboStore.getProxy().extraParams.grade = Ext.getCmp('grade').getValue();
                shipNoteComboStore.getProxy().extraParams.group = combo.rawValue;
                shipNoteComboStore.load();
            }
						...
        }
    }
}
```

# FT入库备注（定时任务更新备注有效值）

🔥 因为代码是每次查询后才会更新有效值，如果不查询，直接在其他页面下拉获取入库备注时，还是会把过期的入库备注返回的。因此之前做[FT入库备注](https://www.notion.so/FT-ecc0c07f6c404589a145f8c47e31ac5c?pvs=21) 需要制作一个定时任务，用于每次隔段时间来更新下有效值
## 创建定时

```java
package com.mycim.gc.service.quartz.jobs;

import com.mycim.AppContext;
import com.mycim.ems.model.ChecklistJob;
import com.mycim.gc.service.FtService;
import com.mycim.prp.model.GcFtStorageRemarkConfig;
import org.quartz.*;

import java.text.ParseException;
import java.util.Calendar;
import java.util.Date;
import java.util.List;

public class UpdateRemarkValidJob implements Job {

    protected FtService ftService = AppContext.getBean(FtService.class);

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {

        List<GcFtStorageRemarkConfig> gcFtStorageRemarkList = ftService.getByProductClassifyAndRemarkCode(null,null);

        for (GcFtStorageRemarkConfig c: gcFtStorageRemarkList) {
            try {
                if (updateIsValid(new Date(), c.getTimely(), c.getCreateTime()) != 1){
                    c.setIsValid(0);
                    ftService.updateGcFtStorageRemarkConfig(c);
                }
            } catch (ParseException e) {
                throw new RuntimeException(e);
            }
        }
    }

    private int updateIsValid(Date currentDate, String timely, Date createTime) throws ParseException {
        if("ALL".equals(timely)){
            return 1;
        }else {
            int i = Integer.parseInt(timely);
            java.util.Calendar c = java.util.Calendar.getInstance();
            c.setTime(createTime);
            c.add(Calendar.DATE, i);
            Date newDate = c.getTime();//有效期的最后一
            return newDate.after(currentDate) ? 1 : 0;
        }
    }
}
```

## 新建配置项：applicationContext-quartz.xml

```xml
<!-- UPDATE REMARK IS_VALID -->
    <bean name="updateRemarkValidJobDetail"  class="org.springframework.scheduling.quartz.JobDetailFactoryBean">
        <property name="jobClass" value="com.mycim.gc.service.quartz.jobs.UpdateRemarkValidJob" />
        <property name="jobDataMap">
            <map>
                <entry key="service" value="timelimit"/>
            </map>
        </property>
        <property name="durability" value="true" />
    </bean>
    <bean id="updateRemarkValidCronTrigger"  class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
        <property name="jobDetail" ref="updateRemarkValidJobDetail" />
        <property name="cronExpression" value="0/2 * * * * ?" />
    </bean>

...
<beans profile="development">或beans profile="production">
		<bean id="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
			...
			<property name="jobDetails">
	            <list>
									<ref bean="updateRemarkValidJobDetail" />
	            </list>
	        </property>
	        <property name="triggers">
	            <list>
									<ref bean="updateRemarkValidCronTrigger" />
	            </list>
	        </property>
	    </bean>
	</beans>
```

## application.properties

```
# Properties file for use by myCIM
# Config FTP server
spring.profiles.active=production
ftp.server.host=FTP-SERVER
ftp.server.user=mycim
ftp.server.password=mycim

# Config Redis server
redis.host=127.0.0.1
redis.port=6379
redis.pass=

mail.user=958823167@qq.com
```

## setDomainEnv.cmd

修改环境为production，与项目保持一
![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241017135644.png)

# 新增路径字段

📪 此次增加新字段，无技术困难，实体类，数据表不赘述

```sql
@Column(name="ROUTE_ID")
	private String routeId;

if(StringUtils.isNotEmpty(gcFtStorageRemarkConfig.getRouteId())){
		sqlBuilder.and("ROUTE_ID").eq(gcFtStorageRemarkConfig.getRouteId());
}
```