# Galaxycore MES API参考文档

## 目录

- [一、核心服务接口](#一核心服务接口)
- [二、Web Action接口](#二web-action接口)
- [三、Repository接口](#三repository接口)
- [四、工具类](#四工具类)
- [五、Spring配置](#五spring配置)
- [六、异常处理](#六异常处理)

---

## 一、核心服务接口

### 1.1 WorkOrderService（工单服务）

**包路径**：`com.mycim.gc.service.WorkOrderService`

#### 接口定义

```java
public interface WorkOrderService {
    
    ReferenceFileDetail getReferenceFileValue(String className, String FileKey1);
    String getReferenceFileValue1(String className, String FileKey1);
    void checkWorkorderTime(WorkOrder workOrder);
    String getWorkOrderId(String workOrderType, String bondedProperty, String productClassify);
    void generateWorkOrder(WorkOrder workOrder, WorkOrder workOrderForm);
    List<GcSapProductInfo> queryByLineType(String lineType);
    Item checkMaterialName(String materialName, String waferSource);
    ResultPage<WorkOrderRelation> getWorkOrderRelationByWorkOrderRrn(Long workOrderRrn, int page, int pageSize);
    WorkOrder mergeWorkOrder(WorkOrder workOrder, WorkOrder workOrderForm);
    void autoCloseComStartedZeroWipWorkOrders();
    void mesPushWmsAdd(WorkOrder workOrder);
    void mesPushWmsUpdate(WorkOrder workOrder);
    WorkOrder createWorkOrder(WorkOrder workOrder);
    void addWorkOrderRelation(WorkOrderRelation workOrderRelation, Double num, WorkOrder workOrder);
    void deleteWorkOrderRelation(WorkOrderRelation workOrderRelation, Double num, WorkOrder workOrder);
    void batchCancelWorkOrderPlan(WorkOrder workOrder) throws MyCimException;
    List<BoomInfo> getBoomList(BoomInfo boom);
}
```

#### 使用示例

```java
@Service
public class WorkOrderBusinessService {
    
    @Autowired
    private WorkOrderService workOrderService;
    
    public void createNewWorkOrder(WorkOrderForm form) {
        String workOrderId = workOrderService.getWorkOrderId(
            form.getWorkOrderType(),
            form.getBondedProperty(),
            form.getProductClassify()
        );
        
        WorkOrder workOrder = new WorkOrder();
        workOrder.setWorkOrderId(workOrderId);
        workOrderService.createWorkOrder(workOrder);
    }
}
```

---

### 1.2 LotService（批次服务）

**包路径**：`com.mycim.service.wip.LotService`

| 方法 | 说明 |
|------|------|
| `getLot()` | 获取批次 |
| `getLotList()` | 获取批次列表 |
| `createLot()` | 创建批次 |
| `holdLot()` | 冻结批次 |
| `releaseLot()` | 释放批次 |

---

### 1.3 WorkflowExecuteService（工作流执行服务）

**包路径**：`com.mycim.gc.service.WorkflowExecuteService`

```java
public interface WorkflowExecuteService {
    void executeWorkflow(String workflowId, String lotId, Map<String, Object> params);
    void executeStep(String lotId, String stepId, Map<String, Object> stepData);
    List<WorkflowTask> getPendingTasks(String lotId);
}
```

---

## 二、Web Action接口

### 2.1 PrpSetupAction（基础Action）

**包路径**：`com.mycim.webapp.actions.common.PrpSetupAction`

所有业务Action的基类，提供通用的请求处理能力。

```java
public abstract class PrpSetupAction extends Action {
    
    public ActionForward perform(ActionMapping mapping, ActionForm form,
                                 HttpServletRequest request, HttpServletResponse response);
    
    protected UserProfile getCurrentUser();
    protected String getCurrentFactory();
    protected ActionForward successForward(ActionMapping mapping);
    protected ActionForward errorForward(ActionMapping mapping, String message);
}
```

### 2.2 WorkOrderListAction

| action值 | 说明 |
|----------|------|
| `list` | 查询工单列表 |
| `detail` | 查看工单详情 |
| `export` | 导出工单数据 |

### 2.3 LotTrackAction

| action值 | 说明 |
|----------|------|
| `moveIn` | 进站 |
| `moveOut` | 出站 |
| `hold` | 冻结批次 |
| `release` | 释放批次 |

---

## 三、Repository接口

### 3.1 WorkOrderRepository

```java
@Repository
public interface WorkOrderRepository extends JpaRepository<WorkOrder, Long> {
    WorkOrder findByWorkOrderId(String workOrderId);
    List<WorkOrder> findByStatus(String status);
    Page<WorkOrder> findByStatus(String status, Pageable pageable);
}
```

### 3.2 LotRepository

```java
@Repository
public interface LotRepository extends JpaRepository<Lot, Long> {
    Lot findByLotId(String lotId);
    List<Lot> findByWorkOrderRrn(Long workOrderRrn);
    List<Lot> findByStatusAndFactoryId(String status, String factoryId);
}
```

---

## 四、工具类

### 4.1 AppContext（应用上下文）

**包路径**：`com.mycim.AppContext`

```java
public class AppContext {
    public static <T> T getBean(Class<T> clazz) {
        return SpringContextHolder.getBean(clazz);
    }
}
```

### 4.2 BaseService（基础服务）

**包路径**：`com.mycim.bas.service.BaseService`

| 方法 | 说明 |
|------|------|
| `save(Object)` | 保存对象 |
| `update(Object)` | 更新对象 |
| `delete(Object)` | 删除对象 |
| `executeQuery(sql)` | 执行SQL查询 |
| `executeUpdate(sql)` | 执行更新 |

---

## 五、Spring配置

### 5.1 事务配置

**文件**：`web/WEB-INF/config/spring/applicationContext-service.xml`

```xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="save*" rollback-for="Exception" propagation="REQUIRED"/>
        <tx:method name="insert*" rollback-for="Exception" propagation="REQUIRED"/>
        <tx:method name="delete*" rollback-for="Exception" propagation="REQUIRED"/>
        <tx:method name="update*" rollback-for="Exception" propagation="REQUIRED"/>
        <tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
        <tx:method name="query*" propagation="SUPPORTS" read-only="true"/>
    </tx:attributes>
</tx:advice>
```

---

## 六、异常处理

### 6.1 异常体系

```
MyCimException (基础异常)
    ├── MyCimParameterException (参数异常)
    ├── MyCimBusinessException (业务异常)
    │   ├── PrpExceptions (产品异常)
    │   └── WipExceptions (WIP异常)
    └── MyCimSystemException (系统异常)
```

### 6.2 异常使用

```java
throw new MyCimBusinessException("工单不存在: " + workOrderId);

try {
    workOrderService.createWorkOrder(workOrder);
} catch (MyCimBusinessException e) {
    log.error("业务异常: {}", e.getMessage());
    throw e;
}
```

---

## 相关文档

- [开发者指南](DEVELOPER_GUIDE.md) - 开发规范与流程
- [核心模块说明](MODULES.md) - 模块详细设计
- [数据库设计](DATABASE.md) - 数据表结构
