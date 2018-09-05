---
title: 解决“Pod install”慢的问题
date: 2018-09-04 14:07
tags: iOS
---

前段时间使用cocopods安装百度地图的SDK，发现失败了

<img src="/img/BEEABCAD3116C15F85755B6933808675.jpg" width="50%">

<!--More-->

直到我看到了看了这么一篇博客：
[pod install速度慢的终极解决方案 - CSDN博客](https://blog.csdn.net/wuquan0625/article/details/47401235)

现在简单记录一下：
其实真正的原因不是Pod命令，而是在于Github上代码库访问速度变慢，那么我们终极的解决办法就是加快Git命令的速度。        

那么怎么才能加快Git命令的速度呢，当然是`科学上网`了🤫。
我用的是Shadowsocks-NG-R8，具体怎么使用，大家可以自行百度。

打开`Shadowsocks` - `高级设置`选项，可以看到下图所示：

<img src="/img/WX20180904-135254@2x.png" width="50%">

红框所示圈出来的就是下面我们需要用到的。
#### 接下来我们输入命令：
`git config --global http.proxy socks5://127.0.0.1:1086`

如果我们不希望国内Git库也走代理（我觉得还是全局好，国内库也有特别慢的），可以输入命令：
`git config --global http.https://github.com.proxy socks5://127.0.0.1:1086`


#### 当然，如果要恢复/移除上面设置的Git代理
`git config --global --unset http.proxy`
`git config --global --unset http.https://github.com.proxy`


