---
title: 新增COM退库入库备注
date: 2026-05-03
tags: [MES, 工作流]
---
# 工单变更表单 —「决议编号有无」联动控制功能说明文档

## 1. 功能概述

在**工单变更（Com Note Change）** 表单中，当用户选择 **"指定出"** 类型时，会出现一个 **「决议编号有无」** 的下拉选项。根据用户选择 **"有"** 或 **"无"**，表单会动态显示/隐藏相关字段，并控制这些字段的可用状态（启用/禁用）。

## 2. 涉及文件

| 文件路径 | 说明 |
|---------|------|
| `core/web/wip/comNoteChange/comNoteChangeForm.js` | 工单变更表单前端代码 |

## 3. 涉及字段

| 字段 ID                | 中文名称   | 类型             |
| -------------------- | ------ | -------------- |
| `ifResolutionNumber` | 决议编号有无 | 下拉框（触发联动的源头字段） |
| `customer`           | 客户     | 下拉框（联动受控字段）    |
| `customerId`         | 客户ID   | 隐藏字段（联动受控字段）   |
| `resolutionNumber`   | 决议编号   | 文本输入框（联动受控字段）  |

## 4. 业务逻辑流程


### 详细行为说明：

#### 选择 **"有"** 时：
| 操作 | 目标字段 | 说明 |
|-----|---------|------|
| `show()` | customer | 显示客户下拉框 |
| `setDisabled(false)` | customer | ✅ 启用客户下拉框（可编辑） |
| `setValue('')` | customer | 清空当前值 |
| `show()` | customerId | 显示客户ID字段 |
| `setValue('')` | customerId | 清空当前值 |
| `show()` | resolutionNumber | 显示决议编号输入框 |
| `setDisabled(false)` | resolutionNumber | ✅ 启用决议编号输入框（可编辑） |
| `setValue('')` | resolutionNumber | 清空当前值 |

#### 选择 **"无"** 或其他值时：
| 操作 | 目标字段 | 说明 |
|-----|---------|------|
| `hide()` + `setDisabled(true)` | resolutionNumber | ❌ 隐藏并禁用决议编号 |
| `setValue('')` | resolutionNumber | 清空值 |
| `hide()` + `setDisabled(true)` | customer | ❌ 隐藏并禁用客户 |
| `setValue('')` | customer | 清空值 |
| `hide()` | customerId | 隐藏客户ID |
| `setValue('')` | customerId | 清空值 |

## 5. 本次修改的核心变更点

### 与旧版本的差异：

| 对比项 | 旧版本 | 新版本（本次修改） |
|-------|-------|-------------------|
| 选择"有"时，customer 字段 | 仅 `show()` 显示 | **新增** `setDisabled(false)` 启用编辑 |
| 选择"有"时，resolutionNumber 字段 | 仅 `show()` 显示 | **新增** `setDisabled(false)` 启用编辑 |
| 选择"无"时，customer 字段 | **未处理**（可能残留显示） | **新增** `hide()` + `setDisabled(true)` 隐藏并禁用 |
| 选择"无"时，customerId 字段 | **未处理** | **新增** `hide()` 隐藏 |
| 选择"无"时，resolutionNumber 字段 | 仅 `hide()` 隐藏 | **新增** `setDisabled(true)` 同步禁用 |

> **关键改进：** 新增了 `setDisabled()` 调用，确保字段的 **可见性** 和 **可编辑状态** 始终保持同步，避免出现"字段可见但无法编辑"或"切换后字段残留显示"的问题。

## 6. 代码位置参考

修改位于 `comNoteChangeForm.js` 第 **220~246** 行，具体在 `ifResolutionNumber` 字段的 `listeners.change` 事件中：

```javascript
                    listeners: {
                        change: function (combo, value){
                            if(!Ext.isEmpty(value) && value == '有'){
                                Ext.getCmp('customer').show();
                                Ext.getCmp('customer').setDisabled(false);     // 新增：启用客户
                                Ext.getCmp('customer').setValue('');

                                Ext.getCmp('customerId').show();
                                Ext.getCmp('customerId').setValue('');

                                Ext.getCmp('resolutionNumber').show();
                                Ext.getCmp('resolutionNumber').setDisabled(false); // 新增：启用决议编号
                                Ext.getCmp('resolutionNumber').setValue('');
                            }else {
                                Ext.getCmp('resolutionNumber').hide();
                                Ext.getCmp('resolutionNumber').setDisabled(true);  // 新增：同步禁用
                                Ext.getCmp('resolutionNumber').setValue('');

                                Ext.getCmp('customer').hide();
                                Ext.getCmp('customer').setDisabled(true);         // 新增：同步禁用
                                Ext.getCmp('customer').setValue('');

                                Ext.getCmp('customerId').hide();
                                Ext.getCmp('customerId').setValue('');
                            }
                        }
                    }
