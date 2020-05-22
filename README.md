# android learning
希望点满android技能树

## update @2020-05-22
时隔这么久，想起来自己有这么一篇完全没有完成的东西，甚是惶恐，还是要重新捡起来，不然容易抑郁。

## 开篇
这段时间有这样一个错觉，总觉得自己懂不少，但是真的说懂了些啥好像又说不出来。想要体系地学习好像是件非常奢侈的事情，所以想着用这样一种方式来分析一下自己知识体系里面的漏洞，尝试画一下技能树，看看自己到底几斤几两。

应用开发涉及到的内容范围广、其内涵也可以很深，所以要在使用的基础上弄清楚内涵，不仅要运行示例代码，还要去追踪源码探索原理。

最好的学习资料是官方文档。不过这个文档非常长，学起来肯定会繁琐。鉴于之前一直徘徊的状态，还是觉得拿这部分内容来啃比较好。 No pain，no gain 吧。

## 基础内容

Android docs 里面包含了以下目录的内容，用以描述所有应用场景中使用的各个控件等的原理。

1.  Introduction
2.  App Components
3.  App Resources
4.  App Manifest
5.  User Interface
6.  Animation and Graphics
7.  Computation
8.  Media and Camera
9.  Location and Sensors
10.  Connectivity
11.  Text and Input
12.  Data Storage
13.  Administration
14.  Web Apps

然后就是各个 practices 的示例代码了。

这次主要是想把这部分内容全部过一遍，用 github 来记录、并鞭策自己努力完成吧。

## Framework

主要包括Android的Framework的分析和学习

1. [SystemServer] (https://github.com/xingchueng/androidlearning/blob/gh-pages/Framework/services/SystemServer.md)
2. [Usb 模式总结] (https://github.com/xingchueng/androidlearning/blob/gh-pages/Framework/services/Android%20USB%E6%A8%A1%E5%BC%8F.md)

## JVM相关

JVM知识比较多，这里要慢慢总结

1. JVM内存结构
2. Class加载机制
3. GC
4. 多线程
5. 同步

主要需要对JVM有比较全面的认识

## Android开发高阶

在需要处理问题的时候，总会遇到这些

1. Activity启动模式
2. Activity启动流程
3. Handler底层原理
4. Binder
5. View绘制
6. Bitmap
7. 插件化
8. 内存优化
9. RecyclerView优化（UI优化）

## 网络

1. HTTPS
2. HTTP 2.0

## 算法

1. 搜索
2. 排序
3. DP
4. 常见算法题
