
# FT、RT、FA站点第二次扫描后结束不显示信息
# 需求

💡 在开始结束作业页面，左下角需要扫描批次，然后开始作业，第二次扫描右下角不需要显示信息。实际尝试后发现，第二次扫描后，会直接跳转到结束页面，因此，**该需求实际上是在开始作业页面跳转回来后右下角不显示信息**

![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241017133433.png)

# 探索

💡 **需要知道右下角的信息是通过后端的哪个方法获取的，所以需要debug**
通过打多个断点，知道是`LotOfEquip4ExtAction.java`的`viewJobList`方法
![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241017133604.png)

 💡 **总感觉怪怪的，问了一下后，需求其实就是该三个站点的右下角不显示信息就好了，增加一个站点判断的逻辑即可。**

- **在该方法内添加如下代码**

```java
if (station.getStationName().equals("FT300")
            || station.getStationName().equals("FT400")
            || station.getStationName().equals("FT500")){
      JSONObject result = new JSONObject();
		  result.element("results", jsonArray.size());
		  result.element("rows", jsonArray);
		  renderJson(response, result.toString());
}
...
```

结果：点击三个站点，右下角不显示数据了，点击其余站点刷新数据，**`但是再次点击三个站点，信息依然存在。`**

- 代码做如下修改

```java
if (!station.getStationName().equals("FT300")
          && !station.getStationName().equals("FT400")
          && !station.getStationName().equals("FT500")) {
... ....
}

JSONObject result = new JSONObject();
result.element("results", jsonArray.size());
result.element("rows", jsonArray);
renderJson(response, result.toString());
```

**结果：达到预期**

# ?

<aside> 💡 代码提交上去后，也没报错，但是三个站点第二次扫描没有跳转到结束界面，不清楚什么情况。上午没问题，下午就出错，下次我测试成功一定要录个屏。

</aside>

……

发现了，原来上午测试的只是右下角的显示，没有实际测试数据的入站出站操作，显示是不显示，但是出站没有自动跳转。

# 站点名称不一致

**测试与正式上的站点名称不一致。**
![image.png](https://obsidian-oss-sync.oss-cn-hangzhou.aliyuncs.com/pictures/20241017133936.png)

**需要获取站点名称来判断是否在三个站点，可以获取到id值和站点名，但是获取到后还是要和本地的常量对比，这个常量测试和本地是不一样的。Id和站点名都不一样，没办法确定一个测试和正式共同使用的方法**


# FT ATE FT RT FA 站点MES逻辑调整+报表抓取逻辑变更

## MES FT RT FA过站
| before                                                                       | now                                                                                                   |
| :--------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------- |
| MES 在 FT RT 站点不允许产生 F；<br>产线员工手工录入 key 数量；<br>当前不校验过站文件数量与人工 key 的 F 数量是否一致。 | 允许 FT RT 站点产生 F 等级；<br>人员 key 的 F 数量必须与过站文件中的 F 数量保持一致；<br>若存在少料、压碎等芯片，相关数量需 key 入 RT 数量中，避免误录入 F 数量。 |

## MES HOLD缺陷率抓取
| before                                                             | now                                                                          |
| :----------------------------------------------------------------- | :--------------------------------------------------------------------------- |
| MES HOLD 料检测逻辑中，当前是获取物料在 FA 产出的 F 数量，并基于该数量计算缺陷率，再进行数据分析处理是否 HOLD。 | MES HOLD 料检测逻辑调整为：获取物料在 FT、RT、FA 三站产出的 F 数量，并进行累加求和，再按汇总结果计算缺陷率，最终决定是否 HOLD。 |
|                                                                    |                                                                              |

