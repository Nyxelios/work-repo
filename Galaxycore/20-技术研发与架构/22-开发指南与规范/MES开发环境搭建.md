# MyCIM2 / MES 开发环境搭建详细教程 (新手版)

> 来源：根目录 | 迁移日期：2026-05-09  
> 归类：运维手册 - 环境配置

本教程整合了项目所需的所有环境搭建步骤，旨在帮助新手快速上手。

## 一、 基础软件准备

在开始前，请确保获取并安装以下版本的软件：

- **JDK**: 必须使用 **JDK 1.7** 版本。
- **Ant**: 必须使用 **Ant 1.9.7** 或以上版本。
- **WebLogic**: 10.3.6.0。
- **数据库**: Oracle 11g 或更高版本。
- **IDE**: Eclipse。

### 1. JDK 安装注意事项

- **路径冲突避坑**：安装时如果提示安装 JRE，务必将其安装在与 JDK **不同** 的目录下。若安装在同一目录下，`javac.exe` 可能会被冲突删除，导致后期打包失败。

### 2. Ant 环境配置

- 解压 Ant 后，配置 `ANT_HOME` 环境变量指向解压目录。
- 在 `Path` 中添加 `%ANT_HOME%\bin`。
- 验证：在 cmd 输入 `ant -version` 查看版本。

---

## 二、 数据库环境搭建 (Oracle)

### 1. 创建表空间

请以 `sysdba` 身份登录并执行脚本。确保根据实际磁盘路径修改文件位置：

```sql
-- 示例：创建基础表空间
create tablespace TBS_MYCIM datafile 'E:\app\Administrator\oradata\tablespace\TBS_MYCIM.dbf' size 200M;
ALTER DATABASE DATAFILE 'E:\app\Administrator\oradata\tablespace\TBS_MYCIM.dbf' AUTOEXTEND ON NEXT 200M ;
-- (请参照《MES创建表空间.sql》完成所有表空间的创建)
```

### 2. 用户授权与数据导入

```sql
-- 创建用户并授权
create user mestest identified by mestest;
grant dba to mestest;

-- 导入 DMP 数据文件
-- cmd导入示例
impdp mestest/mestest remap_schema=mesprod:mestest directory=mesdb dumpfile=20191204MES002.DMP full=y
```
