---
title: 定时关闭COM工单（未自动关单的工单再处理）
date: 2026-05-03
tags: [MES, 工作流]
---
![image.png](https://files-1259440452.cos.ap-nanjing.myqcloud.com/Obsidian/20260427164648797.png)

```
┌─────────────────────────────────────────────────────────────┐
│                 定时任务：COM工单自动关闭                      │
│                 每1分钟执行一次 (0 0/1 * * * ?)               │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. AutoCloseComWorkOrderJob.execute()                       │
│    ├─ 设置 ThreadLocal: username="COM_AUTO_CLOSE"           │
│    ├─ 设置 ThreadLocal: facilityRrn=1L                      │
│    ├─ 调用 workOrderService.autoCloseComStartedZeroWip...() │
│    └─ finally: 清理 ThreadLocal                              │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. WorkOrderServiceImpl.autoCloseComStartedZeroWipWorkOrders│
│    ├─ 查询: PRODUCT_CLASSIFY=COM(4)                         │
│    │         AND STATUS=STARTED                              │
│    │         AND (WIP_QTY=0 OR WIP_QTY IS NULL)             │
│    │   → 即：COM类工单 + 已投产 + 在制品数为0                  │
│    │                                                         │
│    ├─ 为每个工单执行关闭流程:                                   │
│    │   ├─ 二次查询验证条件未变 (COM, STARTED, WIP_QTY=0)      │
│    │   ├─ 设置 transType = CLOSED                            │
│    │   ├─ 开启事务日志                                        │
│    │   ├─ judgeSetWorkOrderStatus → AUTOCLOSED               │
│    │   ├─ prpSetupService.updateWorkOrder 更新工单            │
│    │   ├─ 保存 WorkOrderHistory (transType=CLOSED)           │
│    │   ├─ 标记事务日志                                        │
│    │   └─ (mesPushWmsUpdate 已注释，未推送WMS)                │
│    │                                                         │
│    └─ 打印统计: total/success/skipped/failed                 │
└─────────────────────────────────────────────────────────────┘

```

**总结**：每分钟扫描所有 COM 类（产品分类=4）且状态为 STARTED 且在制品数量为 0 的工单，二次校验后将其状态改为 AUTOCLOSED 并记录关闭历史。

要改成每三天执行一次，建议将 cron 表达式改为：

0 0 0 _/3 _ ?

各字段含义：

字段 值 含义
秒 0 第 0 秒
分 0 第 0 分
时 0 凌晨 0 点
日 _/3 每隔 3 天（1号、4号、7号...）
月 _ 每月
周 ? 不指定
修改后的配置如下：

xml
<bean id="autoCloseComWorkOrderCronTrigger" class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
<property name="jobDetail" ref="autoCloseComWorkOrderJobDetail"/>
<property name="cronExpression" value="0 0 0 */3 * ?"/>
</bean>
需要注意的一点：\*/3 是基于每月日期的，即每月 1、4、7、10、13、16、19、22、25、28、31 号执行，跨月会重置（比如 1 月 31 号之后，2 月 1 号又会执行，间隔只有 1 天）。如果你需要的是严格每隔 72 小时执行一次，cron 表达式无法完美实现，需要改用 SimpleTrigger 或在 Job 内部做间隔控制。

