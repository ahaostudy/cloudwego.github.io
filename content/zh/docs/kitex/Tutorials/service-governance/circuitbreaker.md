---
title: "熔断"
date: 2023-10-24
weight: 6
keywords: ["Kitex","熔断","熔断器"]
description: "Kitex 熔断使用指南、原理介绍。"
---

## 介绍

Kitex 提供了熔断器的实现，但是没有默认开启，需要用户主动使用。下面简单介绍一下如何使用以及 Kitex 熔断器的策略。

## 使用方式

### 使用示例：

```go
import (
        ...
        "github.com/cloudwego/kitex/client"
        "github.com/cloudwego/kitex/pkg/circuitbreak"
        "github.com/cloudwego/kitex/pkg/rpcinfo"
)

// GenServiceCBKeyFunc returns a key which determines the granularity of the CBSuite
func GenServiceCBKeyFunc(ri rpcinfo.RPCInfo) string {
        // circuitbreak.RPCInfo2Key returns "$fromServiceName/$toServiceName/$method"
        return circuitbreak.RPCInfo2Key(ri)
}

func main() {
        // build a new CBSuite with 
        cbs := circuitbreak.NewCBSuite(GenServiceCBKeyFunc)

        var opts []client.Option
        
        // add to the client options
        opts = append(opts, client.WithCircuitBreaker(cbs))
        
        // init client 
        cli, err := echoservice.NewClient(targetService, opts...)
        
        // update circuit breaker config for a certain key (should be consistent with GenServiceCBKeyFunc)
        // this can be called at any time, and will take effect for following requests
        cbs.UpdateServiceCBConfig("fromServiceName/toServiceName/method", circuitbreak.CBConfig{
                Enable: true,
                ErrRate: 0.3,   // requests will be blocked if error rate >= 30%
                MinSample: 200, // this config takes effect if sampled requests are more than `MinSample`
        })

        // send requests with the client above
        ...
}

```

### 使用说明

Kitex 大部分服务治理模块都是通过 middleware 集成，熔断也是一样。Kitex 提供了一套 CBSuite，封装了服务粒度的熔断器和实例粒度的熔断器。

- 服务粒度熔断

  - 按照服务粒度进行熔断统计，通过 WithMiddleware 添加。服务粒度的具体划分取决于 Circuit Breaker Key，既熔断统计的 key，初始化 CBSuite 时需要传入 **GenServiceCBKeyFunc**，默认提供的是 circuitbreak.RPCInfo2Key ，该 key 的格式是 `fromServiceName/toServiceName/method`，即按照方法级别的异常做熔断统计。
- 实例粒度熔断

  - 按照实例粒度进行熔断统计，主要用于解决单实例异常问题，如果触发了实例级别熔断，框架会自动重试。
  - 注意，框架自动重试的前提是需要通过 **WithInstanceMW** 添加，WithInstanceMW 添加的 middleware 会在负载均衡后执行。
- 熔断阈值及**阈值变更**

  - 默认的熔断阈值是 `ErrRate: 0.5, MinSample: 200`，错误率达到 50% 触发熔断，同时要求统计量 >200。若要调整阈值，调用 CBSuite 的 `UpdateServiceCBConfig` 和 `UpdateInstanceCBConfig` 来更新 Key 的阈值。

## 熔断器作用

在进行 RPC 调用时，下游服务难免会出错；

当下游出现问题时，如果上游继续对其进行调用，既妨碍了下游的恢复，也浪费了上游的资源；

为了解决这个问题，你可以设置一些动态开关，当下游出错时，手动的关闭对下游的调用；

然而更好的办法是使用熔断器，自动化的解决这个问题。

这里是一篇更详细的[熔断器介绍](https://msdn.microsoft.com/zh-cn/library/dn589784.aspx)。

比较出名的熔断器当属 hystrix 了，这里是它的[设计文档](https://github.com/Netflix/Hystrix/wiki)。

### 熔断策略

**熔断器的思路很简单：根据****RPC****的成功失败情况，限制对下游的访问；**

通常熔断器分为三个时期： CLOSED、OPEN、HALFOPEN；

RPC 正常时，为 CLOSED；

当 RPC 错误增多时，熔断器会被触发，进入 OPEN；

OPEN 后经过一定的冷却时间，熔断器变为 HALFOPEN；

HALFOPEN 时会对下游进行一些有策略的访问，然后根据结果决定是变为 CLOSED，还是 OPEN；

总的来说三个状态的转换大致如下图：

```
 [CLOSED] ---> tripped ----> [OPEN]<-------+
    ^                          |           ^
    |                          v           |
    +                          |      detect fail
    |                          |           |
    |                    cooling timeout   |
    ^                          |           ^
    |                          v           |
    +--- detect succeed --<-[HALFOPEN]-->--+

```

### 触发策略

Kitex 默认提供了三个基本的熔断触发策略：

- 连续错误数达到阈值 (ConsecutiveTripFunc)
- 错误数达到阈值 (ThresholdTripFunc)
- 错误率达到阈值 (RateTripFunc)

当然，你可以通过实现 TripFunc 函数来写自己的熔断触发策略；

Circuitbreaker 会在每次 Fail 或者 Timeout 时，去调用 TripFunc，来决定是否触发熔断；

### 冷却策略

进入 OPEN 状态后，熔断器会冷却一段时间，默认是 10 秒，当然该参数可配置 (CoolingTimeout)；

在这段时期内，所有的 IsAllowed() 请求将会被返回 false；

冷却完毕后进入 HALFOPEN；

### 半打开时策略

在 HALFOPEN 时，熔断器每隔 " 一段时间 " 便会放过一个请求，当连续成功 " 若干数目 " 的请求后，熔断器将变为 CLOSED； 如果其中有任意一个失败，则将变为 OPEN；

该过程是一个逐渐试探下游，并打开的过程；

上述的 " 一段时间 “(DetectTimeout) 和 " 若干数目 “(DEFAULT_HALFOPEN_SUCCESSES) 都是可以配置的；

## 统计算法

### 默认参数

熔断器会统计一段时间窗口内的成功，失败和超时，默认窗口大小是 10S；

时间窗口可以通过两个参数设置，不过通常情况下你可以不用关心 .

### 统计方法

统计方法是将该段时间窗口分为若干个桶，每个桶记录一定固定时长内的数据；

比如统计 10 秒内的数据，于是可以将 10 秒的时间段分散到 100 个桶，每个桶统计 100ms 时间段内的数据；

Options 中的 BucketTime 和 BucketNums，就分别对应了每个桶维护的时间段，和桶的个数；

如将 BucketTime 设置为 100ms，将 BucketNums 设置为 100，则对应了 10 秒的时间窗口；

### 抖动

随着时间的移动，窗口内最老的那个桶会过期，当最后那个桶过期时，则会出现了抖动；

举个例子：

- 你将 10 秒分为了 10 个桶，0 号桶对应了 [0S，1S) 的时间，1 号桶对应 [1S，2S)，…，9 号桶对应 [9S，10S)；
- 在 10.1S 时，执行一次 Succ，则 circuitbreaker 内会发生下述的操作；

  - (1) 检测到 0 号桶已经过期，将其丢弃；
  - (2) 创建新的 10 号桶，对应 [10S，11S)；
  - (3) 将该次 Succ 放入 10 号桶内；
- 在 10.2S 时，你执行 Successes() 查询窗口内成功数，则你得到的实际统计值是 [1S，10.2S) 的数据，而不是 [0.2S，10.2S)；

如果使用分桶计数的办法，这样的抖动是无法避免的，比较折中的一个办法是将桶的个数增多，可以降低抖动的影响；

如划分 2000 个桶，则抖动对整体的数据的影响最多也就 1/2000； 在该包中，默认的桶个数也是 2000，桶时间为 5ms，总体窗口为 10S；

当时曾想过多种技术办法来避免这种问题，但是都会引入更多其他的问题，如果你有好的思路，请 issue 或者 PR.
