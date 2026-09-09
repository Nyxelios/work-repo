---
title: 梓一OTP烧录配置
date: 2024-10-21
author: MES Team
status: completed
tags: [工艺, FT, 功能]
---

# 前端

模板：[[../../../MES开模板/配置菜单模块/菜单模板1——指定参数]]

# 后端

## Action

```java
/**  
 * @ClassName ZiYiOtpConfigAction  
 * @Description: 类描 
 * @Author: natsume_wang  
 * @CreateDate: 2024/12/10 15:50  
 * @Version: 1.0  
 */
 public class FtZiYiOtpConfigAction extends PrpSetupAction {  
    private final String ACTION = "action";  
    private final String ACTION_UPDATE = "update";  
    private final String ACTION_SEARCH = "search";  
    private final String ACTION_DELETE = "delete";  
    private final String ACTION_SAVE = "save";  
    private final String ACTION_PRODUCTIDCOMBOBOX = "productIdComboBox";  
    private final String ACTION_PROCESSIDCOMBOBOX = "processIdComboBox";  
  
    public ActionForward perform(ActionMapping mapping, ActionForm form, HttpServletRequest request,  
                                 HttpServletResponse response) throws ServletException {  
        String userName = ThreadLocalContext.getUsername();  
        Long facilityRrn = ThreadLocalContext.getFacilityRrn();  
        String action = WebUtils.getParameter(ACTION, request);  
        if (userName == null) {  
            return mapping.findForward(Constants.LOGIN_KEY);  
        }  
  
        if(StringUtils.equals(action, ACTION_SEARCH)){  
            return search(request, response);  
        }else if(StringUtils.equals(action, ACTION_PRODUCTIDCOMBOBOX)){  
            return queryProductIdComboBox(request, response, Lists.newArrayList(WorkOrder.FT,WorkOrder.WLFT));  
        }else if(StringUtils.equals(action, ACTION_SAVE)){  
            return save(request, response);  
        }else if(StringUtils.equals(action, ACTION_DELETE)){  
            return delete(request, response);  
        }else if(StringUtils.equals(action, ACTION_UPDATE)){  
            return update(request, response);  
        }else if(StringUtils.equals(action, ACTION_PROCESSIDCOMBOBOX)){  
            return queryProcessIdComboBox(request, response, facilityRrn);  
        }  
  
        return mapping.findForward("modify");  
    }  
  
    private ActionForward update(HttpServletRequest request, HttpServletResponse response) {  
        Map<String, Object> map = new HashMap<>();  
        try{  
            Long objectRrn = Long.valueOf(WebUtils.getParameter("objectRrn", request));  
            String productId = WebUtils.getParameter("productId", request);  
            String processId = WebUtils.getParameter("processId", request);  
            String packageId = WebUtils.getParameter("packageId", request);  
            List<GcFtZiYiOtpConfig> gcFtZiYiOtpConfigList = ftService.getGcFtZiYiOtpConfigList(productId, processId, packageId);  
            if(CollectionUtils.isNotEmpty(gcFtZiYiOtpConfigList)){  
                throw new MyCimException(WipExceptions.THIS_RULE_SOUDN_SETTINGS);  
            }  
            GcFtZiYiOtpConfig gcFtZiYiOtpConfig = ftService.getGcFtZiYiOtpConfigByRrn(objectRrn);  
            gcFtZiYiOtpConfig.setProductId(productId);  
            gcFtZiYiOtpConfig.setProcessId(processId);  
            gcFtZiYiOtpConfig.setPackageId(packageId);  
            ftService.updateGcFtZiYiOtpConfig(gcFtZiYiOtpConfig);  
            WebUtil.writeJson(response, ReponseJSONBuilder.buildSuccessMsg(JSONUtils.toJSONString(map)));  
        }catch(Exception e){  
            WebUtil.writeJson(response, ReponseJSONBuilder.buildErrorMsg(e));  
        }  
        return WebUtils.NULLActionForward;  
    }  
  
    private ActionForward delete(HttpServletRequest request, HttpServletResponse response) {  
        Map<String, Object> map = new HashMap<>();  
        try{  
            Long objectRrn = Long.valueOf(WebUtils.getParameter("objectRrn", request));  
            GcFtZiYiOtpConfig gcFtZiYiOtpConfig = ftService.getGcFtZiYiOtpConfigByRrn(objectRrn);  
            if(gcFtZiYiOtpConfig != null){  
                ftService.deleteGcFtZiYiOtpConfig(gcFtZiYiOtpConfig);  
            }  
            WebUtil.writeJson(response, ReponseJSONBuilder.buildSuccessMsg(JSONUtils.toJSONString(map)));  
        }catch(Exception e){  
            WebUtil.writeJson(response, ReponseJSONBuilder.buildErrorMsg(e));  
        }  
        return WebUtils.NULLActionForward;  
    }  
  
    private ActionForward save(HttpServletRequest request, HttpServletResponse response) {  
        Map<String, Object> map = new HashMap<>();  
        try{  
            String productId = WebUtils.getParameter("productId", request);  
            String processId = WebUtils.getParameter("processId", request);  
            String packageId = WebUtils.getParameter("packageId", request);  
            List<GcFtZiYiOtpConfig> gcFtZiYiOtpConfigList = ftService.getGcFtZiYiOtpConfigList(productId, processId, packageId);  
            if(CollectionUtils.isNotEmpty(gcFtZiYiOtpConfigList)){  
                throw new MyCimException(WipExceptions.THIS_RULE_SOUDN_SETTINGS);  
            }  
            GcFtZiYiOtpConfig gcFtZiYiOtpConfig = new GcFtZiYiOtpConfig();  
            gcFtZiYiOtpConfig.setProductId(productId);  
            gcFtZiYiOtpConfig.setProcessId(processId);  
            gcFtZiYiOtpConfig.setPackageId(packageId);  
            ftService.saveGcFtZiYiOtpConfig(gcFtZiYiOtpConfig);  
  
            WebUtil.writeJson(response, ReponseJSONBuilder.buildSuccessMsg(JSONUtils.toJSONString(map)));  
        }catch(Exception e){  
            WebUtil.writeJson(response, ReponseJSONBuilder.buildErrorMsg(e));  
        }  
        return WebUtils.NULLActionForward;  
    }  
  
    private ActionForward search(HttpServletRequest request, HttpServletResponse response) {  
        Map<String, Object> map = new HashMap<>();  
        try{  
            String productId = WebUtils.getParameter("productId", request);  
            String processId = WebUtils.getParameter("processId", request);  
            String packageId = WebUtils.getParameter("packageId", request);  
            List<GcFtZiYiOtpConfig> gcFtZiYiOtpConfigList = ftService.getGcFtZiYiOtpConfigList(productId, processId, packageId);  
            map.put("data", gcFtZiYiOtpConfigList);  
  
            WebUtil.writeJson(response, ReponseJSONBuilder.buildSuccessMsg(JSONUtils.toJSONString(map)));  
        }catch(Exception e){  
            WebUtil.writeJson(response, ReponseJSONBuilder.buildErrorMsg(e));  
        }  
        return WebUtils.NULLActionForward;  
    }  
  
}
```