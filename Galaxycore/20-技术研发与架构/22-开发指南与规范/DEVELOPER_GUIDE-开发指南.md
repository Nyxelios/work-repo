# Galaxycore MES 开发者指南

## 一、开发环境搭建

### 1.1 基础环境要求

| 组件 | 版本要求 | 说明 |
|------|----------|------|
| JDK | 1.8+ | Java开发工具包 |
| Eclipse | IDE | 代码编辑器（推荐） |
| WebLogic | 12c+ | 应用服务器 |
| Oracle | 11g+ | 数据库 |
| SVN | - | 版本控制客户端 |
| Ant | 1.9+ | 构建工具 |

### 1.2 环境配置步骤

#### 1.2.1 安装配置JDK

```bash
export JAVA_HOME=/path/to/jdk1.8
export PATH=$JAVA_HOME/bin:$PATH
```

#### 1.2.2 配置WebLogic

1. 下载并安装WebLogic Server 12c+
2. 创建域（Domain）
3. 配置数据源连接Oracle数据库

#### 1.2.3 配置Oracle数据库

```sql
CREATE TABLESPACE TBS_MYCIM 
DATAFILE '/oracle/data/mycim01.dbf' 
SIZE 100M AUTOEXTEND ON;

CREATE USER mycim IDENTIFIED BY password 
DEFAULT TABLESPACE TBS_MYCIM;
GRANT CONNECT, RESOURCE TO mycim;
```

#### 1.2.4 检出源码

```bash
svn checkout http://your-svn-server/repos/gc/trunk gc
```

### 1.3 本地运行

```bash
cd Code-Projects/gc
ant -f build.xml clean
ant -f build.xml build-webapp
```

---

## 二、项目构建

### 2.1 Ant构建命令

```bash
ant -f build.xml
ant -f build-gc.xml
ant -f build-prp.xml
ant -f build-wip.xml
ant -f build-workflow.xml
```

### 2.2 模块构建说明

| 构建文件 | 模块 | 说明 |
|----------|------|------|
| `build.xml` | 主构建脚本 | 整合所有模块 |
| `build-gc.xml` | Galaxycore业务模块 | 工单、制造等业务 |
| `build-prp.xml` | PRP配置模块 | 产品、工艺配置 |
| `build-wip.xml` | WIP执行模块 | 生产执行 |
| `build-workflow.xml` | 工作流模块 | 流程引擎 |

---

## 三、代码规范

### 3.1 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 类名 | UpperCamelCase | `WorkOrderService` |
| 接口名 | UpperCamelCase + Service后缀 | `WorkOrderService` |
| 实现类 | 接口名 + Impl后缀 | `WorkOrderServiceImpl` |
| Action类 | 功能名 + Action后缀 | `WorkOrderListAction` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |

### 3.2 包命名

```
com.mycim.{模块}.{分层}
├── service          # 服务接口
├── service.impl     # 服务实现
├── dao              # 数据访问
├── model             # 数据模型
├── actions           # Web动作
├── forms             # 表单对象
└── util              # 工具类
```

### 3.3 方法命名

| 操作 | 前缀 | 示例 |
|------|------|------|
| 查询 | `find`, `get`, `query` | `findById()` |
| 新增 | `create`, `add`, `insert` | `createWorkOrder()` |
| 修改 | `update`, `modify` | `updateWorkOrder()` |
| 删除 | `delete`, `remove` | `deleteById()` |
| 校验 | `check`, `validate` | `checkWorkOrderTime()` |

---

## 四、新功能开发流程

### 4.1 添加新功能菜单

1. 数据库菜单表添加记录
2. 权限管理模块关联菜单与角色
3. Action开发：创建新的Action类处理请求
4. Service开发：实现业务逻辑
5. JSP页面：创建前端展示页面

### 4.2 新增Service步骤

1. 定义Service接口
2. 实现ServiceImpl类
3. 添加 `@Service` 注解
4. 在Action中注入使用

```java
@Autowired
private XxxService xxxService;

// 或使用AppContext
private XxxService xxxService = AppContext.getBean(XxxService.class);
```

### 4.3 新增定时任务

1. 创建Job类继承 `QuartzJobBean`
2. 配置Spring Bean
3. 在 `applicationContext-quartz.xml` 注册

---

## 五、调试与排错

### 5.1 日志配置

项目使用Logback进行日志管理，配置位于 `web/WEB-INF/config/logback.xml`。

```xml
<logger name="com.mycim.gc" level="DEBUG"/>
<logger name="org.springframework" level="INFO"/>
```

### 5.2 常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| 登录失败 | 数据库连接异常 | 检查数据源配置 |
| Action找不到 | Struts配置错误 | 检查struts-config.xml |
| Service注入失败 | Spring扫描问题 | 检查component-scan配置 |

---

## 六、测试

### 6.1 单元测试

测试框架位于 `core/src/service/test/` 目录。

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {
    "classpath:spring/applicationContext-service.xml"
})
public class XxxServiceTest extends AbstractServiceTests {

    @Autowired
    private XxxService xxxService;

    @Test
    public void testDoSomething() {
        Result result = xxxService.doSomething("test");
        Assert.assertNotNull(result);
    }
}
```

---

## 七、代码提交规范

### 7.1 提交信息格式

```
[模块] 简短的描述

详细的说明（可选）

修复: 相关BUG编号
```

## 相关文档

- [系统概览](../1.MES系统/00-系统概览.md) - 业务知识
- [技术架构](../1.MES系统/01-技术架构.md) - 技术栈详解
- [MES功能导航](../1.MES系统/MES功能导航.md) - 功能入口
- [开发知识库](../1.MES系统/开发知识库/) - 开发模板与异常记录
- [运维手册](../1.MES系统/运维手册/) - 环境运维
