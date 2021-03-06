
---
date: 2017-03-03T08:36:25+08:00
title: "如何用Go实现一款类似滴滴优步的网络约车软件(含源码)"
description: ""
disqus_identifier: 1488501385251770496
slug: "ru-he-yong-Goshi-xian-yi-kuan-lei-shi-di-di-you-bu-de-wang-lao-yao-che-ruan-jian-(han-yuan-ma-)"
source: "http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&amp;mid=2653548297&amp;idx=1&amp;sn=71fc309da16f1ed2b3d8b0d847f730a0&amp;scene=0"
tags: 
-  
categories:
- 编程语言与开发
---

导读：我们经常使用打车软件出行，也经常思考其架构设计。本文作者在所在国家也负责开发一款打车软件，并且开源了其中大部分代码，可以帮助我们更好了解网络约车软件的架构体系。本文由高可用架构翻译。

\

![](https://static.yushuangqi.com/blog/2017/0303081604b1r0nhhzaek.jpg)

\

各位读者好，本文将给大家分享我们如何通过内存存储实现地图动画车效果。
我们公司也运营了一个类似 Uber 的软件 Namba
Taxi，我们需要在客户端主屏幕上显示动画车。
这篇文章是关于功能如何完整实现的文章，主要目的不是介绍 Go 语言。\

\
-

**开始**
--------

\

这个故事始于2015年，我们的移动开发人员开发一款软件，工作主题是为出租车司机提供打车服务。
在应用程序中，动画汽车看起来像下面的图中动画那样 [1] 。

\
\

![](https://static.yushuangqi.com/blog/2017/03030816040dyj3xaavt4.jpg)

\
我们的第一个挑战是缺乏地图跟踪数据。我们每 15 秒获取一次位置数据。
我们不能简单减小上报间隔，因为当司机端程序上行数据时候，同时需要获取当前订单，下一个订单，以及一些警报功能（一个SOS按钮，
当司机按下它，其他司机就可以帮助他）。当我们减少更新间隔时，系统流量更大。
我们不确认我们是否能够扛住如此大的刷新。

\
-

**实现的第一步**
----------------

\

我们第一次的尝试比较简单：\

\

1.  处理请求并保存坐标。

2.  创建另一个请求并为汽车设置动画。

\

显而易见，这样做存在一些问题，如大家在一些打车软件所见，我们不能正确地绘制汽车路线，汽车可能跑在田野，森林，湖泊和公寓上，用这种方法后效果看起来是这样的
[2]。\

\

![](https://static.yushuangqi.com/blog/2017/0303081604stk5julcps4.jpg)

\
作为问题的解决方案，我们使用 OpenStreetMap Routeing
Machine（OSRM）来规划线路并改进我们的算法，并使用相同的超时设置。\

\

1.  发起请求。

2.  获取坐标。

3.  将保存的坐标发送到服务器。

4.  通过 OSRM 构建路线。

5.  返回数据到客户端。

\

通过线路规划体系，现在似乎可以工作了，但我们又面临单向道路的问题\

\

![](https://static.yushuangqi.com/blog/2017/030308160541phrsr0dud.jpg)

\

例如，司机停留在红点的十字路口。
但他的设备位置准确性有问题，导致数据标记在十字路口的对面。
在客户端，我们获取这些坐标，保存并发送到后端，OSRM
建立一个合法的路线，并返回给应用程序。因为客户端移动得非常快，所以这种情况路线规划很可笑。\
\
我们以一种朴素的方式解决了这个问题。
**我们检查两点之间的最短距离，并且不建立距离小于 20 米的路线。**
使用该算法经过几天的测试后，我们决定发布我们的应用程序并希望获取一些反馈。\
\
尽管如此，我们的版本还存在一些问题，所以我们决定进行第二次迭代。\

\

1.  第一是车费计算器，**计算是在司机端（客户端）完成，这样避免发送无用的请求**，可以节约很多服务端资源。
    另一方面，为了安全等方面考虑，我们需要在服务器端复制数据并保存它。

2.  此外，我们意识到每 15
    秒一次上报太少，因为用户在屏幕打开后，15秒后才会看到车在移动。

3.  此外，我们在司机端的 GPS
    模块有很多问题，这个可能跟司机的手机设备相关。

4.  最后，我们想要在主屏幕上渲染动画车。

**\
**
---

**还需要解决的问题**
--------------------

**\
**

1.  从司机收集更多的数据

2.  在主屏幕上显示动画车

3.  在服务器端存储行车过程中计费数据

4.  节约移动流量

5.  每秒收集一次数据

\

我想谈一谈有关节约移动流量带宽的问题。在我们国家，出租车收费非常便宜，我们像使用公共交通那样使用出租车。
例如，从城市的一边跑到另一边可能只需要 2
欧元，这就跟在巴黎坐地铁价格差不多。但另外一方面移动带宽成本还也很高，如果我们每秒节约
100 字节，那么我们将给为公司节省差不多 2000 美元。\

\
-

**数据追踪**
------------

**\
**

1.  司机位置（纬度，经度）

2.  司机当前的 session 信息，在登录时我们会给司机端提供 session id

3.  订单信息（订单 ID 和车费）

\

我们决定每一次数据上报应小于 100 字节。 我们寻找传输协议来解决这个问题\
\

正如你可以看到，我们审视了以下几个协议：\

\

1.  HTTP

2.  WebSockets

3.  TCP

4.  UDP

\

**对我们来说理想的选择是 UDP**，因为：\

\

1.  我们只发送数据报

2.  我们不需要保证送达

3.  极简主义

4.  保存大量数据

5.  只有 20 字节开销

6.  在我们的国家的移动网络没有被阻止

\

至于数据序列化，我们考察了：\

\

1.  JSON

2.  MsgPack

3.  Protobuf

\

**我们选择 ProtoBuf**，因为它对小数据非常有效。

\

![](https://static.yushuangqi.com/blog/2017/03030816054hshsrdpzdh.png)

\

以看到最近的竞争对手是 PB 的三倍。（小编：可以参考 TimYang 的一条微博
[3] ）\

\

**每次上报总共的数据**
----------------------

**\
**

1.  42 字节的业务数据

2.  加上 20 字节的 IP 报头

3.  得到每次上报 62 字节数据

\

当我们获得数据时，我们考虑如何存储。\

\
-

**数据存储**
------------

**\
**

我们需要存储这些数据：\

\

1.  标识司机的会话信息 session id

2.  车牌号

3.  订单 ID 和计费信息

4.  执行搜索的最后位置

5.  N 次最后位置以规划路线

**\
**
---

**使用的存储**
--------------

**\
**

1.  使用 Percona 存储所有数据。 我们存储司机，订单，计费等。

2.  Redis 作为用于缓存。

3.  Elasticsearch 用于地理编码

\

如上所述，当有大量在线司机时候，使用这些存储来保存数据并不方便。
所以我们需要地理索引。\
\

我们评估了两个地理索引：\

\

1.  KD 树

2.  R 树。

\

我们对地理索引的要求：\

\

1.  搜索 N 个最近的点。

2.  我们需要一个平衡树，以在最糟糕的情况下提供最好的搜索

\
-

KD 树![](https://static.yushuangqi.com/blog/2017/0303081605q0ycsanmc4y.png)
---------------------------------------------------------------------------

KD 树并不适合我们的需要，因为它是不平衡的，只能搜索一个最近的点。
我们可以在 kd-tree 上实现 k-nearest 邻居，但是没必要重造轮子，因为
R-tree 已经解决了这个问题。\

\

R-树
----

\

![](https://static.yushuangqi.com/blog/2017/03030816054tqwmqrxkf4.jpg)

\

它看起来像这样。 我们可以执行搜索 N 个最近点，并且它是平衡树。
我们选择了这个。\
\

您可以得到它的 Go 语言实现源码 [5]。

\
另外，我们需要一个过期机制，因为我们需要使司机的超时机制，比如司机端 900
秒没有响应则在服务器删除会话。 所以我们需要 LRU
数据结构来存储最后的位置。 同时因为我们只存储 N 个位置。
如果我们尝试添加数据时候，队列存储已满，我们则删除最少使用的那个条目。 

\

下面是我们的存储架构。\

\

1.  我们将所有数据存储在内存中。

2.  **我们使用 R-tree 执行搜索最近的司机**。

3.  此外，我们使用两个检索图，可以并按车牌号或session执行搜索

\
-

**我们打车软件最终算法**
------------------------

**\
**

这里是后端的最终算法：\

\

1.  使用 UDP 传输数据

2.  尝试从存储获取司机

3.  如果存储不存在 - 则从 Redis 获取司机

4.  检查并验证数据

5.  将司机保存到存储

6.  如果不存在 - 初始化 LRU

7.  更新 r-tree

    \

**HTTP 接口**
-------------

\

我们实现了这些接口：\

\

1.  返回最近的司机；

2.  从存储中删除司机（通过车牌号或session id）

3.  获取行程信息

4.  获取司机信息

**\
**
---

**结论**
--------

\

最后，我想给出我们在后端系统中总结的经验：\

\

1.  UDP + Protobuf 以节省数据

2.  内存存储

3.  R 树获取最近的司机

4.  LRU 缓存用于存储最后的 n 个位置

5.  OSRM 用于地图匹配和定制路线

    \

    \

    ![](https://static.yushuangqi.com/blog/2017/0303081605v2jo3phsfr3.jpg)

\

您可以在 github
[5] 上查看上面整个过程的源代码。现在功能还比较简单，但实现了文章中描述的许多功能。

\

**参考资源**

-   [GIF
    动画下载](https://cdn-images-1.medium.com/max/1600/1*nI6cNApASR1mg6F5Sjgp7Q.gif%0Ahttps://cdn-images-1.medium.com/max/1600/1*KfGB1SARPoqOUtPtl4NNBg.gif)
-   [http://weibo.com/10503/24F1QpDmL](http://weibo.com/10503/24F1QpDmL)
-   [https://github.com/dhconnelly/rtreego](https://github.com/dhconnelly/rtreego)
-   [https://github.com/maddevsio/openfreecabs](https://github.com/maddevsio/openfreecabs)
-   [本文英文原文](https://blog.maddevs.io/how-we-built-a-backend-system-for-uber-like-map-with-animated-cars-on-it-using-go-29d5dcd517a)

**推荐阅读**

\

-   [Uber容错设计与多机房容灾方案](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=209153643&idx=1&sn=e0a64a8e7fcb8b43ceac85cc5782719f&scene=21)\

-   [为什么Uber宣布从Postgres切换到MySQL?](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653547609&idx=1&sn=cbb55ee823ddec9d98ef1fa984e001f6&scene=21)

-   [滴滴passport设计之道:帐号体系高可用的7条经验(含PPT)](http://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653547482&idx=1&sn=13675fae5e037d720a9e9fb4a4861afd&scene=21)

\

本文由高可用架构翻译，转载请注明出处，技术原创及架构实践文章，欢迎通过公众号菜单「联系我们」进行投稿。

