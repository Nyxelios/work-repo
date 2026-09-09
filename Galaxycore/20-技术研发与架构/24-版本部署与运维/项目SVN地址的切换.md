---
title: 项目SVN地址的切换
date: 2026-05-03
tags: [MES, 运维]
---

mes之前一直用的上扬（供应商）的代码服务器来管理，现在需要把代码改到我们的服务器上管理了。
主管的做法是把代码全部copy一份到我们的打包服务器上的，然后我们在重定位项目的svn的地址，但是我使用svn重定位会报错。
![](../../附件/Pasted%20image%2020250813095145.png)
重定位之后无法更新，提交代码，显示uuid不一致。
https://www.iteye.com/blog/usench-2187048

---

由于过于难搞，我选择直接clone一份新的项目，然后在重新配置下吧。1.创个文件夹，checkout下来。
![](../../附件/Pasted%20image%2020250813095453.png) 2.这个文件很重要！
![](../../附件/Pasted%20image%2020250813100326.png)

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<eclipse-userlibraries version="2">
    <library name="weblogic" systemlibrary="false">
        <archive path="C:\Oracle\Middleware\wlserver_10.3/server/lib/weblogic.jar"/>
        <archive path="C:\Oracle\Middleware\modules/javax.servlet_1.0.0.0_2-5.jar"/>
        <archive path="C:\Oracle\Middleware\modules/javax.jsp_1.3.0.0_2-1.jar"/>
        <archive path="C:\Oracle\Middleware\modules/javax.annotation_1.0.0.0_1-0.jar"/>
        <archive path="C:\Oracle\Middleware\modules/javax.mail_1.1.0.0_1-4-1.jar"/>
        <archive path="C:\Oracle\Middleware\modules/javax.mail_1.4.jar"/>
        <archive path="C:\Oracle\Middleware\modules/javax.ejb_3.0.1.jar"/>
        <archive path="C:\Oracle\Middleware\modules/javax.jms_1.1.1.jar"/>
        <archive path="C:\Oracle\Middleware\modules/com.bea.core.datasource6_1.10.0.0.jar"/>
        <archive path="C:\Oracle\Middleware\modules/com.bea.core.weblogic.rmi.client_1.11.0.0.jar"/>
        <archive path="C:\Oracle\Middleware\modules/javax.transaction_1.0.0.0_1-1.jar"/>
        <archive path="C:\Oracle\Middleware\modules/org.springframework.web.servlet-3.0.5.RELEASE.jar"/>
    </library>
</eclipse-userlibraries>
```

3.配置ivy，选择默认的，依赖会下载到用户文件夹下
![](../../附件/Pasted%20image%2020250813102138.png)
![](../../附件/Pasted%20image%2020250813102200.png)4.修改build配置
![](../../附件/Pasted%20image%2020250813104222.png) 5.配置ant
![](../../附件/Pasted%20image%2020250813104446.png)

> 其他的应该没了
