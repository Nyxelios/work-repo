---
title: ENG物料与正常物料无区分-目的：ENG物料用不同底色展示，增加辨识度
date: 2026-05-03
tags: [MES, LOT]
---
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
    getRowClass: function(record, rowIndex, store) { // 根据状态改变当前行字体颜色
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