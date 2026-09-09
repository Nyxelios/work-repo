# 一、MES开发环境搭建及项目介绍

# 1. MyCIM2环境安装

## 1.1 安装JDK7

> 百度网盘地址：链接:[https://pan.baidu.com/s/1X_rfBaVqC68-Ood4AoP6UQ](https://pan.baidu.com/s/1X_rfBaVqC68-Ood4AoP6UQ) 密码:ys09
如何安装具体百度
> 

## 1.2 安装ANT

> 百度网盘地址：链接:[https://pan.baidu.com/s/1yVzEAtxKUtNzIW-xDAoN9w](https://pan.baidu.com/s/1yVzEAtxKUtNzIW-xDAoN9w) 密码:rba5
直接解压即可。`配置环境变量 ANT_HOME 指向解压路径`。并在 path 中引用。windows的此
处不介绍和配置JAVA环境变量一样。linux/mac参考如下
> 

```bash
vi /etc/profile
ANT_HOME= /Users/apple/Desktop/JAVA/apache-ant-1.9.6
PATH=$PATH:$ANT_HOME/bin
export PATH ANT_HOME
```

**安装校验**

```bash
ant -version
#Apache Ant(TM) version 1.9.6 compiled on June 29 2015
```

## 1.3 安装weblogic

> 百度网盘地址：链接:[https://pan.baidu.com/s/1D0oiYgAdJjxbd-dsqOhnsg](https://pan.baidu.com/s/1D0oiYgAdJjxbd-dsqOhnsg) 密码:fu9d
> 

### 1.3.1 启动weblogic

> `start_weblogic.cmd` 解释
> 

```bash
@Rem 相应weblogic的路径
cd /d C:\Users\guoxunbo\Desktop\FA_dev_envirment\weblogic\user_projects\d
omains\base_domain\bin
::weblogic路径
set WL_HOME=C:\Users\guoxunbo\Desktop\FA_dev_envirment\weblogic\wlserver_
10.3
::启动weblogic所对应的域
set DOMAIN_HOME=C:\Users\guoxunbo\Desktop\FA_dev_envirment\weblogic\user_
projects\domains\base_domain
::设置JDK是否是64位
set JAVA_64BIT=true
::启动debug模式
set debugFlag=true
::对应的DEBUG端口
set DEBUG_PORT=8787
::设置JAVA基本的堆栈 如果是32位需要加上-d64不然内存最多只能用到2G
set JAVA_MEMORY_OPTIONS_64BIT=-Xms1024m -Xmx2048m
set JAVA_MEMORY_OPTIONS_32BIT=-d64 -Xms512m -Xmx1024m
::设置JAVA基本的元空间
set JAVA_PER_MEMORY_OPTIONS_64BIT=-XX:PermSize=256m -XX:MaxPermSize=512m
set JAVA_PER_MEMORY_OPTIONS_32BIT=-XX:PermSize=128m -XX:MaxPermSize=256m
startWebLogic
```

> `启动成功`标志
> 

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled.png)

### 1.3.2 配置数据源 将数据源改成自己的数据源即可

> 访问 `ip:7001/console` 出现如下页面 使用用户名密码`admin/admin123`进行登录
> 

> 配置数据库
> 

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%201.png)

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%202.png)

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%203.png)

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%204.png)

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%205.png)

### 1.3.3 配置数据库连接池->console修改

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%206.png)

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%207.png)

### 配置数据库连接池->直接修改文件

> 修改 `weblogic安装目录/user_projects\domains\base_domain\config\jdbc`
> 

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%208.png)

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%209.png)

修改数据源之后，重新启动配置才生效

## 1.4  使用eclipse

> 我用的是IDEA，相关配置是类似的。
> 

### 1.4. 安装ANT-ivy

> 打开 `Eclipse->Help->Eclipse Marketplate` 进行安装ivy 如果已安装就忽略
> 

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%2010.png)

> 导入ivy依赖的jar包
> 

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%2011.png)

### 1.4.2 eclipse修改weblogic lib

> 百度网盘地址：链接:[https://pan.baidu.com/s/1b1v5hOjtYdlNA5ac5kSLHw](https://pan.baidu.com/s/1b1v5hOjtYdlNA5ac5kSLHw) 密码:dkjz
将 `org.springframework.web.servlet-3.0.5.RELEASE.jar` 拷贝到 `weblogic` 的
安装目录`/modules` 目录下。
修改 `weblogic.userlibraries` 文件。将里面的引用路径换成自己的路径。
> 

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%2012.png)

> 删除weblogic library重新添加
> 

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%2013.png)

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%2014.png)

### 1.4.3 修改build.properties

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%2015.png)

### 1.4.4 eclipse配置远程debug模式

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%2016.png)

![Untitled](../../附件/CIM01-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-Untitled%2017.png)

debug配置中 `source` 即自己所拥有的源代码。

# 2. MyCIM2项目介绍