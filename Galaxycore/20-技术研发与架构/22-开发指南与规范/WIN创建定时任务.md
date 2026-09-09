---
title: WIN创建定时任务
date: 2026-05-03
tags: [MES, 开发]
---
#定时任务

```bash
schtasks /create /tn ShutdownIE_1 /tr "D:\\\\ShutdownIe.cmd" /sc daily /st 07:00

```

- `/CREATE` 表示创建一个新任务。
- `/SC DAILY` 表示任务的频率是每天。
- `/TN "MyTask1"` 指定任务的名称。
- `/TR "notepad.exe"` 指定任务需要执行的命令或程序。
- `/ST 09:00` 指定任务开始的具体时间。

```bash
taskkill /F /IM iexplore.exe //关闭进程

清理IE浏览器缓存
@echo off
RunDll32.exe InetCpl.cpl,ClearMyTracksByProcess 2 (Deletes Cookies Only)

"C:/Program Files/Internet Explorer/iexplore.exe"

```