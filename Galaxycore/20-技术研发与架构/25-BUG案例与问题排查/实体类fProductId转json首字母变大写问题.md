---
title: 实体类fProductId转JSON首字母变大写
date: 2024-10-21
author: MES Team
status: completed
tags: [BUG, 开发]
---

遇到了个坑死我的问题了，实体类我写的fProductId，但是用了@DATA注解，自动生成的方法时getTProductId，导致json也就变成了TProductId
下次再也不命令这种抽象的变量名了。TAT