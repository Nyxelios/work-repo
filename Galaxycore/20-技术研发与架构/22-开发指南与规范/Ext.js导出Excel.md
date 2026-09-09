---
title: Ext.js导出Excel
date: 2026-05-03
tags: [MES, 开发]
---
#导出Excel
# 第一步、添加导出按钮
```javascript
{
	xtype: 'button',
	text: i18n_fld_exportExcel,
	columnWidth: 0.5,
	region: 'center',
	handler: Ext.exportExcel
}
```
# 第二部、编辑导出方法
```jsx
Ext.exportExcel = function(){
	var exportInfoJson = exportInfoJson || {};
	var headers = new Array();
	var rows = new Array();

	var me = Ext.getCmp('FTQsGradeBoxGrid');//替换FTQsGradeBoxGrid
	var view = me.up("viewport");
	var form = view.down("form").getForm();
	var record = store2.getAt(0);//替换store2

	var rowIndex = store2.getCount();//替换store2
	var colIndex = me.columns.length;

	for(var i = 0; i< colIndex ; i++){
		var data = {};
		var colId = me.columns[i].dataIndex;
		if(colId == 'seq' || colId == 'operation'){
			continue;
		}
		var colName = me.columns[i].text;
		var colWidth = 100;
		data.colId = colId;
		data.colName = colName;
		data.width = colWidth;
		data.hidden = me.columns[i].hidden;
		headers.push(data);
	}

	exportInfoJson.headers = headers;
	var items = new Array();
	items = store2.data.items;//替换store2
	for(var i = 0; i< items.length ; i++){
		rows.push(items[i].data);
	}
	exportInfoJson.rows = rows;
	exportInfoJson.baseInfo = {};
	var fd=Ext.get('frmDummy');
	if (!fd) {
		var fd = Ext.DomHelper.append(Ext.getBody(), {
			tag: 'form',
			method: 'post',
			id: 'frmDummy',
			action: '/mycim2/exportExcel.do?action=exportToExcel',
			target: '_self',
			name: 'frmDummy',
			cls: 'x-hidden',
			cn: [{
				tag: 'input',
				name: 'exportInfoJson',
				id: 'exportInfoJson',
				type: 'hidden'
			}]
		}, true);
	}
	fd.child('#exportInfoJson').set({value:Ext.encode(exportInfoJson)});
	fd.dom.submit();
}
```
# 已开发功能
## FT 入库备注配置
*0000434 FT 入库备注配置界面增加“导出数据”按钮*
FT 入库备注配置界面，有查看显示的功能，但是还缺一个“导出list”的功能，即我们每周要导出一下这个表格，打印出来供QC员工检验核对；
如上午所讨论，麻烦增加1个“导出数据”按钮
## F3等级入库，生产损耗入库增加导出功能
在左下的表单制作一个导出excel的功能

![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241017103008.png)

- `fThreeBoxPackageGrid.js`
- `ftQsGradeBoxGrid.js`
两个文件修改的位置相似
