---
title: LOT作业-前端颜色显示修改
date: 2022-07-25
tags: [MES, LOT, 前端]
---
# 前端颜色显示修改与回车查询

## 任务的开端

![[前端颜色显示修改-Untitled.png]]

![[前端颜色显示修改-Untitled 1.png]]

> 💡 在7月8号，王欣琪离职了，李智威给我分配了一个任务，初看感觉比较简单，是对前端页面的排版重新搞下。

## DO

![[前端颜色显示修改-Untitled 2.png]]

从发给我的图知道，需要修改开始界面的排版，于是我在浏览器开启F12调试，然后锁定到这个页面的名字信息，之后就在项目中修改代码。

> 💡 可以知道，开始界面的文件名是包含start的，同时要修改的是表格的列名，那就是在grid.js文件中。即**`ftFqcStartGrid.js`**

## 代码修改

```jsx
......
columns: [{
            xtype: 'rownumberer',
            id: 'startSeq',
            align: 'center',
            flex: 1,
            sortable: false,
            header: i18n_fld_SEQ
          },{
			xtype : 'gridcolumn',
			dataIndex : 'workorderId',
			width:160,
			align : 'center',
			sortable : true,
			header : i18n_msg_WorkOrderId,
			hidden: true,
		},{
			xtype : 'gridcolumn',
			dataIndex : 'productId',
			align : 'center',
			width:160,
			sortable : true,
			header : i18n_msg_productModel
		},{
			xtype : 'gridcolumn',
			dataIndex : 'boxId',
			width:160,
			align : 'center',
			sortable : true,
			header : i18n_fld_Vbox_Id
		},{
			xtype : 'gridcolumn',
            dataIndex : 'levelTwoCode',
            align : 'center',
            width:100,
			sortable : true,
			header : i18n_fld_SecondCode
		},{
			xtype : 'gridcolumn',
			dataIndex : 'grade',
			align : 'center',
			width:80,
			sortable : true,
			header :i18n_fld_grade
		}, {
			xtype : 'gridcolumn',
			dataIndex : 'quantity',
			align : 'center',
			width:80,
			sortable : true,
			header : i18n_msg_Count
		},{
			xtype : 'gridcolumn',
			dataIndex : 'finalOperationTime',
			align : 'center',
			width:160,
			sortable : true,
			header :i18n_fld_finalOperatorTime
		},{
			xtype : 'gridcolumn',
			dataIndex : 'treasuryNote',
			align : 'center',
			width:160,
			sortable : true,
			header :i18n_fld_Ship_Note
		},{
			xtype : 'gridcolumn',
			dataIndex : 'processId',
			align : 'center',
			width:160,
			sortable : true,
			header : i18n_msg_Process_ID
		},
......
]
```

> 💡 主要就是修改column数组的顺序即可，工单号不需要就设置hidden属性隐藏掉就🆗了。

## 交付

说实话，只花了我十分钟，正当我准备交差，然后李智威告诉我其实待入组，出组，开始，结束都要修改，和开始的排版类似，同时对于最后操作的时间的格式要进行更改，格式为：**`2022-06-01  11：05：13`**

没办法，那我只能照做了。

## DO 2

首先我打算把四个页面的前端的列名都搞下，这个比较简单，直接复制start的，然后删去没有的列名，修改下列名的属性就好了。

完事之后，遇到了比较头疼的问题，也是我之前做胶水金线遇到的，金线胶水的数据结构中，关于时间的**`数据类型是DATE`**的，这个没问题，但是传到前端的时间需要时字符串的，但是这种字符串格式并不规范，是**`"26/02/2022 12:46:02"** 的，2022年的2月26日，这种格式就很奇怪，因为之前是重新新建了一个实体类，然后把日期改成了string类，**`也就是在java中做了格式转换的工作`**，不过这样做似乎没有得到王欣琪的赞同，所以这次尝试在js中修改格式。

## T1

> 💡 在form.js文件中添加转换的代码

```jsx
Ext.doTransDate = function dateFormat(fmt, date) {
    let ret;
    const opt = {
        "Y+": date.getFullYear().toString(),
        "m+": (date.getMonth() + 1).toString(),
        "d+": date.getDate().toString(),
        "H+": date.getHours().toString(),
        "M+": date.getMinutes().toString(),
        "S+": date.getSeconds().toString()
    };
    for (let k in opt) {
        ret = new RegExp("(" + k + ")").exec(fmt);
        if (ret) {
            fmt = fmt.replace(ret[1], (ret[1].length == 1) ? (opt[k]) : (opt[k].padStart(ret[1].length, "0")))
        };
    };
    return fmt;
}
```

这样做确实是可以但是，因为这个日期格式太奇怪了，把**`2021/12/9当成了2021/9/12`**

这个问题就很严重，没办法，那我只能试试java中的格式修改下试试了。

> 💡 等等，既然他月份和日期交换了位置，那我直接在`"yyyy-MM-dd HH:mm:ss"` 修改不就行了嘛？`"yyyy-dd-MM HH:mm:ss"`

![[前端颜色显示修改-Untitled 3.png]]

好吧，还是不行，只是最后的步骤位置交换，但他转不了date类型就没有后续了。还是从java中改改吧。

## T2

准备在实体类加注解**`@JsonFormat(pattern="yyyy-MM-dd HH:mm:ss",timezone="GMT+8")`**

结果还是失败！

> 💡 意外发现，这里是由字符串类型的，原来只需要加一个字段就行了。学到了。我以为加一下会影响数据库的映射。

![[前端颜色显示修改-Untitled 4.png]]

```jsx
var data = responseJson.msg.data;
for (i = 0; i < data.length; i++) { 
    data[i].finalOperationTime = data[i].stringFinalOperationTime;
}
startStore.add(data);
```

```java
SimpleDateFormat simpleDateFormat1 = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
String stringFinalOperationTime = simpleDateFormat1.format(packedLot.getFinalOperationTime());
packedLot.setStringFinalOperationTime(stringFinalOperationTime);
```

**`终于解决了一个困扰的问题😋`**

## 后期

又增加了两个任务，不过时间不急，可以慢慢来。

![[前端颜色显示修改-Untitled 5.png]]

## Add Q

> 新增了一个小需求，在出组的表格增加一个最后操作的时间列

![[前端颜色显示修改-Untitled 6.png]]

```java
private JSONObject outGroupLotsToJsonObject(Lot lot, String binId, Long lotDieBinQty) {
		PackedLot packedLot = packageService.getPackedVBoxInfoByVboxId(lot.getLotId());
		String stringFinalOperationTime = null;
		if(packedLot != null){
			treasuryNote = packedLot.getTreasuryNote();
			SimpleDateFormat simpleDateFormat1 = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
			stringFinalOperationTime = simpleDateFormat1.format(packedLot.getFinalOperationTime());
			lot.setStringFinalOperationTime(stringFinalOperationTime);
		}
		packedLotJson.put("stringFinalOperationTime", stringFinalOperationTime);
		return packedLotJson;
	}
```

> 💡 在封装成json的那一步，put增加一个字段，传入最后操作的时间，key对应前端的grid的属性值。

```jsx
Ext.define('FtFqcOutGroupModel',{
	extend:'Ext.data.Model',
	fields: [
	{
	  name: 'stringFinalOperationTime',
	}]
})

columns: [
		{
			xtype : 'gridcolumn',
			dataIndex : 'stringFinalOperationTime',
			align : 'center',
			width:160,
			sortable : true,
			header :i18n_fld_finalOperatorTime
		}
```

## [嘉善MES 0000260]: waferId/lotId输入后自动回车查询

`waferId/lotId`输入后自动回车查询

`FT`业务相关功能所有`waferId/lotId`查询的功能都需要有自动回车查询功能

## [嘉善MES 0000238]: ENG物料与正常物料无区分

> 这次的任务负责将页面的表格中的某一满足条件的行的背景色变成黄色。判断条件是processID 流程号的前缀是ENG开头的就变黄色。同时一个窗口有三个小窗口，所以要改动三处。

**`JobManagementPanel.ui.js`**

```jsx
id: 'SelectedLotList',
	viewConfig: {
		getRowClass: function (record, rowIndex, rowParams, store) {
			if (record.data.processId.startsWith("ENG")) {
				return 'x-grid-record-yellow';
			}
		}
	},

id: 'RunningLotList',
	viewConfig: {
		getRowClass: function (record, rowIndex, rowParams, store) {
			if (record.data.processId.startsWith("ENG")) {
				return 'x-grid-record-yellow';
			}
		}
	},

id: 'AvailableLotList',
	viewConfig: {
		getRowClass: function (record, rowIndex, rowParams, store) {
			if (record.data.LotStatus == 'HOLD') {
				return 'x-grid-record-red';
			}else if (record.data.processId.startsWith("ENG")) {
				return 'x-grid-record-yellow';
			}
		}
	},
```

**`JobManagementPanel.css`**

```css
tr.x-grid-record-yellow .x-grid-td {
    background: #FABB3D;
}
```

**`lotGroupOperationGrid.js`**

```jsx
viewConfig: {
    getRowClass: function(record, rowIndex, store) {
        var status = record.get('lotStatus');
        var processIdStr = record.data.processId.substr(0, 3);
        if (status == "HOLD") {
            return 'x-grid-row-red';
        } else if (processIdStr == "ENG") {
            return 'x-grid-record-yellow';
        } else {
            return 'x-grid-row-black';
        }
    }
},
```
