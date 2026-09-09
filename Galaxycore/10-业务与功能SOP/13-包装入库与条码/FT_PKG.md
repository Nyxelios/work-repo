---
title: FT PKG 包装
date: 2024-10-21
author: MES Team
status: completed
tags: [包装, FT, 功能]
---


# FT-PKG节点PKG结束动作可批量进行
## 摘要

FT-PKG节点PKG结束动作可批量进行

## 说明

FT-PKG节点PKG结束动作设定改进为：可扫描1至N包，只需1次结束。即改善后员工仍需扫描400次，但结束可以控制在10次以内。

**`Due date => 2022-08-15`**

## 介绍
![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241017151534.png)

> PKG结束页面每次输入一个VBOX的ID回车，表格内会把之前的搜索的数据清除，然后显示新的查询的一条数据，这不是需要的效果。

> 更新后，每次输入箱号回车，会在原先数据中添加新的数据，不会清除原先的数据。

## 代码

### FtPkgEndForm.js

```java
Ext.dopkgEndSearch = function(flag) {
    ...
    let limitRows = 40;
    if ('查看' == flag.text) {
        pkgEndStore.removeAll();
    }
    if (pkgEndStore.data.length >= limitRows) {
        MyCim.notify.alert('至多存储' + limitRows + '行数据');
        Ext.getCmp('pkgEndVboxId').focus(false, 100);
        return;
    }
    Ext.Ajax.request({
        url: '/mycim2/ftPkg.do?action=query',
        params: {
            vboxId: vboxId,
            actionType: actionType
        },
        ...
    })
};
```