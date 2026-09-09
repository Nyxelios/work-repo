---
title: RW入库-修改中间表RW_ENG_LOT_INFORM
date: 2024-07-15
tags: [MES, RW, 入库]
---
# RW入库-修改中间表RW_ENG_LOT_INFORM

## 0. 可行性分析

![[RW入库修改中间表-Untitled.png]]

需要在入库的时候，修改RW_ENG_LOT_INFORM表的信息，可以实现。

## 1. 逻辑实现

```jsx
@Override
    public List<Map> getRwEngLotInform(String lotId, String grade) throws MyCimException {
        try {
            if (StringUtils.isEmpty(lotId) || StringUtils.isEmpty(grade)) {
                return new ArrayList<>();
            }
            //获取当前运行环境
            Resource resource = new ClassPathResource("application.properties");
            Properties props = PropertiesLoaderUtils.loadProperties(resource);
            String active = props.getProperty("spring.profiles.active");
            String user = "EIFPROD";
            if (active.equals("development")){
                user = "EIFTEST";
            }
            String sql = "SELECT T.CST_ID, T.QTY, T.PRODUCT_ID FROM " + user + ".RW_ENG_LOT_INFORM T WHERE T.LOT_ID ='" + lotId + "' AND T.GRADE = '" + grade + "'";
            List<Map> datas = iRepository.useSqlQuery(sql, Map.class);
            return datas;
        } catch (Exception se) {
            throw ExceptionHandler.handlerException(se, logger);
        }
    }

    @Override
    public void updateRwEngLotInform(String s, String binType, String cstId, String qtyStr, String productId) throws MyCimException {
        try {
            //获取当前运行环境
            Resource resource = new ClassPathResource("application.properties");
            Properties props = PropertiesLoaderUtils.loadProperties(resource);
            String active = props.getProperty("spring.profiles.active");
            String user = "EIFPROD";
            if (active.equals("development")){
                user = "EIFTEST";
            }
            String sql = " UPDATE " + user + ".RW_ENG_LOT_INFORM T " + " SET T.CST_ID = ?, T.QTY = ?, T.PRODUCT_ID = ? WHERE T.LOT_ID = ? AND T.GRADE = ?";

            Object[] args = new Object[] { cstId, qtyStr, productId, s, binType};
            int a = iRepository.useSqlUpdate(sql.toString(), args);
        } catch (Exception se) {
            throw ExceptionHandler.handlerException(se, logger);
        }
    }
```

```jsx
//入库根据内批和等级修改中间表RW_ENG_LOT_INFORM信息
List < Map > maps = lotService.getRwEngLotInform(packedLotInfo.getLotId().split("\\.")[0], packedLotInfo.getBinType());
if (!maps.isEmpty()) {
    Map m = maps.get(0);
    String cstId = (String) m.get("CST_ID");
    if (StringUtils.isNotEmpty(cstId) && !cstId.contains(packedLot.getCstId())) {
        cstId = cstId + "," + packedLot.getCstId();
    } else if (StringUtils.isEmpty(cstId)) {
        cstId = packedLot.getCstId();
    }
    String qtyStr = (String) m.get("QTY");
    if (StringUtils.isNotEmpty(qtyStr)) {
        qtyStr = String.valueOf(Integer.parseInt(qtyStr) + packedLotInfo.getQuantity());
    } else if (StringUtils.isEmpty(qtyStr)) {
        qtyStr = String.valueOf(packedLotInfo.getQuantity());
    }
    String productId = (String) m.get("PRODUCT_ID");
    if (StringUtils.isNotEmpty(productId) && !productId.contains(packedLotInfo.getProductId())) {
        productId = productId + "," + packedLotInfo.getProductId();
    } else if (StringUtils.isEmpty(productId)) {
        productId = packedLotInfo.getProductId();
    }
    lotService.updateRwEngLotInform(packedLotInfo.getLotId().split("\\.")[0],
        packedLotInfo.getBinType(), cstId, qtyStr, productId);
}
```

## 2.功能测试

[MES_0005045_RW 入库根据内批和等级修改中间表 RW_ENG_LOT_INFORM 信息.xlsx](RW%20%E5%85%A5%E5%BA%93%E6%A0%B9%E6%8D%AE%E5%86%85%E6%89%B9%E5%92%8C%E7%AD%89%E7%BA%A7%E4%BF%AE%E6%94%B9%E4%B8%AD%E9%97%B4%E8%A1%A8%20RW_ENG_LOT_INFORM%20%E4%BF%A1%E6%81%AF%2005de-4cd5/MES_0005045_RW_%25E5%2585%25A5%25E5%25BA%2593%25E6%25A0%25B9%25E6%258D%25AE%25E5%2586%2585%25E6%2589%25B9%25E5%2592%258C%25E7%25AD%2589%25E7%25BA%25A7%25E4%25BF%25AE%25E6%2594%25B9%25E4%25B8%25AD%25E9%2597%25B4%25E8%25A1%25A8_RW_ENG_LOT_INFORM_%25E4%25BF%25A1%25E6%2581%25AF.xlsx)

## 3.结语

重点是入库时修改的sql语句，这个表是另一个oracle用户下的，所以需要授予权限或者sql加上用户名前缀。