---
title: RW入库尾箱
date: 2025-08-15
author: MES Team
status: completed
tags: [制造管 RW, 功能]
---

![](../../附件/Pasted%20image%2020250815170000.png)

# 查询
## 1.查询符合条件的箱
```java
@Override  
public List<PackedLot> getRwPackedLotGroup(PackedLot packedLot) {  
    String sql = " SELECT\n" +  
            "\t*\n" +  
            "FROM\n" +  
            "\tMM_PACKED_LOT mpl\n" +  
            "WHERE\n" +  
            "\tPACKED_STATUS IN ('IN', 'CREATED', 'CHECKED')\n" +  
            "\tAND INNER_ORDER_NO IS NOT NULL\n" +  
            "\tAND PRODUCT_Category = 'RW'\n" +  
            "\tAND TYPE = 'RWCSTB'\n" +  
            "\tAND mpl.GRADE IN ('HA', 'MA', 'ZA', 'TA')\n" +  
            "\tAND mpl.WAFER_QTY < 8\n";  
    if (StringUtils.isNotEmpty(packedLot.getBoxId())) {  
        sql += "\tAND mpl.BOX_ID = '" + packedLot.getBoxId() + "'\n";  
    }  
    if (StringUtils.isNotEmpty(packedLot.getProductId())) {  
        sql += "\tAND mpl.PRODUCT_ID = '" + packedLot.getProductId() + "'\n";  
    }  
    if (StringUtils.isNotEmpty(packedLot.getLevelTwoCode())) {  
        sql += "\tAND mpl.LEVEL_TWO_CODE = '" + packedLot.getLevelTwoCode() + "'\n";  
    }  
    if (StringUtils.isNotEmpty(packedLot.getGrade())) {  
        sql += "\tAND mpl.GRADE = '" + packedLot.getGrade() + "'\n";  
    }  
    if (StringUtils.isNotEmpty(packedLot.getBondedProperty())) {  
        sql += "\tAND mpl.BONDED_PROPERTY = '" + packedLot.getBondedProperty() + "'\n";  
    }  
    if (StringUtils.isNotEmpty(packedLot.getEngineeringNotes())) {  
        sql += "\tAND mpl.ENGINEERING_NOTES = '" + packedLot.getEngineeringNotes() + "'\n";  
    }  
    if (StringUtils.isNotEmpty(packedLot.getInnerOrderNo())) {  
        sql += "\tAND mpl.INNER_ORDER_NO = '" + packedLot.getInnerOrderNo() + "'\n";  
    }  
    sql = sql + "\tORDER BY mpl.PRODUCT_ID, mpl.LEVEL_TWO_CODE, mpl.GRADE, mpl.BONDED_PROPERTY, mpl.ENGINEERING_NOTES, mpl.INNER_ORDER_NO ";  
    return getSqlBuilder().useSql(sql).queryList(PackedLot.class);  
}
```

## 2.根据箱号查询库位配置、流程号、是否满

库位配置表：CP_LOCATION_CONF
根据内批号：查询流程
满片逻辑：查询表RW_WTW_CONFIG，参数是包装表的原型号和流程
```java
//WTW基数  
List<RwWtwConfig> wtwConfigs = gcService.getRwWtwConfigList(packedLots.get(i).getOrgProductId(),  
        lot.getProcessId());  
if (CollectionUtils.isEmpty(wtwConfigs)) {  
    throw new MyCimException(  
            GcExceptions.GC_CHECK_RW_WTW_STATION_CONFIGURATION + lot.getLotId());  
}  
//是否满片  
double max = Integer.parseInt(wtwConfigs.get(0).getWtwCardinality()) * 0.8;  
packedLots.get(i).setFullMark("Y");  
for (PackedLot value : packedLotList) {  
    if (value.getQuantity() < max) {  
        packedLots.get(i).setFullMark("N");  
        break;  
    }  
}
```
## 