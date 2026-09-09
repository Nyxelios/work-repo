---
title: Windows与Linux文件互传
date: 2026-05-03
tags: [MES, 运维]
---

# 服务包发送到11.12.41.42

# 远程win传linux

本来想本地电脑控制传打包服务器到四个linux的，发现不行，因为window的ssh没开22端口

```
scp D:\Oracle\Middleware\user_projects\domains\base_domain\autodeploy\mycim2.zip root@10.181.160.11:/app_mes/weblogic/user_projects/domains/base_domain/stage
```

```
scp D:\Oracle\Middleware\user_projects\domains\base_domain\autodeploy\mycim2.zip root@10.181.160.12:/app_mes/weblogic/user_projects/domains/base_domain/stage
```

```
scp D:\Oracle\Middleware\user_projects\domains\base_domain\autodeploy\mycim2.zip root@10.181.160.41:/app_mes/weblogic/user_projects/domains/base_domain/stage
```

```
scp D:\Oracle\Middleware\user_projects\domains\base_domain\autodeploy\mycim2.zip root@10.181.160.42:/app_mes/weblogic/user_projects/domains/base_domain/stage
```
## 补充

```jsx
scp 源文件/文件夹 主机用户名@主机IP:目标文件夹
```

1. 源文件/文件夹：D:\Oracle\Middleware\user_projects\domains\base_domain\autodeploy\1.txt
2. 主机用户名：root
3. 主机IP：10.181.160.11
4. 目标文件夹：/app_mes/weblogic/user_projects/domains/base_domain/stage

# SCP命令

首先，让我们简要了解一下**scp命令**。scp（secure copy）是一个基于SSH协议的安全文件复制工具，可以在本地与远程主机之间或两台远程主机之间传输文件。**scp命令**简单易用，同时保证了数据传输的安全性。

## 1.基础的文件传输

最简单的scp命令用法是从本地向远程主机传输文件：

```text
scp file.txt user@remotehost:/path/to/directory
```

这里的file.txt是要传输的文件，user是远程主机上的用户名，remotehost是远程主机的地址，/path/to/directory是远程主机上的目标目录。
## 2.使用密钥认证

为了提高安全性，可以通过SSH密钥认证代替密码输入：

```text
scp -i ~/.ssh/id_rsa file.txt user@remotehost:/path/to/directory
```

这里 **-i** 选项指定了私钥文件的位置。

## 3.复制整个目录

使用-r选项可以复制整个目录：

```text
scp -r directory/ user@remotehost:/path/to/directory
```

## 4.从远程主机复制文件

也可以从远程主机复制文件到本地：

```text
scp user@remotehost:/path/to/file.txt .
```

这里的.表示当前目录。
## 5.同时传输多个文件

可以一次传输多个文件：

```text
scp file1.txt file2.txt user@remotehost:/path/to/directory
```

## 6.递归传输多个目录

同时复制多个目录时也需要使用-r选项：

```text
scp -r directory1/ directory2/ user@remotehost:/path/to/directory
```

## 7.传输文件的同时更改权限

可以使用-p选项保持文件的权限不变：

```text
scp -rp file.txt user@remotehost:/path/to/directory
```

## 8.跨越多台远程主机传输

scp还支持跨越多台远程主机传输文件：

```text
scp file.txt user@remotehost1:/path/to/directory user@remotehost2:/path/to/directory
```
## 9.指定端口

如果远程主机使用了非标准端口，可以通过-P选项指定端口号：

```text
scp -P 2222 file.txt user@remotehost:/path/to/directory
```

## 10.传输进度条

虽然scp不直接支持进度条，但可以借助pv工具实现：

```text
pv file.txt | scp -i ~/.ssh/id_rsa - - user@remotehost:/path/to/directory
```

这里**pv**工具用来显示传输进度，-作为scp的输入，表示从标准输入读取数据。

通过上述示例，可以看出scp命令的强大之处不仅在于其简单易用，更重要的是它可以灵活应用于各种场景。无论是单个文件的传输，还是整个目录的复制，甚至是跨多台远程主机的数据迁移，**scp**都能够胜任。


