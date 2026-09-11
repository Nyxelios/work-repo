---
title: COM实验工单入库数据汇总与邮件提醒定时任务
date: 2026-09-11
tags: [定时任务, 需求交付, 工单管理, 邮件提醒, 分布式锁, Done]
---

# COM实验工单入库数据汇总与邮件提醒定时任务

## 一、 需求背景与业务价值

在半导体制造与封装测试中，**COM 实验工单**（试产工单）主要用于新产品或新工艺的试制打样。业务与工艺工程部门需要密切监控试产产品的实际已入库数量：
1. **转量产评估**：当试产工单累计入库数量达到既定批次或阈值时，需及时通知跨部门团队评估是否正式转入量产（HVM - High Volume Manufacturing）。
2. **自动化提醒**：过去由工程师手工去 CIM 系统中导出各批次工单、核对入库台账，效率低且容易遗漏；现需要系统定时自动汇总并格式化发送邮件。

---

## 二、 整体设计与技术架构

```text
+-----------------------------------------------------------------------------------+
|                        WebLogic 集群 (多节点同时运行)                              |
|                                                                                   |
|  [Node A: 14:00] --抢锁成功--> [原子更新 SYS_SCHEDULE_LOCK]                       |
|         |                                 |                                       |
|         +--> 1. 读取 REFERENCE_FILE_DETAIL 动态配置 (目标型号 + 统计时间 + 收件人)  |
|         |                                                                         |
|         +--> 2. 关联查询 WORK_ORDER 入库数量 (SUM(COMPLETE_QTY))                  |
|         |                                                                         |
|         +--> 3. 组装企业级标准 HTML 表格邮件 (深蓝表头 + 居中格式)                |
|         |                                                                         |
|         +--> 4. 通过 mail.gcoreinc.com:587 稳定发送 (带自动重试与30s超时)         |
|         |                                                                         |
|         +--> 5. 释放锁并保留 30s 冷却期 (防止后续节点重复触发)                     |
|                                                                                   |
|  [Node B: 14:00] --抢锁失败--> [检测到锁尚未过期或仍在冷却期] -> 自动跳过执行     |
+-----------------------------------------------------------------------------------+
```

---

## 三、 核心实现与改动清单

### 1. 代码文件清单

| 文件路径 | 模块 | 核心改动说明 |
| :--- | :--- | :--- |
| `core/src/gc/src/main/java/com/mycim/gc/task/ComExpWorkOrderScheduleTask.java` | `gc` | **定时任务核心实现**：配置读取、入库汇总、HTML 组装、邮件发送与分布式锁协同。 |
| `core/src/gc/src/main/java/com/mycim/gc/utils/ClusterScheduleLockHelper.java` | `gc` | **集群通用分布式锁组件**：基于 `SYS_SCHEDULE_LOCK` 提供行级原子抢锁与冷却保护机制。 |
| `core/src/gc/src/main/java/com/mycim/gc/controller/SelfApiController.java` | `gc` | **管理触发端点**：提供 `triggerComExpJob(boolean force)` 便于现场免等 Cron 手工触发验证。 |

### 2. 数据库配置约定 (`REFERENCE_FILE_DETAIL`)

本功能全面采用**热配置化设计**，增删产品型号或修改收件人均无需重启系统或重新打包：

#### ① 统计型号与时间配置 (`FILE_NAME = '$$COM_EXP_WO_PRODUCT_ID'`)
* **目标型号配置**：`PARAM_NAME` 为型号代码（如 `GC08A8-MADD9-4.7`），`ACTIVE_FLAG = 'Y'`。
* **时间窗口配置**（可选）：
  * `PARAM_NAME = 'startTime'`，`VALUE1 = '2024/01/01'`：仅统计此日期之后的入库数据。
  * `PARAM_NAME = 'endTime'`，`VALUE1 = '2026/12/31'`：仅统计此日期之前的入库数据。
  * 若未配置时间或格式不匹配，系统自动 fallback 为不限时间范围。

#### ② 收件人邮箱配置 (`FILE_NAME = '$$COM_EXP_WO_EMAIL'`)
* `ACTIVE_FLAG = 'Y'` 的记录中的 `VALUE1` 列配置收件人邮箱。
* 支持逗号 `,` 或分号 `;` 分隔配置多个邮箱，系统自动去重与过滤空白格式。

---

## 四、 邮件模板规范

邮件正文遵循公司企业级邮件规范，采用全内联 CSS 渲染，保证各类邮件客户端（Outlook、Foxmail、网页端、移动端）展示完全一致：

* **主题**：`COM新产品试产已入库数量提醒`
* **问候语**：
  > Dear All,  
  > COM新产品已入库数量统计如下，请评估是否转量产！！！
* **表格样式**：
  * 表头：深蓝色背景（`#1F4E79`），白色加粗居中文字。
  * 表体：边框贴合（`border-collapse: collapse; border: 1px solid #000000;`），产品型号居左，入库数量居中展示。

---

## 五、 部署与生效验证

1. **打包与部署**：
   ```bash
   # 仅编译打包 gc 模块 (约 3 秒)
   ant -buildfile build.xml jar.gc
   # 同步到 WebLogic 自动部署目录
   copy /Y stage\gc.jar D:\data\programs\Oracle\Middleware\user_projects\domains\base_domain\autodeploy\mycim2\WEB-INF\lib\gc.jar
   ```
2. **测试与验证入口**：
   * **自动执行**：每 2 分钟（或配置的 Cron）自动抢锁触发一次。
   * **日志核验**：日志统一输出在 `logs/ap/ComExpWorkOrderScheduleTask-yyyy-MM-dd.log`。
   * **管理端强行触发**：调用 `/mycim2/self/triggerComExpJob?force=true` 可立即跳过冷却期执行。
