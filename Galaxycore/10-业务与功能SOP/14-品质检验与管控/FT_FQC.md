---
title: FT FQC 包装
date: 2024-11-04
author: MES Team
status: completed
tags: [包装, FT, 功能]
---

# 结束:FQC真空包AQLHold降级变更

## 描述

>FQC抽检真空包因AQL超标Hold真空包需要将缺陷物料从真空包信息中分出，现状是从小盒信息中分出，易出现异常。变更为从真空包信息中分出
>fqcTrackOut

## 现状

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241104170056.png)

## 变更

> 不显示小盒，等级显示MA，F，数99
> 就是hold的单纯是真空包里的MA部分数量变F就行
> 然后FT真空包补降级不能搜出hold的真空包就行，不会hold的还是和原先一样分离出小盒

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241104170206.png)

## 编码

> [!NOTE] 记录是否AQL hold的标 
> 先判断是否会aqlhold，对于不hold的保持原逻辑，hold的批次，将降级的数据从真空包中分离，而不是小

```java
//记录是否AQL hold的标 
boolean aqlFlag = false;  
Integer totalQtytemp = 0;  
Integer totalDegradeQtytemp = 0;  
List<String> tboxLotIdList = Lists.newArrayList();  
for (PackedLotDetail packageLotDetial : packedLotDetialList) {  
    if (!tboxLotIdList.contains(packageLotDetial.getBoxId())) {  
        tboxLotIdList.add(packageLotDetial.getBoxId());  
    }  
  
    Integer lackQtytemp = packageLotDetial.getLackQty() == null ? 0 : packageLotDetial.getLackQty();  
    Integer degradeQtytemp = packageLotDetial.getDegradedQty() == null ? 0 : packageLotDetial.getDegradedQty();  
    Integer counttemp = lackQtytemp + degradeQtytemp;  
    totalDegradeQtytemp += degradeQtytemp;  
    totalQtytemp += counttemp;  
    if (totalQtytemp > 0) {  
        // 如果降级数量大于拒收数量，将批次Hold  
        if (rejectQty != 0 && rejectQty != null && totalDegradeQtytemp >= rejectQty) {  
            aqlFlag = true;  
        }  
    }  
}
```

1. lot变更为vboxid，其余不变，注意数量的变

```java
if (aqlFlag) {  
    for (PackedLotDetail packageLotDetial : packedLotDetialList) {  
        String tboxId = packageLotDetial.getBoxId();  
        String lotId = packageLotDetial.getLotId();  
        Integer lackQty = packageLotDetial.getLackQty() == null ? 0 : packageLotDetial.getLackQty();  
        Integer degradeQty =  
                packageLotDetial.getDegradedQty() == null ? 0 : packageLotDetial.getDegradedQty();  
        Integer count = lackQty + degradeQty;  
        totalDegradeQty += degradeQty;  
        totalQty += count;  
        Lot lot = lotService.getLot(vboxId);  
        LotDieBin mainLotDieBin = lotService.getLotDieBinByLotRrnAndBinId(lot.getLotRrn(), grade);  
        if (null == mainLotDieBin) {  
            mainLotDieBin = lotService.getLotDieBinByLotRrnAndBinId(lot.getLotRrn(), grade + "*");  
        }  
        if (null == mainLotDieBin) {  
            throw new MyCimParameterException(vboxId + "对应等级数据为空,请联系管理员");  
        }  
        if (lackQty > 0) {  
            // 验证缺陷等级是否已经存在，如果存在修改批次未包装数量，不存在则新 
            // 批次备注  
            reduceQty = reduceQty + lackQty;  
            LotDieBin lotDieBin =  
                    lotDieBinRepository.getByLotRrnAndBinId(lot.getLotRrn(), PackedLot.QS_GRADE);  
            if (lotDieBin != null) {  
                lotDieBin.setUnpackedQty(lotDieBin.getUnpackedQty() + lackQty.doubleValue());  
                lotDieBin.setQty(lotDieBin.getQty() + lackQty);  
                lotDieBin.setStorageRemark("1");  
                lotDieBin.setTranscationTime(new Date());  
                lotDieBin.setOwner(ThreadLocalContext.getUsername());  
                lotDieBinRepository.merge(lotDieBin);  
            } else {  
                lotDieBin = new LotDieBin();  
                PropertyUtils.copyProperties(lotDieBin, mainLotDieBin);  
                lotDieBin.setBinId(PackedLot.QS_GRADE);  
                lotDieBin.setStorageRemark("1");  
                lotDieBin.setQty(lackQty.longValue());  
                lotDieBin.setUnpackedQty(lackQty.doubleValue());  
                lotDieBin.setWltBit("");  
                lotDieBin.setBinDesc(null);  
                lotDieBin.setBinGroupId(null);  
                lotDieBin.setBinType(null);  
                lotDieBin.setTranscationTime(new Date());  
                lotDieBin.setOwner(ThreadLocalContext.getUsername());  
                lotDieBin.setTransRrn(transactionLog.getTransRrn());  
                lotDieBin.setObjectRrn(getObjectRRN());  
                lotDieBinRepository.save(lotDieBin);  
            }  
            lotDieBinHis = new LotDieBinHis(lotDieBin);  
            lotDieBinHis.setObjectRrn(getObjectRRN());  
            lotDieBinHis.setTransRrn(transactionLog.getTransRrn());  
            lotDieBinHisRepository.save(lotDieBinHis);  
        }  
        if (degradeQty > 0) {  
            // 批次备注  
            reduceQty = reduceQty + degradeQty;  
            List<FtFqcPackageDefect> packageDefectList =  
                    ftFqcPackageDefectRepository.findByVboxIdAndTboxIdAndLotId(vboxId, tboxId, lotId);  
            Map<String, List<FtFqcPackageDefect>> defectMap = Maps.newHashMap();  
            for (FtFqcPackageDefect packageDefect : packageDefectList) {  
                if (defectMap.containsKey(packageDefect.getBinGrade())) {  
                    defectMap.get(packageDefect.getBinGrade()).add(packageDefect);  
                } else {  
                    List<FtFqcPackageDefect> defectList = Lists.newArrayList();  
                    defectList.add(packageDefect);  
                    defectMap.put(packageDefect.getBinGrade(), defectList);  
                }  
            }  
  
            for (String binGrade : defectMap.keySet()) {  
                List<FtFqcPackageDefect> fqcPackageDefectList = defectMap.get(binGrade);  
                Integer degradedQty = 0;  
                for (FtFqcPackageDefect defect : fqcPackageDefectList) {  
                    degradedQty += defect.getQty();  
                }  
                LotDieBin lotDieBin =  
                        lotDieBinRepository.getByLotRrnAndBinId(lot.getLotRrn(), binGrade);  
                if (lotDieBin != null) {  
                    lotDieBin.setUnpackedQty(lotDieBin.getUnpackedQty() + degradedQty.doubleValue());  
                    lotDieBin.setQty(lotDieBin.getQty() + degradedQty);  
                    lotDieBin.setTranscationTime(new Date());  
                    lotDieBin.setOwner(ThreadLocalContext.getUsername());  
                    lotDieBin.setStorageRemark("1");  
                    lotDieBinRepository.merge(lotDieBin);  
                } else {  
                    lotDieBin = new LotDieBin();  
                    PropertyUtils.copyProperties(lotDieBin, mainLotDieBin);  
                    lotDieBin.setWltBit("");  
                    lotDieBin.setBinId(binGrade);  
                    lotDieBin.setStorageRemark("1");  
                    lotDieBin.setQty(degradedQty.longValue());  
                    lotDieBin.setUnpackedQty(degradedQty.doubleValue());  
                    lotDieBin.setBinDesc(null);  
                    lotDieBin.setBinGroupId(null);  
                    lotDieBin.setBinType(null);  
                    lotDieBin.setTranscationTime(new Date());  
                    lotDieBin.setOwner(ThreadLocalContext.getUsername());  
                    lotDieBin.setTransRrn(transactionLog.getTransRrn());  
                    lotDieBin.setObjectRrn(getObjectRRN());  
                    lotDieBinRepository.save(lotDieBin);  
                }  
                lotDieBinHis = new LotDieBinHis(lotDieBin);  
                lotDieBinHis.setObjectRrn(getObjectRRN());  
                lotDieBinHis.setTransRrn(transactionLog.getTransRrn());  
                lotDieBinHisRepository.save(lotDieBinHis);  
            }  
            // 记录缺陷历史  
            for (FtFqcPackageDefect defect : packageDefectList) {  
                DefectBinRelationHis defectBinRelationHis = new DefectBinRelationHis();  
                defectBinRelationHis.setBinGroupRrn(defect.getBinGroupRrn());  
                defectBinRelationHis.setBinRrn(defect.getBinRrn());  
                defectBinRelationHis.setBinName(defect.getBinGrade());  
                defectBinRelationHis.setDefectCode(defect.getDefectCode());  
                defectBinRelationHis.setDefectDesc(defect.getDefectDesc());  
                defectBinRelationHis.setDefectType(defect.getDefectType());  
                defectBinRelationHis.setQty(defect.getQty().toString());  
                defectBinRelationHis.setObjectRrn(getObjectRRN());  
                defectBinRelationHis.setOperationRrn(operationRrn);  
                defectBinRelationHis.setLotRrn(lot.getLotRrn());  
                defectBinRelationHis.setFacilityRrn(ThreadLocalContext.getFacilityRrn());  
                defectBinRelationHis.setOwner(ThreadLocalContext.getUsername());  
                defectBinRelationHis.setTransRrn(transactionLog.getTransRrn());  
                defectBinRelationHis.setTransType(transactionLog.getTransId());  
                defectBinRelationHis.setFromBinName(grade);  
                prpSetupService.saveDefectBinRelationHis(defectBinRelationHis);  
  
                ftFqcPackageDefectRepository.deleteOneById(defect.getObjectRrn());  
            }  
        }  
    }  
}
```

## 等级转换配置

```java
/**  
 * 验证批次在IQC站是否存在等级转换配置，存在则修改等级信 
 *  
 * @param lot  
 */  
private void replaceGradeForTiqc(Lot lot) {  
    try {  
        lot.setLotRemarks("");  
        String operationId = baseService.getNamedObjectId(lot.getOperationRrn());  
        if (StringUtils.equals(Lot.IQC_FUNCTION_TEST, operationId)) {  
            String productId = baseService.getNamedObjectId(lot.getProductRrn());  
            String processId = baseService.getNamedObjectId(lot.getProcessRrn());  
            List<LotDieBin> lotDieBinInfos = lotService.getLotDieBinList(lot.getLotRrn());  
            for (LotDieBin lotDieBin : lotDieBinInfos) {  
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
                                    break;  
                                }  
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
    } catch (Exception e) {  
        throw ExceptionHandler.handlerException(e, log);  
    }  
}
```

# 修改页面的排版，修改时间

## 任务的开

> [!NOTE]
> 💡 号，王欣琪离职了，李智威给我分配了一个任务，初看感觉比较简单，是对前端页面的排版重新搞下
> ![](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241021160613.png)
> ![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241021160632.png)

## DO

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241021160818.png)

从发给我的图知道，需要修改开始界面的排版，于是我在浏览器开启F12调试，然后锁定到这个页面的名字信息，之后就在项目中修改代码

💡 可以知道，开始界面的文件名是包含start的，同时要修改的是表格的列名，那就是在grid.js文件中。即`ftFqcStartGrid.js`

### 代码修改

```jsx
......
columns: [{
            xtype: 'rownumberer',
            id: 'startSeq',
            align: 'center',
            flex: 1,
            sortable: false,
            header: i18n_fld_SEQ
          },{
			xtype : 'gridcolumn',
			dataIndex : 'workorderId',
			width:160,
			align : 'center',
			sortable : true,
			header : i18n_msg_WorkOrderId,
			hidden: true,
		},{
			xtype : 'gridcolumn',
			dataIndex : 'productId',
			align : 'center',
			width:160,
			sortable : true,
			header : i18n_msg_productModel
		},{
			xtype : 'gridcolumn',
			dataIndex : 'boxId',
			width:160,
			align : 'center',
			sortable : true,
			header : i18n_fld_Vbox_Id
		},{
			xtype : 'gridcolumn',
            dataIndex : 'levelTwoCode',
            align : 'center',
            width:100,
			sortable : true,
			header : i18n_fld_SecondCode
		},{
			xtype : 'gridcolumn',
			dataIndex : 'grade',
			align : 'center',
			width:80,
			sortable : true,
			header :i18n_fld_grade
		}, {
			xtype : 'gridcolumn',
			dataIndex : 'quantity',
			align : 'center',
			width:80,
			sortable : true,
			header : i18n_msg_Count
		},{
			xtype : 'gridcolumn',
			dataIndex : 'finalOperationTime',
			align : 'center',
			width:160,
			sortable : true,
			header :i18n_fld_finalOperatorTime
		},{
			xtype : 'gridcolumn',
			dataIndex : 'treasuryNote',
			align : 'center',
			width:160,
			sortable : true,
			header :i18n_fld_Ship_Note
		},{
			xtype : 'gridcolumn',
			dataIndex : 'processId',
			align : 'center',
			width:160,
			sortable : true,
			header : i18n_msg_Process_ID
		},
......
......
]
```

 💡 主要就是修改column数组的顺序即可，工单号不需要就设置hidden属性隐藏掉就🆗了

### 交付

说实话，只花了我十分钟，正当我准备交差，然后李智威告诉我其实待入组，出组，开始，结束都要修改，和开始的排版类似，同时对于最后操作的时间的格式要进行更改，格式为*`2022-06-01 1153`**

没办法，那我只能照做了

## DO 2

首先我打算把四个页面的前端的列名都搞下，这个比较简单，直接复制start的，然后删去没有的列名，修改下列名的属性就好了*`~~话说是这样，但这种简单重复的工作让我很抓狂，感觉没有意义~~`**

完事之后，遇到了比较头疼的问题，也是我之前做胶水金线遇到的，金线胶水的数据结构中，关于时间的**`数据类型是DATE`**的，这个没问题，但是传到前端的时间需要时字符串的，但是这种字符串格式并不规范，是**`"26/02/2022 12:46:02”`** 的，2022年的26日，这种格式就很奇怪，因为之前是重新新建了一个实体类，然后把日期改成了string类，**`也就是在java中做了格式转换的工作`**，不过这样做似乎没有得到王欣琪的赞同，所以这次尝试在js中修改格式

### T1

 💡 在form.js文件中添加转换的代码

```jsx
Ext.doTransDate = function dateFormat(fmt, date) {
    let ret;
    const opt = {
        "Y+": date.getFullYear().toString(),        // 
        "m+": (date.getMonth() + 1).toString(),     // 
        "d+": date.getDate().toString(),            // 
        "H+": date.getHours().toString(),           // 
        "M+": date.getMinutes().toString(),         // 
        "S+": date.getSeconds().toString()          // 
        // 有其他格式化字符需求可以继续添加，必须转化成字符串
    };
    for (let k in opt) {
        ret = new RegExp("(" + k + ")").exec(fmt);
        if (ret) {
            fmt = fmt.replace(ret[1], (ret[1].length == 1) ? (opt[k]) : (opt[k].padStart(ret[1].length, "0")))
        };
    };
    return fmt;
}
```

这样做确实是可以但是，因为这个日期格式太奇怪了，把**`2021/12/9当成021/9/12`**

这个问题就很严重，没办法，那我只能试试java中的格式修改下试试了

💡 等等，既然他月份和日期交换了位置，那我直接在`"yyyy-MM-dd HH:mm:ss"` 修改不就行了嘛？`"yyyy-dd-MM HH:mm:ss"`

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241021160948.png)

好吧，还是不行，只是最后的步骤位置交换，但他转不了date类型就没有后续了。还是从java中改改吧

### T2

准备在实体类加注解`@JsonFormat(pattern="yyyy-MM-dd HH:mm:ss",timezone="GMT+8")`

结果还是失败

<aside> 💡 意外发现，这里是由字符串类型的，原来只需要加一个字段就行了。学到了。我以为加一下会影响数据库的映射

</aside>
![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241021161027.png)

```jsx
var data = responseJson.msg.data;
for (i = 0; i < data.length; i++) { 
    data[i].finalOperationTime = data[i].stringFinalOperationTime;//加的代码
}
startStore.add(data);
```

```java
//在查询方法中，加入如下代码，将date转字符串，给stringfinal赋值即可
SimpleDateFormat simpleDateFormat1 = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
String stringFinalOperationTime = simpleDateFormat1.format(packedLot.getFinalOperationTime());
packedLot.setStringFinalOperationTime(stringFinalOperationTime);
```

**`终于解决了一个困扰的问题😋`**

## 后期

又增加了两个任务，不过时间不急，可以慢慢来

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241021161059.png)

## Add Q

> 新增了一个小需求，在出组的表格增加一个最后操作的时间

![image.png](https://picblog.oss-cn-hangzhou.aliyuncs.com/obsidian/20241021161115.png)

```java
private JSONObject outGroupLotsToJsonObject(Lot lot, String binId, Long lotDieBinQty) {
		/*
		......
		*/
		PackedLot packedLot = packageService.getPackedVBoxInfoByVboxId(lot.getLotId());
		String stringFinalOperationTime = null;
		if(packedLot != null){
			treasuryNote = packedLot.getTreasuryNote();
			/**添加最后操作时*/
			SimpleDateFormat simpleDateFormat1 = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
			stringFinalOperationTime = simpleDateFormat1.format(packedLot.getFinalOperationTime());
			lot.setStringFinalOperationTime(stringFinalOperationTime);
		}
		packedLotJson.put("stringFinalOperationTime", stringFinalOperationTime);
		//省略
		return packedLotJson;
	}
```

<aside> 💡 在封装成json的那一步，put增加一个字段，传入最后操作的时间，key对应前端的grid的属性值

</aside>

```jsx
Ext.define('FtFqcOutGroupModel',{
	extend:'Ext.data.Model',
	fields: [
	//省略
	{
	  name: 'stringFinalOperationTime',
	}]
})

//------
columns: [
		//省略
		{
			xtype : 'gridcolumn',
			dataIndex : 'stringFinalOperationTime',
			align : 'center',
			width:160,
			sortable : true,
			header :i18n_fld_finalOperatorTime
		}
```








