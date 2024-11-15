人们经常追求这些词汇，却没有清楚理解它们到底意味着什么。

> 互联网做得太棒了，以至于大多数人将它看作像太平洋这样的自然资源，而不是什么人工产物。上一次出现这种大规模且无差错的技术，你还记得是什么时候吗？
>
> —— [艾伦・凯](http://www.drdobbs.com/architecture-and-design/interview-with-alan-kay/240003442) 在接受 Dobb 博士杂志采访时说（2012 年）

---


现今很多应用程序都是 **数据密集型（data-intensive）** 的，而非 **计算密集型（compute-intensive）** 的。因此 CPU 很少成为这类应用的瓶颈，更大的问题通常来自数据量、数据复杂性、以及数据的变更速度。

数据密集型应用通常由标准组件构建而成，标准组件提供了很多通用的功能；例如，许多应用程序都需要：

 - 存储数据，以便自己或其他应用程序之后能再次找到 （*数据库，即 databases*）
 - 记住开销昂贵操作的结果，加快读取速度（*缓存，即 caches*）
 - 允许用户按关键字搜索数据，或以各种方式对数据进行过滤（*搜索索引，即 search indexes*）
 - 向其他进程发送消息，进行异步处理（*流处理，即 stream processing*）
 - 定期处理累积的大批量数据（*批处理，即 batch processing*）

## 可伸缩性

### 描述负载
推特的两个主要业务是：

* 发布推文

    用户可以向其粉丝发布新消息（平均 4.6k 请求 / 秒，峰值超过 12k 请求 / 秒）。

* 主页时间线

    用户可以查阅他们关注的人发布的推文（300k 请求 / 秒）。


大体上讲，这一对操作有两种实现方式。
1.当一个用户请求自己的主页时间线时，首先查找他关注的所有人，查询这些被关注用户发布的推文并按时间顺序合并

```sql
SELECT 
WHERE follows.follower_id = current_user
```

![](imgs/fig1-2.png)

2.为每个用户的主页时间线维护一个缓存，就像每个用户的推文收件箱（图 1-3）。当一个用户发布推文时，查找所有关注该用户的人，并将新的推文插入到每个主页时间线缓存中。因此读取主页时间线的请求开销很小，因为结果已经提前计算好了

因为发推频率比查询主页时间线的频率几乎低了两个数量级，所以在这种情况下，最好在写入时做更多的工作，而在读取时做更少的工作。
因为发推频率比查询主页时间线的频率几乎低了两个数量级，所以在这种情况下，最好在写入时做更多的工作，而在读取时做更少的工作。



### 描述性能

Hadoop 吞吐量（throughput）
响应时间（response time），即客户端发送请求到接收响应之间的时间

> #### 延迟和响应时间
>
> 延迟（latency） 和 响应时间（response time） 经常用作同义词，但实际上它们并不一样。响应时间是客户所看到的，除了实际处理请求的时间（ 服务时间（service time） ）之外，还包括网络延迟和排队延迟。延迟是某个请求等待处理的 持续时长，在此期间它处于 休眠（latent） 状态，并等待服务【17】。

即使不断重复发送同样的请求，每次得到的响应时间也都会略有不同。现实世界的系统会处理各式各样的请求，响应时间可能会有很大差异。因此我们需要将响应时间视为一个可以测量的数值 分布（distribution），而不是单个数值。

在 [图 1-4](imgs/fig1-4.png) 中，每个灰条代表一次对服务的请求，其高度表示请求花费了多长时间


![](imgs/fig1-4.png)

**图 1-4 展示了一个服务 100 次请求响应时间的均值与百分位数**

通常报表都会展示服务的平均响应时间。（严格来讲 “平均” 一词并不指代任何特定公式，但实际上它通常被理解为 算术平均值（arithmetic mean）：给定 n 个值，加起来除以 n ）。然而如果你想知道 “典型（typical）” 响应时间，那么平均值并不是一个非常好的指标，因为它不能告诉你有多少用户实际上经历了这个延迟。

通常使用 百分位点（percentiles） 会更好。如果将响应时间列表按最快到最慢排序，那么 中位数（median） 就在正中间：举个例子，如果你的响应时间中位数是 200 毫秒，这意味着一半请求的返回时间少于 200 毫秒，另一半比这个要长。


如果想知道典型场景下用户需要等待多长时间，那么中位数是一个好的度量标准：一半用户请求的响应时间少于响应时间的中位数，另一半服务时间比中位数长。中位数也被称为第 50 百分位点，有时缩写为 p50。注意中位数是关于单个请求的；如果用户同时发出几个请求（在一个会话过程中，或者由于一个页面中包含了多个资源），则至少一个请求比中位数慢的概率远大于 50%。

为了弄清异常值有多糟糕，可以看看更高的百分位点，例如第 95、99 和 99.9 百分位点（缩写为 p95，p99 和 p999）。它们意味着 95%、99% 或 99.9% 的请求响应时间要比该阈值快，例如：如果第 95 百分位点响应时间是 1.5 秒，则意味着 100 个请求中的 95 个响应时间快于 1.5 秒，而 100 个请求中的 5 个响应时间超过 1.5 秒。

响应时间的高百分位点（也称为 尾部延迟，即 tail latencies）非常重要，因为它们直接影响用户的服务体验。例如亚马逊在描述内部服务的响应时间要求时是以 99.9 百分位点为准，即使它只影响一千个请求中的一个。这是因为请求响应最慢的客户往往也是数据最多的客户，也可以说是最有价值的客户 —— 因为他们掏钱了[^19]。保证网站响应迅速对于保持客户的满意度非常重要，亚马逊观察到：响应时间增加 100 毫秒，销售量就减少 1%【20】；而另一些报告说：慢 1 秒钟会让客户满意度指标减少 16%【21，22】

另一方面，优化第 99.99 百分位点（一万个请求中最慢的一个）被认为太昂贵了，不能为亚马逊的目标带来足够好处。减小高百分位点处的响应时间相当困难，因为它很容易受到随机事件的影响，这超出了控制范围，而且效益也很小。

服务级别协议（SLA, service level agreements，即定义服务预期性能和可用性的合同。SLA 可能会声明，如果服务响应时间的中位数小于 200 毫秒，且 99.9 百分位点低于 1 秒，则认为服务工作正常（如果响应时间更长，就认为服务不达标）。这些指标为客户设定了期望值，并允许客户在 SLA 未达标的情况下要求退款。

排队延迟（queueing delay） 通常占了高百分位点处响应时间的很大一部分。由于服务器只能并行处理少量的事务（如受其 CPU 核数的限制），所以只要有少量缓慢的请求就能阻碍后续请求的处理，这种效应有时被称为 头部阻塞（head-of-line blocking） 。即使后续请求在服务器上处理的非常迅速，由于需要等待先前请求完成，客户端最终看到的是缓慢的总体响应时间。因为存在这种效应，测量客户端的响应时间非常重要

### 应对负载的方法



## 参考资料


[^19]: Giuseppe DeCandia, Deniz Hastorun, Madan Jampani, et al.: “[Dynamo: Amazon's Highly Available Key-Value Store](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf),” at *21st ACM Symposium on Operating Systems Principles* (SOSP), October 2007.
[^19]: 