浏览器参数传参时包含#，#后的数据会丢失
# 一.问题

使用get请求时，请求参数<span class="cloze-span">中存在#导致后端</span>request获取不到值

## 发出参数带#的请求

![](https://img2023.cnblogs.com/blog/2135082/202311/2135082-20231107153507810-1339649754.png)

## 后端接收不到SKU的值，连后面platformId的值都没有了

![](https://img2023.cnblogs.com/blog/2135082/202311/2135082-20231107153714613-1382299445.png)

# 二.原因

1、有些符号[参数包含有特殊字符（%、#、&）]在URL中是不能直接传递的，如果要在URL中传递这些特殊符号，那么就要使用他们的编码了。

　　编码的格式为：%+字符的ASCII码，即一个百分号%，后面跟对应字符的ASCII（16进制）码值。

　　例如 空格的编码值是"%20"。 

![](https://img2023.cnblogs.com/blog/2135082/202311/2135082-20231107162955321-1667567922.png)

2、url参数有长度限制，参数太长就会显示不全

# 三.解决方案

js采用encodeURIComponent技术进行编译，将#号转化为%23值

## 测试方案

![复制代码](https://assets.cnblogs.com/images/copycode.gif)

const skus = "ZZY210730610RDS，#C262E";
console.log(encodeURIComponent(JSON.stringify(skus.replace(/\#/g,"%23"))));
console.log(JSON.stringify(skus.replace(/\#/g,"%23")));
console.log(skus.replace(/\#/g,"%23"));
console.log(skus);
console.log(JSON.stringify(skus));
console.log(encodeURIComponent(JSON.stringify(skus)));
console.log(encodeURIComponent(skus));

![复制代码](https://assets.cnblogs.com/images/copycode.gif)

## 测试结果

在研究发现了encodeURIComponent编译可直接转换， 就尝试使用console.log(encodeURIComponent(skus));得到[ZZY210730610RDS%EF%BC%8C%23C262E]，就可以满足隐藏#的需求了，而且在后端也会自动转换为#

![](https://img2023.cnblogs.com/blog/2135082/202311/2135082-20231108110138145-2033486832.png)

# 四.案例

## 前端请求

![复制代码](https://assets.cnblogs.com/images/copycode.gif)

/* 设置参数 */  
let data = "year=" + year + "&platformId=" + platformId + "&departmentId=" + departmentId
                + "&repositoryIds=" + repositoryIds + "&skus=" + encodeURIComponent(skus);  
/* 拼接连接 */
let url = CONTEXT_PATH + '/wzw/export/siteSku?' + data;\

/*发出请求：get*/  
window.open(url, '_blank').location;

![复制代码](https://assets.cnblogs.com/images/copycode.gif)

## 后端接收

![](https://img2023.cnblogs.com/blog/2135082/202311/2135082-20231108111810047-415144635.png)

#  五.总结

1.这里原本想用替换r==eplace,== 但是这个过于麻烦，后端还要转换回来

2.后来看到使用JSON.stringify就能成功的，后面因为他这里时改为String类型，反而多了""号更麻烦了

3.在研究发现了encodeURIComponent编译可直接转换， 就尝试使用console.log(encodeURIComponent(skus));得到[ZZY210730610RDS%EF%BC%8C%23C262E]，就可以满足隐藏#的需求了，而且在后端也会自动转换为#

4.要注意数据源的规范，一些唯一固定值可以避免这些特殊字符

5.有疑问的点：明明%号也是在特殊字符列，为什么这里转成十六进制后却可以进行传输了呢