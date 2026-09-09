# [嘉善MES 0000337]: F级wafer 在相关站点作提示

标签: 真空包
创建时间: 2022年7月25日 11:11
上次编辑时间: 2024年10月17日 10:09
状态: Not started

# 摘要

F级wafer 在相关站点作提示

# 说明

COM线会做F等级wafer，需在相关站点过站时作提示，

Route：COM_B/F_WAFER_CATE，此功能帮忙加急更新，多谢！

站点：盒包装、真空包包装、真空包外观检验

过站时提示内容：F等级wafer1

**`Due date: 2022-08-01`**

```java
//LotRepository.java
String getRouteIdByInstanceRrn(Long InstanceRrn) throws MyCimException;

//LotRepositoryImpl.java
@Override
public String getRouteIdByInstanceRrn(Long InstanceRrn) throws MyCimException {
    try {
        return getSqlBuilder().select("instance_id")
            .from("named_object").where("instance_rrn").eq(InstanceRrn)
            .queryString();
    } catch (Exception e) {
        throw ExceptionHandler.handlerException(e, logger);
    }
}

//LotService.java
String getRouteIdByInstanceRrn(Long InstanceRrn) throws MyCimException;

//LotServiceImpl.java
@Override
public String getRouteIdByInstanceRrn(Long InstanceRrn) throws MyCimException {
    try {
        return lotRepository.getRouteIdByInstanceRrn(InstanceRrn);
    } catch (Exception e) {
        throw ExceptionHandler.handlerException(e, logger);
    }
}

//PackageService.java
public String getRouteIdByBoxId(String vboxId);

//PackageServiceImpl.java
@Override
public String getRouteIdByBoxId(String vboxId) {
    PackedLot packedLot = packedLotRepository.getPackedLotByBoxId(vboxId);
    List < PackedLot > packedLotList = new ArrayList < > ();
    List < PackedLotDetail > packedLotDetailList = new ArrayList < > ();

    packedLotList = packedLotRepository.getPackedLotInfoByParentRrn(packedLot.getPackedLotRrn());

    if (!packedLotList.isEmpty()) {
        packedLotDetailList = packedLotDetailRepository.getLotInfoByPackedLotRrn(packedLotList.get(0).getPackedLotRrn());
        if (!packedLotDetailList.isEmpty()) {
            Lot lot = lotRepository.getLotById(packedLotDetailList.get(0).getLotId());
            String routeId = lotRepository.getRouteIdByInstanceRrn(lot.getProcessRrn());
            return routeId;
        }
    }
    return "none";
}

//PackageCheckoutAction.java
private ActionForward packageChexkoutNG(ActionMapping mapping, ActionForm form, HttpServletRequest request,
    ...
    try {
        ...
        List < String > vboxIdList = Arrays.asList(vboxIdInfo1.split(","));
        for (String vboxId: vboxIdList) {
            PackedLot packedLot = packageService.getPackedLotByBoxId(vboxId);
            if (packageService.getRouteIdByBoxId(vboxId).equals("COM_B/F_WAFER_CATE")) {
                map.put("alterFlag", "COM_B/F_WAFER_CATE");
            }
            packedVboxInfo.add(packedLot);
        }
        ...
    } catch (Exception e) {
        WebUtil.writeJson(response, ReponseJSONBuilder.buildErrorMsg(e));
    }
    return WebUtil.NULLActionForward;
}

private ActionForward packageChexkoutPass(ActionMapping mapping, ActionForm form, HttpServletRequest request,
    ...
    try {
        ...
        List < String > vboxIdList = Arrays.asList(vboxIdInfo1.split(","));
        for (String vboxId: vboxIdList) {
            PackedLot packedLot = packageService.getPackedLotByBoxId(vboxId);
            if (packageService.getRouteIdByBoxId(vboxId).equals("COM_B/F_WAFER_CATE")) {
                map.put("alterFlag", "COM_B/F_WAFER_CATE");
            }
            packedVboxInfo.add(packedLot);
        }
        ...
    } catch (Exception e) {
        WebUtil.writeJson(response, ReponseJSONBuilder.buildErrorMsg(e));
    }
    return WebUtil.NULLActionForward;
}

//PackageLotAction.java
private ActionForward doBoxPackageLot(ActionMapping mapping, HttpServletRequest request, Long facilityRrn,
    ...
    try {
        ...
        for (Lot lot: lotsInfo) {
            ...
            if (lotService.getRouteIdByInstanceRrn(lot.getProcessRrn()).equals("COM_B/F_WAFER_CATE")) {
                map.put("alterFlag", "COM_B/F_WAFER_CATE");
            }
        }
        ...
    } catch (Exception e) {
        WebUtil.writeJson(response, ReponseJSONBuilder.buildErrorMsg(e));
    }
    return WebUtil.NULLActionForward;
}

private ActionForward boxLoosePackage(ActionMapping mapping, HttpServletRequest request, Long facilityRrn,
    ...
    try {
        ...
        for (Lot lot: lotsInfo) {
            ...
            if (lotService.getRouteIdByInstanceRrn(lot.getProcessRrn()).equals("COM_B/F_WAFER_CATE")) {
                map.put("alterFlag", "COM_B/F_WAFER_CATE");
            }
        }
        ...
    } catch (Exception e) {
        WebUtil.writeJson(response, ReponseJSONBuilder.buildErrorMsg(e));
    }
    return WebUtil.NULLActionForward;
}

//PackageTBoxAction.java
private ActionForward packageTbox(ActionMapping mapping, HttpServletRequest request, Long facilityRrn,
    ...
    try {
        ...

        if (packageService.getRouteIdByBoxId(vboxId).equals("COM_B/F_WAFER_CATE")) {
            map.put("alterFlag", "COM_B/F_WAFER_CATE");
        }

        ...
    } catch (Exception e) {
        WebUtil.writeJson(response, ReponseJSONBuilder.buildErrorMsg(e));
    }
    return WebUtil.NULLActionForward;
}
```

```jsx
//aspectSamplingForm.js
Ext.Ajax.request({
    url: '/mycim2/packageCheckout.do?action=packageCheckoutPass', //装包检验
    params: {
        ...
    },
    success: function(response) {
        var responseJson = Ext.JSON.decode(response.responseText);
        if (responseJson.success != null && responseJson.success) {
            var alterFlag = responseJson.msg.alterFlag;
            if (alterFlag != null) {
                Ext.Msg.confirm(i18n_fld_CONFIRM, "F等级wafer", function(btn) {
                    if (btn == 'yes') {
                        passWindowClose();
                    } else {
                        passWindowClose();
                    }
                });
            }else {
								passWindowClose();
						}
        } else {
            MyCim.notify.alert(responseJson.msg);
        }
    }
});

Ext.Ajax.request({
    url: '/mycim2/packageCheckout.do?action=packageCheckoutNG', //装包检验
    params: {
        ...
    },
    success: function(response) {
        var responseJson = Ext.JSON.decode(response.responseText);
        if (responseJson.success != null && responseJson.success) {
            var alterFlag = responseJson.msg.alterFlag;
            if (alterFlag != null) {
                Ext.Msg.confirm(i18n_fld_CONFIRM, "F等级wafer", function(btn) {
                    if (btn == 'yes') {
                        nGWindowClose(ngCode);
                    } else {
                        nGWindowClose(ngCode);
                    }
                });
            }
        } else {
            MyCim.notify.alert(responseJson.msg);
        }
    }
});

function passWindowClose(){
	window.opener.store1.each(function(record){
		record.set('vboxStatus',"PQC1");
	});
	window.close();
	window.opener.store1.removeAll();
	window.opener.Ext.getCmp('packageCheckout').enable();
	window.opener.Ext.getCmp('rework').enable();
}

function nGWindowClose(ngCode){
	window.opener.store1.each(function(record){
		record.set('vboxStatus',"HOLD1");
		record.set('packageCheckComment',ngCode);
	})
	window.opener.Ext.getCmp('rework').enable();
	window.close();
}
```

```jsx
//packageLotGrid.js
if (responseJson.success != null && responseJson.success) {
    ...
    if (data.productClassify == 4) {
        ...
        var alterFlag = responseJson.msg.alterFlag;
        if (alterFlag != null) {
            Ext.Msg.confirm(i18n_fld_CONFIRM, "F等级wafer", function(btn) {
                if (btn == 'yes') {
                    bartenderPrint(finalPrintInfo, btw, 1);
                } else {
                    bartenderPrint(finalPrintInfo, btw, 1);
                }
            });
        } else {
            bartenderPrint(finalPrintInfo, btw, 1);
        }
    } else {
        ...
        bartenderPrint(finalPrintInfo, btw, 1);
    }
} else {
    MyCim.notify.alert(responseJson.msg);
}

//packageTBoxGrid.js
if (responseJson.success != null && responseJson.success) {
    ...
    var alterFlag = responseJson.msg.alterFlag;
    if (alterFlag != null) {
        Ext.Msg.confirm(i18n_fld_CONFIRM, "F等级wafer", function(btn) {
            if (btn == 'yes') {
                bartenderPrint(finalPrintInfo, btw, 1);
            } else {
                bartenderPrint(finalPrintInfo, btw, 1);
            }
        });
    } else {
        bartenderPrint(finalPrintInfo, btw, 1);
    }
    store1.removeAll();
} else {
    MyCim.notify.alert(responseJson.msg);
}
```