---
title: SQL
date: 2026-05-03
tags: [MES, 运维]
---

# 过站批次



# 定时任务
- QRTZ_CRON_TRIGGERS：定时任务触发器名
- QRTZ_TRIGGERS：触发器执行明细

> [!NOTE] Title
> 删除定时任务，可以删除两表的对应的触发器记录行

```sql
SELECT * FROM QRTZ_CRON_TRIGGERS qct ;

SELECT * FROM QRTZ_TRIGGERS qt ;
```

# 工单
## wafer排料
```
SELECT * FROM WAFER_REGEX_CONFIG wrc ;
```

# 辅料
## 物料库存
```
SELECT * FROM LOT_INVENTORY li WHERE lot_number = 'C2560169290';
```

## BOM
```
SELECT * FROM BILL_OF_RESOURCE bor WHERE BOR_RRN = '579668361' AND RESOURCE_RRN = '247784182';
```





