---
title: FT IQC功能测试等级转换配置
date: 2024-11-04
author: MES Team
status: completed
tags: [工艺, FT, 功能]
---

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241104173202.png)

等级出站时转换，如果没有配置，正常转
如果已有WA，则增加新的HA等级，若已有HA，则数量增加到现有HA

```java
/**  
 * 验证批次在IQC站是否存在等级转换配置，存在则修改等级信 
 *  
 * @param lot  
 */  
private void replaceGradeForTiqc(Lot lot) {  

	List<GcGradeSwitchConfig> configList = gcGradeSwitchConfigRepository.getGcGradeSwitchConfigList(productId, processId, lotDieBin.getBinId(), "");  
    if (CollectionUtils.isNotEmpty(configList)) {  
        for (GcGradeSwitchConfig config : configList) {  
            if (lot.getLevelTwoCode().length() <= 4) {  
                throw new MyCimException(WipExceptions.LEVEL_TWO_CODE_LENGTH_MUST_BE_GREATER_THAN_4_DIGITS);  
            }  
            String fiveCode = lot.getLevelTwoCode().substring(4, 5);  
            if (StringUtils.equals(config.getFiveLevelCode(), fiveCode)) {  
                TransactionLog transactionLog = baseService.startTransactionLog(ThreadLocalContext.getUsername(), LotDieBin.UPDATE_GRADE);  
                boolean flag = true;  
                for (LotDieBin bin : lotDieBinInfos) {  
                    if (StringUtils.equals(bin.getBinId(), config.getNewGrade())) {  
                        bin.setQty(bin.getQty() + lotDieBin.getQty());  
                        lotDieBin.setQty(0L);  
                        bin.setUnpackedQty(bin.getUnpackedQty() + lotDieBin.getUnpackedQty());  
                        lotDieBin.setUnpackedQty(0d);  
                        lotService.updateLotDieBin(bin);  
                        bin.setTranscationTime(new Date());  
                        bin.setTransRrn(transactionLog.getTransRrn());  
                        lotService.saveLotDieBinHis(bin);  
                        flag = false;  
                        break;                    }  
                }  
                if (flag) {  
                    lotDieBin.setBinId(config.getNewGrade());  
                }  
                lotService.updateLotDieBin(lotDieBin);  
                lotDieBin.setTranscationTime(new Date());  
                lotDieBin.setTransRrn(transactionLog.getTransRrn());  
                lotService.saveLotDieBinHis(lotDieBin);  
                lot.setTransId(transactionLog.getTransId());  
                LotTransHistory lotTransHistory = new LotTransHistory(transactionLog.getTransRrn(), new Long(1), lot);  
                lotService.saveLotTransHistory(lotTransHistory);  
                baseService.markTransactionLog(transactionLog);  
            }  
        }  
  
    }  
}  
for (LotDieBin lotDieBin : lotDieBinInfos) {  
    if (lotDieBin.getQty() <= 0) {  
        lotService.deleteLotDieBin(lotDieBin.getObjectRrn());  
    }else {  
        lotService.updateLotDieBin(lotDieBin);  
    }  
}

}
```