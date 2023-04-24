[架构设计](https://www.jianshu.com/c/753debf1423d)系列文章，请参见连接。

## 背景
今天在 高效开发运维 的微信公众号上看到一篇文章《[Stack Overflow：我们是如何做监控的](https://mp.weixin.qq.com/s/iiO1EiHXWmOAbQSYqxLQ3A)》原文是[Nick Craver](https://stackoverflow.com/users/13249/nick-craver)编写的、由谢丽翻译的。Nick Craver是Stack Overflow的架构师，在这里感谢原文作者以开放的态度描述了Stack Overflow的内部架构与性能。感谢高效开发运维的翻译与分享。

[Stack Overflow](https://stackoverflow.com/)是一个与程序相关的IT技术问答网站。国内应该有90%以上的程序员是上过Stack Overflow，如果不记得或者不知道可能是在不经意间就被Baidu导流到Stack Overflow上。从这篇文章内我大概体会到Stack Overflow使用的是.Net技术栈，并且在.Net技术栈上使用了各种开源组件，而且Stack Overflow也开源了很多在.Net技术栈下的监控类的软件。

## 原始数据

* #### 原始文章：
一个技术人员要成长、要经验，不仅仅是通过自己不断的摸索，还需要了解业界大佬们都是怎么解决他们的问题的。所以需要不断的了解业界大佬们的技术动向，在了解一个大型网站肯定是看这个大型网站的底层架构以及处理方式。所以，我就根据这篇文章下面的原文地址找了其他的文章。有三篇文章引了我的兴趣：
> [Stack Overflow: The Architecture - 2016 Edition](https://nickcraver.com/blog/2016/02/17/stack-overflow-the-architecture-2016-edition/)
[Stack Overflow: The Hardware - 2016 Edition](https://nickcraver.com/blog/2016/03/29/stack-overflow-the-hardware-2016-edition/)
[Stack Overflow: How We Do Deployment - 2016 Edition](https://nickcraver.com/blog/2016/05/03/stack-overflow-how-we-do-deployment-2016-edition/)

* #### 部署结构：
这三篇文章主要说明了2016时Stack Overflow的各项内容。本文中主要是根据这三篇文章进行一些数据的分析并将原始数据记录在这里。

![Stack Overflow架构](https://upload-images.jianshu.io/upload_images/2454595-d08dbde7e91b8483.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
部署结构图如上图所示，在这里各个服务的情况如下：
|编号|服务|节点个数|
|:-:|:-:|:-:|
|1|Microsoft SQL Servers|4|
|2|IIS Web Servers|11|
|3|Redis Servers|1|
|4|Tag Engine servers|3|
|5|Elasticsearch|3|
|6|HAProxy|2(Web Service) + 2(CloudFlare)|


* ### 硬件结构：
软件服务的前期规划和软件服务持续发展过程中都会针对容量，运维进行规划与设计，在这方面Stack Overflow对硬件的规划和考虑重点包括：硬件弹性的问题，存储，内存，计算，网络，冗余，电源。在这方面Stack Overflow非常喜欢Dell的服务器，在这里并且配合SSD进行数据中心硬件基础的规划：

|编号|硬件|CPU|内存|存储|网络|作用|
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|1|Dell R720xd|Dual E5-2697v2 12 cores @2.7–3.5GHz|384 GB DIMMs|4.2T SSD|Dual 10 Gbps|SQL Servers 2台|
|2|Dell R630|Dual E5-2690v3 12 cores @2.6–3.5GHz|64 GB DIMMs|300GB SATA SSDs|Dual 10 Gbps|Web Servers 2台|
|3|Dell R630|Dual E5-2643<br>Dell R620|Dual E5-2643 v3 6 cores @3.4–3.7GHz<br>Dual E5-2667 6 cores @2.9–3.5GHz|600GB SATA SSD|Dual 10 Gbps|Service Servers (Workers) 2台|
|4|Dell R630|Dual E5-2687W v3 10 cores @3.1–3.5GHz|256 GB DIMMs|240GB SATA SSD|Dual 10 Gbps|Redis Servers 2台|
|5|Dell R620|Dual E5-2680 8 cores @2.7–3.5GHz|192 GB DIMMs|1.6T SATA SSDs|Dual 10 Gbps|Elasticsearch Servers (Search)3台|
|6|Dell R620|Dual E5-2650 8 cores @2.0–2.8GHz|64 GB DIMMs|2TB SATA HDDs|2个Dual 10 Gbps|HAProxy Servers2台|
因为在Stack Overflow上还运维了Stack Exchange服务，所以这里没有把其他的也整理到这里。并且这里面的服务器有没有与Stack Exchange公用也未可知。所以，这里是列出来大概的情况。

* #### 软件结构：
从上可以看到硬件大体结构，软件接入如下所所示：
|编号|软件|版本|作用|
|:--:|:--:|:--:|:--:|
|1|CentOS7|Unknown|操作系统|
|2|HAProxy|1.5.15|load balancers|
|3|IIS|8.5|Web Tier，Service Tier|
|4|ASP.Net MVC|5.2.3|Web Tier，Service Tier|
|5|.Net|4.6.1|Web Tier，Service Tier|
|6|HTTP.SYS||Service Tier|
|7|Redis||L1/L2 Cache & Pub/Sub|
|8|Websockets|||
|9|Elasticsearch||Search|
|10|SQL Server||Databases|

这里看到了的内容包括整体是基于.Net技术栈的。使用HAProxy进行的LB工作、使用IIS提供WEB动态服务、使用Redis做为一级缓存和二级缓存，而且还是用Redis做为消息队列使用、使用ElasticSearch作为搜索服务、SQL Server作为数据库存储使用。

* #### 访问量数据：
下面是Stack Overflow在2016年时一天的访问情况：
```
负载均衡器的HTTP请求数：209420973，其中66294789是页面加载(静态文件)
HTTP协议的发送流量1240266346053字节（1.24 TB）
共接收569449470023字节（569 GB）
总共发送3084303599266字节（3.08 TB）
SQL查询（仅来自HTTP请求）504816843次
5831683114次Redis点击量
17158874次弹性搜索
3661134标记引擎请求
607073066ms（168小时）用于运行SQL查询
10396073ms（2.8小时）用于Redis点击
147018571ms（40.8小时）用于标记引擎请求
花费在ASP.NET中的处理时间：1609944301ms（447小时）
49180275次问题页呈现平均22.71毫秒（ASP.NET中为19.12毫秒）
6370076次主页呈现的平均值11.80ms（ASP.NET为8.81 ms）
```

几个基本信息，请求的处理时间基本上要求在毫秒级处理结束，最多处理20毫秒。这基本上是行业规范了。

从负载上看到Redis服务器的CPU使用率基本上在2%左右，所以，可以看到Redis的数据处理速度还是非常快的。

## 分析数据
从上面看我的第一印象是谁说.Net技术栈不行的，而且惊讶于这里的技术人员的技术好厉害。使用.Net技术栈的技术，并组合很多现在业界流行的开源软件解决方案。不光有Java技术栈，Python技术栈、开源组件、还有服务治理技术栈的。所以说在任何技术栈下都可以很好的完成工作，不过是每个技术人对技术的理解和对业务的理解所造成的的。

* #### 带宽：
现在开始上重头戏，开始分析这些数据并给出一个初步结论。第一步我们分析一下网络带宽的要求：
```
每天接收569G数据，发送3.08T数据。
```
那每秒的上行的平均带宽为3.08T/24/60/60约等于37.4MB/s$\color{red}{约等于374Mbps}$。
那每秒的下行带宽为569G/24/60/60约等于6.7MB/s$\color{red}{约等于67Mbps}$。
Stack Overflow是一个技术问答类业务，所以在峰值与平均值得关系不会像其他业务系统一样有很大的差别。这里评估峰值的方式以平均值得5倍进行大概的评估。也就是峰值的上行带宽大概在$\color{red}{约等于1870Mbps}$，下行带宽的峰值$\color{red}{约等于335Mbps}$。

$\color{red}{在2016年时Stack Overflow网站提供服务的带宽大概是2205Mbps。}$一个网络服务产品，在进行服务时肯定有静态和动态部分。静态使用cdn完成，所以我们才可以在中国很快的访问到Stack Overflow服务。

* #### 运营和管理（容量数据）:
大概评估一下Stack Overflow的网络服务的运营和管理的数据：
```
负载均衡器的HTTP请求数：209420973，其中66294789是页面加载(静态文件)
```
这里以用户访问的量或者是用户喜欢Stack Overflow的程度来说明情况。这里使用PV、UV、会话数、用户数做一个Stack Overflow容量的大概容量。（因为Blog文章中没有提供网站上的Issue数所以不进行存储容量的评估）
PV：大概看了一下Stack Overflow的页面情况。大概加载一个页面会有12~20个静态资源需要加载，所以，简单的给出一个平均值18个请求加载一个页面。所以，66294789个静态页面加载请求/15$\color{red}{约等于3683043.8PV}$。
UV：一个用户在一天的时间内可能会访问5~20个独立页面。那么根据PV/8$\color{red}{约等于460380.5UV}$
用户数：每天都使用Stack Overflow服务的人肯定不是很多，所以用户量肯定是每天都放问Stack Overflow的数十倍。UV*20$\color{red}{约等于9207609.5个}$。

$\color{red}{在2016年的时候Stack Overflow大概有千万级的用户。}$Stack Overflow的内容数未知，每天都有46万人使用Stack Overflow。

* #### 性能：
并发说明网络服务提供服务的能力。性能也说明用户的体验是否好，并在这样的性能下保持持续提供服务的能力。
```
负载均衡器的HTTP请求数：209420973，其中66294789是页面加载(静态文件)
花费在ASP.NET中的处理时间：1609944301ms（447小时）
49180275次问题页呈现平均22.71ms（ASP.NET中为19.12ms）
6370076次主页呈现的平均值11.80ms（ASP.NET为8.81ms）
```
我发现这里的数据有一点对不上，所以，不考虑其他的数据是否可以对上。这里使用这几条数据对系统整体堆外性能进行评估。
会话数（同时在线用户数）：从数据上对不上，所以结果可能不能完全反应情况。所以这里就不进行会话数的评估。
每天动态数据请求数：209420973-66294789=143126184（可能还有其他请求，所以这个值比下面的值大的多。）
ASP.NET的请求数：   49180275 +6370076  =55550351
因为从原始中，第一行的数据可能包含其他。最后三条放在一起，所以数据的关联性比较好。所以这里使用后三条数据进行评估。
请求处理平均时间：ASP.NET的处理时间/(问题页次数+主页次数)$\color{red}{约等于28.98ms}$。
动态数据的并发数：每天动态数据请求/24/60/60$\color{red}{约等于2423.85并发}$。
峰值动态数据的并发量：上面说到大概是均值的5倍，所以$\color{red}{约等于12119.27并发}$。

这里的性能是一个局部数据评估的结果，并且不知道Stack Overflow的处理架构。所以，这块分析结果出入会比较大。但是，从前面的用户数或者活跃用户数对比的话还是比较可信的。评估的结果是：$\color{red}{请求平均处理时间为28.98ms，平均并发量为2423.85个/秒，峰值并发量为12119.27个/秒}$。

## 结果
从Nick Craver的博客中看到Stack Overflow是在不断的成长的。
2012年 Stack Overflow在搞各种各样的编码、环境的事情。
2013年 Stack Overflow环境基本上稳定了，在这个过程中升级了数据库，并转了Https
2015年 Stack Overflow的各种升级，软件升级，硬件升级
2016年 Stack Overflow做了技术体系建设的工作
2017年 Stack Overflow做了全面的Https
2018年 Stack Overflow开始高服务监控，提高系统服务稳定性了。

一步一步的Stack Overflow在慢慢的成长，从刚开始在搞代码，环境。到后来开始逐渐的想运维方向的转变。也是一个产品从发展到成熟的必经之路。这个过程可能用了6年、也可能用了更久（因为我看到Stack Overflow成立于2008年），所以一个产品的成长是需要漫长的时间进行磨砺的。

有人可以介绍认识一下硅谷的牛人吗？
