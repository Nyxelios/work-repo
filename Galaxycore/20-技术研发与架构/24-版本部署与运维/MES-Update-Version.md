---
title: MES-Update-Version
date: 2026-05-03
tags: [MES, 运维]
---
## 0. 前言

> 今天学了关于mes项目部署到服务器的知识。
> 
> 服务器分为正式、测试、UAT（用户测试验收）。
> 正式服务器地址一共有四个，不过发布方式都是一样的，发布四次就可以了。
> 四个服务器分为41,42和11,12两部分。
> 测试、UAT（用户测试验收）发布方式相同，且比较简单。
### 0.1 主机账号&密码

| 主机名      | IP             | 用户名           | 初始密码              |
| -------- | -------------- | ------------- | ----------------- |
| MES AP01 | 10.181.160.11  | root          | abc@123           |
| MES AP02 | 10.181.160.12  | root          | abc@123           |
| RW AP01  | 10.181.160.41  | root          | abc@123           |
| RW AP02  | 10.181.160.42  | root          | abc@123           |
| MESTEST  | 10.181.160.18  | administrator | abc@123           |
| 项目打包服务器  | 172.18.108.53  | administrator | vpM8RcbT9uT5      |
| UAT预发布   | 172.18.108.103 | Administrator | 1qaz2wsx3edc@.com |


> [!NOTE]
> 💡 每次发布，mes的其中一个服务器需要启动，rw的也一样，不然就宕机产线没法用系统了。

## 1. Publish 准备工作

### 1.1 提交最新代码

本地测试完毕后`svn commit`提交最新代码

![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241021164509.png)

### 1.2 连接项目打包服务器
MobaXterm远程连接项目打包服务器172.18.108.53
1. 更新本地代码`SVN update`
2. 启动IDE（eclipse）`run ant build.xml` ，生成mycim2文件
3. 打包成压缩包，发送至本地，打包工作完成
	![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241021164550.png)

## 2. Publish 11（for example）

### 2.1 连接服务器放置压缩包

连接服务器，在规定目录下`/app_mes/weblogic/user_projects/domains/base_domain/stage/`放置压缩包，[[Windows与Linux文件互传]]
#### 11 12 41 42压缩包位置
```
/app_mes/weblogic/user_projects/domains/base_domain/stage/
```
#### 28 29压缩包位置
```
/app_mes/weblogic/Oracle/Middleware/user_projects/domains/base_domain/stage
```

---

```bash
cd /app_mes/mycim_release/2024/
```

```bash
cd /app_mes/release/2024/
```

在`release`下备份，创建日期文件夹，拖入项目压缩包

![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241021164824.png)

### 2.2 execute script

| `clearNohup.sh`       | 每天七点清下控制日志 |
| --------------------- | ---------- |
| `startAdminServer.sh` | 启动weblogic |
| `startMes.sh`         | 启动mes项目    |
| `stopAdminServer.sh`  | 关闭weblogic |
| `stopMes.sh`          | 关闭mes项目    |
#### 1. 切换到`mes`启动目录下：`cd /app_mes/sh`
```bash
cd /app_mes/sh

```
#### 2. 先执行`./stopMes.sh`命令
```bash
./stopMes.sh

```
#### 3. 再执行`./stopAdminServer.sh`
```bash
./stopAdminServer.sh

```
#### 4.替换文件
停止两个服务后，连接xftp（也可以直接在命令行删除），删除下图目录下的`myCim`文件夹，然后**替换打包好的文件**
- 删除可以直接右键删除，也可以`rm -rf myCim2`删除
- 目录：`\\app_mes\\weblogic\\user_projects\\domains\\base_domain\\stage`

##### 11 12 41 42 
```bash
cd /app_mes/weblogic/user_projects/domains/base_domain/stage/

```
##### 28 29
```bash
cd /app_mes/weblogic/Oracle/Middleware/user_projects/domains/base_domain/stage

```

```bash
rm -rf mycim2

```

右键解压或者`unzip myCim2.zip`命令解压。

```bash
unzip mycim2.zip

```

删除解压好的压缩包

```bash
rm -rf mycim2.zip

```
#### 5.删除缓存
删除`\\app_mes\\weblogic\\user_projects\\domains\\base_domain\\servers\\AdminServer`目录下的缓存，`cache，logs，tmp`（weblogic缓存）
##### 11 12 41 42
```bash
 cd /app_mes/weblogic/user_projects/domains/base_domain/servers/AdminServer
 
```
##### 28 29
```bash
 cd /app_mes/weblogic/Oracle/Middleware/user_projects/domains/base_domain/servers/AdminServer
 
```

```bash
rm -rf cache/ logs/ tmp/

```
    
删除`\\app_mes\\weblogic\\user_projects\\domains\\base_domain\\servers\\mes-1`下的缓存，`cache，logs，tmp`（mes缓存）

##### 11 12 41 42
```bash
cd /app_mes/weblogic/user_projects/domains/base_domain/servers/mes-1

```
##### 28 29
```bash
cd /app_mes/weblogic/Oracle/Middleware/user_projects/domains/base_domain/servers/mes-1

```

```bash
rm -rf cache/ logs/ tmp/

```
#### 6.启动sh

切换回启动文件夹`cd /app_mes/sh`

```bash
cd /app_mes/sh

```

执行`startAdminServer.sh`：启动weblogic

```bash
./startAdminServer.sh

```

执行`startMes.sh`：启动mes项目
```bash
./startMes.sh

```

检查是否启动成功，在浏览器输入

|                                                  |                                                  |
| ------------------------------------------------ | ------------------------------------------------ |
| [11](http://10.181.160.11/mycim2/main/main.html) | [10.181.160.11:7003](http://10.181.160.11:7003/) |
| [12](http://10.181.160.12/mycim2/main/main.html) | [10.181.160.12:7003](http://10.181.160.12:7003/) |
| [41](http://10.181.160.41/mycim2/main/main.html) | [10.181.160.41:7003](http://10.181.160.41:7003/) |
| [42](http://10.181.160.42/mycim2/main/main.html) | [10.181.160.42:7003](http://10.181.160.42:7003/) |
| [28](http://172.18.203.28/mycim2/main/main.html) | [172.18.203.28:7003](http://172.18.203.28:7003/) |
| [29](http://172.18.203.29/mycim2/main/main.html) | [172.18.203.29:7003](http://172.18.203.29:7003/) |
| [31](http://10.181.160.31/mycim2/main/main.html) | [10.181.160.31:7003](http://10.181.160.31:7003/) |
| [32](http://10.181.160.32/mycim2/main/main.html) | [10.181.160.32:7003](http://10.181.160.32:7003/) |

> [!NOTE]
> 启动成功才能重复以上操作完成`12`的部署更新
> 更新好`12`后检查`12`是否启动成功
> 然后检查`31`的`vip`是否启动成功

## 3. Test&UAT Deploy

💡 上传压缩包，解压，删除，缓存，运行
