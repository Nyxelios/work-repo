---
title: AQL 抽样标准
date: 2024-12-13
author: MES Team
status: completed
tags: [工艺, 品质, AQL]
---

# 产品AQL设置

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241213111743.png)

根据产品型号、工步、等级获取==aql值，aql等级，aql检查状态==

```SQL
SELECT * FROM product_relations WHERE PRODUCT_ID = 'GC4023-3.5' AND BIN_GRADE IN ('TA', 'ALL');
```

AQL快捷配置生成SQL EXCEL文件：![[FQC_AOI巡检表 1.xlsx]]

# AQL查询

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241213112318.png)

根据批次数量获取NAME得到aqlSample，在根据aql等级匹配aqlSample中合适的本字。

```sql
SELECT * FROM COM_AQL_DEF cad ;
```

```java
private String getAlso(String sampleGrade, AQLSample aqlSample) {  
    String also = "";  
    if (StringUtils.equals(sampleGrade, AQLSample.EVENT_SPEC_GRADE1)) {  
        also = aqlSample.getSpecGrade1();  
    } else if (StringUtils.equals(sampleGrade, AQLSample.EVENT_SPEC_GRADE2)) {  
        also = aqlSample.getSpecGrade2();  
    } else if (StringUtils.equals(sampleGrade, AQLSample.EVENT_SPEC_GRADE3)) {  
        also = aqlSample.getSpecGrade3();  
    } else if (StringUtils.equals(sampleGrade, AQLSample.EVENT_SPEC_GRADE4)) {  
        also = aqlSample.getSpecGrade4();  
    } else if (StringUtils.equals(sampleGrade, AQLSample.EVENT_GENERAL_GRADE1)) {  
        also = aqlSample.getGeneralGrade1();  
    } else if (StringUtils.equals(sampleGrade, AQLSample.EVENT_GENERAL_GRADE2)) {  
        also = aqlSample.getGeneralGrade2();  
    } else if (StringUtils.equals(sampleGrade, AQLSample.EVENT_GENERAL_GRADE3)) {  
        also = aqlSample.getGeneralGrade3();  
    } else if (StringUtils.equals(sampleGrade, AQLSample.EVENT_GENERAL_GRADE4)) {  
        also = aqlSample.getGeneralGrade4();  
    }  
    return also;  
}
```

# AQL抽样数定义

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241213112457.png)

```sql
SELECT * FROM SAMPLE_LINE WHERE GRADE = '0.25' AND ALSO = 'N';
```

根据aql值和本字，检查状态获取接受数，拒绝数。