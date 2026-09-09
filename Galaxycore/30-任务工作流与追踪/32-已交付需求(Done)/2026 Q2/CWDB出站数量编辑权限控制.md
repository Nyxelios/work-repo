# CWDB出站数量编辑权限控制

> **状态**：✅ 已完成 | **SVN**：r845 (2026-05-08)

## 功能说明

CWDB站出站时，`moveOutQty` 字段默认取自 `ComAteUpload.BondPassCnt`，并通过按钮权限 `BTN_CWDB_IN_QTY_EDIT` 控制是否可编辑。

## 变更内容

### MoveOutAction.java:705-714（新增）

```java
// CWDB出站数量编辑权限控制：只有拥有BTN_CWDB_IN_QTY_EDIT权限的角色才能修改
Collection<Long> roles = userService.getUserRoleList(ThreadLocalContext.getUserRrn());
Map cwdbButton = utilService.getButtonInfoWithRoles(roles, "BTN_CWDB_IN_QTY_EDIT");
request.setAttribute("cwdbQtyEditable", MapUtils.isNotEmpty(cwdbButton) ? "true" : "false");
```

### moveout.jsp:244-252（修改）

```jsp
<logic:notEmpty name="rootForm" property="cwdbQty">
    <c:choose>
        <c:when test="${cwdbQtyEditable eq 'true'}">
            <td><input name="moveOutQty" type="text" value="${cwdbQty}"></td>
        </c:when>
        <c:otherwise>
            <td><input name="moveOutQty" readonly class="myreadonly" value="${cwdbQty}"></td>
        </c:otherwise>
    </c:choose>
</logic:notEmpty>
```

## 权限配置

在系统按钮权限管理中配置 `BTN_CWDB_IN_QTY_EDIT`，分配给需要修改数量的角色（如测试工程师、产线主管）。

## 测试场景

| 场景 | 预期结果 |
|------|---------|
| 有权限用户 + ATE数据 | 输入框可编辑，默认值=BondPassCnt |
| 无权限用户 + ATE数据 | 输入框只读（readonly+myreadonly） |
| 无ATE数据 | 不渲染该字段（走默认逻辑） |

## 注意事项

- 仅取第一个批次的ATE数据（`lots[0]`）
- 当前仅前端控制readonly，⚠️ 建议后端提交时增加二次校验
