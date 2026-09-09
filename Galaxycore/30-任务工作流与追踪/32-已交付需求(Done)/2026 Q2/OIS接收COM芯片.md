---
title: OIS接收COM芯片
date: 2026-05-03
tags: [MES, 工作流]
---
# OIS COM 芯片接收功能修改报告

## 一、功能概述

本次修改实现了一个类似晶圆线边仓接收的 OIS 专用功能，支持扫描箱号/载具号自动查询和接收 OIS COM 芯片，同时增加了从工单关联数据自动创建接收记录的能力。

## 二、变更内容

### 1. 前端优化
- **移除了待绑数据功能**：删除了 `oisComChipReceiveBindForm.js` 和 `oisComChipReceiveBindGrid.js` 文件
- **简化页面布局**：从原有的"查询+绑定"双区域布局简化为单一的"扫描+展示"布局
- **清理引用**：更新了 `oisComChipReceive.jsp` 中的 script 引用，移除了已删除文件的引用

### 2. 后端增强
- **移除冗余功能**：删除了 `OisComChipReceiveAction` 中的 `searchBind` 操作
- **增加新功能**：
  - 在 `ToolService` 接口中新增 `searchAndCreateOisWaferFromWorkOrderRelation` 方法
  - 在 `ToolServiceImpl` 中实现该方法，支持从工单关联数据自动创建接收记录
  - 修改 `OisComChipReceiveAction` 的 `queryWaferList` 方法，实现双路径查询逻辑

## 三、技术实现

### 1. 双路径查询逻辑
```
扫描箱号/载具号
  ├─ 路径1: BACKEND_WAFER_RECEIVE 中存在 → 直接返回
  └─ 路径2: BACKEND_WAFER_RECEIVE 中不存在
       ├─ 通过 boxId 查 WmsMmsMaterialLot（或通过 cstId 查 WmsMmsMaterialLotUnit → 再查 WmsMmsMaterialLot）
       ├─ 校验 WmsMmsMaterialLot.STATUS = 'Issue'
       ├─ 查 WorkOrderRelation（OBJECT_ID = materialLotId）
       ├─ 校验 WmsMmsMaterialLot.WORK_ORDER_ID = WorkOrderRelation 对应工单的 workorderId
       ├─ 自动创建 BACKEND_WAFER_RECEIVE 记录（主记录 + 子单元记录）
       └─ 保存历史记录并返回数据
```

### 2. 关键代码
- **前端**：`oisComChipReceiveViewer.js` - 简化布局，移除 Bind 相关组件
- **后端**：
  - `OisComChipReceiveAction.java` - 增强搜索逻辑，支持双路径查询
  - `ToolServiceImpl.java` - 实现 `searchAndCreateOisWaferFromWorkOrderRelation` 方法
  - `ToolService.java` - 新增接口方法定义

## 四、关键设计点

1. **兼容性**：保持了原有功能的完整性，仅移除了不必要的 Bind 功能
2. **扩展性**：新增的双路径查询逻辑可适用于其他类似场景
3. **数据一致性**：自动创建 `BACKEND_WAFER_RECEIVE` 记录时，同时创建对应的历史记录
4. **性能优化**：优先查询 `BACKEND_WAFER_RECEIVE`，减少复杂关联查询的次数
5. **错误处理**：对各种异常情况进行了合理的错误处理和返回

## 五、预期效果

1. **操作简化**：用户只需扫描箱号或载具号，系统会自动处理数据查询和创建
2. **数据完整**：即使 `BACKEND_WAFER_RECEIVE` 中不存在记录，只要工单关联数据完整且物料状态为 Issue，也能正常扫描和接收
3. **用户体验**：移除了不必要的 Bind 功能，界面更加简洁明了
4. **系统集成**：与现有工单系统和物料管理系统实现了更好的集成

## 六、使用说明

1. **扫描操作**：在输入框中扫描箱号或载具号，按回车键或点击"查询"按钮
2. **数据展示**：系统会自动查询并展示相关的 OIS COM 芯片数据
3. **接收操作**：选择需要接收的芯片，点击"接收"按钮完成接收操作

此功能现已完全就绪，可以投入使用。
