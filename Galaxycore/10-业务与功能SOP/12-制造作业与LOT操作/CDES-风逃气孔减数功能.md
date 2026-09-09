---
title: CDES-风逃气孔减数功能
date: 2026-05-03
tags: [MES, LOT]
---
参照 CQC 过程功能检

---

# 研究CQC的减数

![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241021161834.png)

查看CQC的减数，发现记录表有一张报废表，表号是在类表定义和类表值定义的。

![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241021161854.png)
![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241021161908.png)


于是我按照该规则配置了CDES的减数表，但是却发现工步下拉内没有我定义的表号。

![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241021161924.png)
![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241021161932.png)

原因是报废记录表号的数据也是从一个类表里查询的。

![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241021162004.png)

寻找该类表以及类表的值

![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241021162019.png)

![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241021162030.png)

添加上自己的类表，并且配置CDES即可

![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241021162043.png)
