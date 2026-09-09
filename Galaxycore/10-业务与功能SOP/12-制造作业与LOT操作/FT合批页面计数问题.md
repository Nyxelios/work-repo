# [嘉善MES 0000237]: 待入组、待出组、FT合批、开始/结束页面无计数显示

标签: 站点
创建时间: 2022年7月27日 10:26
上次编辑时间: 2024年10月29日 16:39
状态: Not started

摘要:

待入组、待出组、FT合批页面无计数显示

说明:

待入组、待出组，FT合批、展示待作业的总批次数

# FT合批

```jsx
//ftLotMergeForm.js
{
		xtype : 'textfield',
		name : 'countNum',
		id : 'countNum',
		columnWidth : 0.15,
		readOnly: true,
		value: '0',
		fieldLabel : '已选行数'
}

function countSelectedLine() {
	var records = Ext.getCmp('FtLotMergeGrid').getSelectionModel().getSelection();
	Ext.getCmp('countNum').setValue(records.length);
}

//ftLotMergeGrid.js
listeners : {
		selectionchange : function(sm) {
			countSelectedLine();
		}
	},
```

# 待入组、待出组

```jsx
//LotGroupOperationOperatorForm.js
function countSelectedLine() {
	var records = Ext.getCmp('lotGroupOperationGrid').getSelectionModel().getSelection();
	console.log(records.length);
}

{
	xtype: 'textfield',
	name: 'countNum',
	id:'countNum',
	fieldLabel: '已选行数',
	columnWidth: 0.2,
	value: '0',
	enableKeyEvents:true,
	readOnly: true
}

//lotGroupOperationGrid.js
listeners : {
		selectionchange : function(sm) {
			countSelectedLine();
		}
	},
```