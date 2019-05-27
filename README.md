# my blog

# 这是一级标题
## 这是二级标题
**加粗**
*倾斜*
***加粗倾斜***
~~这是加删除线的文字~~

>这是引用的内容


___ 分割线
____

![图片alt](图片地址 ''图片title'')

图片alt就是显示在图片下面的文字，相当于对图片内容的解释。
图片title是图片的标题，当鼠标移到图片上时显示的内容。title可加可不加

![blockchain](https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/
u=702257389,1274025419&fm=27&gp=0.jpg "区块链")


超链接

[超链接名](超链接地址 "超链接title")
title可加可不加


[简书](http://jianshu.com)
[百度](http://baidu.com)



无序列表
* 列表内容
* 列表内容

有序列表
1. 列表内容
2. 列表内容

列表嵌套(下级比上级缩进一个tab)
* java书籍   
    * java编程思想
    * java从入门到精通
    
代码：
`System.out.println("代码之间用反引号包起来")`

代码块用三个反引号包起来  反引号单独占一行
```
   public static <T> void splitAndProcessList(List<T> list, Integer size, Consumer<List<T>> function) {
        int length = list.size();
        int count = length / size;
        Stream.iterate(0, n -> n + 1).limit(count + 1).forEach(n -> {
            List<T> collect = list.stream().skip(n * size).limit(size).collect(Collectors.toList());
            if (!CollectionUtils.isEmpty(collect)) {
                function.accept(collect);
            }
        });

    }
```
![Image text](https://raw.githubusercontent.com/lucky-zhao/blog/master/img/QQ%E6%88%AA%E5%9B%BE20190527180746.png?token=AK4KFZQP3IT2UFZT3TFQHEK45O4RW)
