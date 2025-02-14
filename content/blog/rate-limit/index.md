---
title: 详解几种常见限流算法及限流器设计
date: 2025-02-01T10:30:31+08:00
tags: ["限流", "令牌桶"]
series: []
featured: true
---

在服务端开发中，限流（Rate Limiting）是一种常见的技术，用于控制请求的流量，防止请求过多导致服务器压力过大或响应时间过长。限流算法有很多种，本文将介绍主要的几种常见限流算法及限流器设计。

<!--more-->

## 限流场景

在实际开发过程中，我们需要考虑一些需要限流的场景。比如防止突发流量激增导致服务崩溃，或者防止恶意请求导致服务器资源占用过多，又或者某些业务场景下我们需要限制用户的请求频率。

衡量请求频率的指标主要有两种：

TPS: 每秒事务数（Transaction Per Second）

TPS 代表每秒事务数，是衡量系统处理能力的一个指标。事务通常指的是一组操作的集合，可能包括多个数据库操作、消息队列处理等。在许多情况下，TPS 用于衡量数据库系统或应用程序在特定负载下的性能。

QPS: 每秒查询数（Query Per Second）

QPS 代表每秒查询数，是衡量系统查询能力的一个指标。查询通常指的是数据库查询操作，包括 SELECT、INSERT、UPDATE、DELETE 等。在许多情况下，QPS 用于衡量数据库系统或应用程序的查询性能。

在某些系统中，一个事务可能包含多个查询，因此在这些情况下，TPS 和 QPS 之间可以有一定的关系。例如，一个事务包含三个查询，那么在理想情况下，QPS 可能是 TPS 的三倍。

## 限流算法

### 基于计数器的限流

基于计数器的限流算法理解起来很容易。我们只需要维护一个计数器，然后每一分钟的开始重置计数器为 0，然后在这一分钟内如果计数器超过了某个阈值，那么对于超出阈值的请求直接拒绝，这样就实现了基于计数器的限流。

![计数器限流](image/image.png)

实现这样的限流很简单，但是问题也很明显：有时候请求可能会在前一秒的末尾和后一秒的开始突然出现某个阈值的流量，而根据我们队限流算法的设计，这样的请求是被允许的。但是这样的话对于 0.9s 到 1.1s 之间的流量就超过了我们设定的阈值，这样导致的瞬时流量也会对服务造成压力。所以有了滑动窗口算法。

![计数器限流问题](image/image-1.png)

### 基于滑动窗口的限流

基于滑动窗口的限流算法是基于计数器的限流算法的改进。它将限流的时间窗口分为多个小窗口，然后在每个小窗口内独立维护一个计数器，当计数器超过了某个阈值，那么对于超出阈值的请求直接拒绝。随着时间的推移，每次都会排除掉过去最近的一个小窗口，然后引入最新的一个小窗口。相比于基于计数器的限流，基于滑动窗口的限流算法每次推移的单位是一个小窗口区间，而不是一个大时间区间，这样就实现了更精确的时间片限流。

![滑动窗口限流](image/image-2.png)

但是即是通过滑动窗口算法人为将时间分成了比较小的片，也依然有可能在时间区间内发生突发且不均匀的流量，可能压垮服务。但是如果小将片划分的足够小，又可能会带来系统的损耗和性能的问题。

### 漏桶算法

漏桶算法的实现原理类似于缓冲。不管请求是怎么来的，都直接放到桶里面。然后桶里面的请求以固定的速度流出。如果流量过多或者过大超出了桶的容量，那么就丢弃一些请求，桶里面的请求以固定的速度流出。

![漏桶算法](image/image-3.png)

最常见的模拟漏桶算法的方式就是 FIFO 的消息队列。请求会被放到队列中，等待被服务消费。

漏桶算法可以实现流量的平滑处理，但是如果突然来了很多流量，那么桶一下就被填满了。由于流出的速率是恒定的，所以当短时间有大量的突发请求到了桶里面，那么每个请求也得等一段时间才能被相应。所以漏桶算法无法应对突发流量。

### 令牌桶算法

相较于漏桶算法，令牌桶算法可以更好的处理突发流量。每到一个请求，必须先去令牌桶里面申请令牌。申请到的令牌才会被服务器处理，否则就会拒绝。

![令牌桶算法](image/image-4.png)

在请求较低的时候，令牌数量会被慢慢增加，一直到达到桶的最大容量。在遇到突发流量时，令牌桶内会有一定的令牌数。新到达的请求会申请这些令牌且不会被拒绝，这样就有了当面对突发流量时的应对能力。

令牌桶算法当然可以基于队列实现，这样比较直观。我们也可以直接用时间的方式去实现，这样就不用在内存中维护队列了。

两个简单的获取令牌的实现示例：

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class TokenBucketRateLimiter {
    private final int bucketCapacity; // 令牌桶的容量
    private final int refillRate; // 令牌生成速率（每秒生成的令牌数）
    private AtomicInteger tokens; // 当前令牌数

    public TokenBucketRateLimiter(int bucketCapacity, int refillRate) {
        this.bucketCapacity = bucketCapacity;
        this.refillRate = refillRate;
        this.tokens = new AtomicInteger(0);
        startRefilling(); // 启动定期生成令牌的线程
    }

    // 尝试获取一个令牌
    public boolean tryAcquire() {
        int currentTokens = tokens.get();
        if (currentTokens > 0) {
            if (tokens.compareAndSet(currentTokens, currentTokens - 1)) {
                System.out.println("请求获得令牌，允许通过");
                return true;
            }
        }
        System.out.println("请求被拒绝：无可用令牌");
        return false;
    }

    // 定期为令牌桶添加令牌
    private void startRefilling() {
        ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);
        executor.scheduleAtFixedRate(() -> {
            int currentTokens = tokens.get();
            if (currentTokens < bucketCapacity) {
                int newTokens = Math.min(bucketCapacity, currentTokens + refillRate);
                tokens.set(newTokens);
                System.out.println("新增令牌，当前令牌数：" + newTokens);
            }
        }, 0, 1, TimeUnit.SECONDS); // 每秒填充令牌
    }

    public static void main(String[] args) {
        TokenBucketRateLimiter rateLimiter = new TokenBucketRateLimiter(10, 2); // 容量为10，每秒生成2个令牌

        // 模拟10个请求
        for (int i = 0; i < 10; i++) {
            boolean allowed = rateLimiter.tryAcquire();
            if (!allowed) {
                System.out.println("请求 " + i + " 被拒绝");
            }
            try {
                Thread.sleep(200); // 模拟请求间隔
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

```go
package ratelimit

import (
  "fmt"
  "sync"
  "time"
 )

 type RateLimiter struct {
  rate   int64 // 令牌放入速度
  max    int64 // 令牌最大数量
  last   int64 // 上一次请求发生时间
  amount int64 // 令牌数量
  lock   sync.Mutex // 由于读写冲突，需要加锁
 }

    // 获得当前时间
 func cur() int64 {
  return time.Now().Unix()
 }

 func New(rate int64, max int64) *RateLimiter {
  // TODO: 检查一下rate和max是否合法
  return &RateLimiter{
   rate:   rate,
   max:    max,
   last:   cur(),
   amount: max,
  }
 }

 func (rl *RateLimiter) Pass() bool {
  rl.lock.Lock()
  defer rl.lock.Unlock()

        // 距离上一次请求过去的时间
  passed := cur() - rl.last
  fmt.Println("passed is: ", passed)

        // 计算在这段时间里 令牌数量可以增加多少
  amount := rl.amount + passed*rl.rate

        // 如果令牌数量超过上限；我们就不继续放入那么多令牌了
  if amount > rl.max {
   amount = rl.max
  }

        // 如果令牌数量仍然小于0，则说明请求应该拒绝
  if amount <= 0 {
   return false
  }

        // 请求被放行则令牌数-1
  amount--
  rl.amount = amount
        // 更新上次请求时间
  rl.last = cur()

  return true
 }
```

## 限流器设计

如果我们要设计一个限流器，通常要考虑下面几个方面资源的限流：

1. 全局接口限流
2. 用户的账号限流
3. 某请求限流
4. 设备限流

当然具体需要针对哪部分进行限流，还应该取决于具体的业务需求。

限流器通常位于网关的部分，属于网关的一个过滤器，通过责任链模式进行校验后转发到后端服务。

限流的模式有本地限流和远程限流两种：本地限流和远程限流。

本地限流就是每个网关服务器自身进行限流。网关直接亦为集群负载均衡部署，假设每个网关限流为 100 的 QPS，那么 10 个网关集群的 QPS 大约为 1000。

远程限流就是通过远程服务进行限流，比如通过配置中心或者 Redis 进行限流。远程限流亦需要考虑 Redis 的可用性和集群数据一致性。

同时我们还需要考虑限流的高可用性：当远程限流不可用时，亦需要降级为本地限流。

除此以外我们就需要考虑限流算法的选用。一般企业界选用的限流算法为令牌桶算法，即能够应对突发流量，也能够平滑流量。

## Guava 中限流的实现

很多著名的限流项目如 Resukuebce4j、Sentinel 以及某些大厂自研限流组件都参考了 Guava 的限流实现。而 Guava 的限流算法采用的就是令牌桶算法。一个示例 demo 如下：

```java
import com.google.common.util.concurrent.RateLimiter;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class RateLimiterExample {

    public static void main(String[] args) {
        // 创建一个RateLimiter，每秒允许处理5个请求
        RateLimiter rateLimiter = RateLimiter.create(5.0);

        // 创建一个固定大小的线程池
        ExecutorService executorService = Executors.newFixedThreadPool(10);

        // 模拟处理20个请求
        for (int i = 0; i < 20; i++) {
            final int requestNumber = i;
            executorService.submit(() -> handleRequest(requestNumber, rateLimiter));
        }

        // 关闭线程池
        executorService.shutdown();
    }

    private static void handleRequest(int requestNumber, RateLimiter rateLimiter) {
        // 获取一个令牌，可能会阻塞直到令牌可用
        rateLimiter.acquire();

        // 模拟处理请求的逻辑
        System.out.println("Handling request " + requestNumber + " at " + System.currentTimeMillis());
    }
}
```

在 Guava 内部为了优化空间，并没有采用设计一个桶，而是采用了时间比较的方式计算令牌数量，从而实现限流的功能。

Guava 的限流器有突发模式（SmoothBursty）和预热模式（SmoothWarmingUp）两种。突发模式令牌以稳定的速率生成，。在预热模式下，限流器发放令牌的速度会随着流量的上涨逐渐加快，从而让服务端平滑从空闲状态过度到高负载状态。（类似于 TCP 的拥塞控制算法的思想）

## 自适应限流

自适应限流是指根据当前的请求量和系统的负载情况，动态调整限流策略。比如在高峰期，可以适当提高限流阈值，在低谷期，可以适当降低限流阈值，从而达到平衡。

阿里的 AHAS 自适应监控、[蚂蚁的 MOSNzk 自适应限流](https://www.sofastack.tech/blog/service-mesh-exploration-thinking-after-1111-1/)以 CPU 利用率为指标，使用 PID 控制器进行自适应限流。而[Sentinel](https://github.com/alibaba/Sentinel/wiki/%E7%B3%BB%E7%BB%9F%E8%87%AA%E9%80%82%E5%BA%94%E9%99%90%E6%B5%81)这种则采用 CPU 负载、QPS、RT、并发请求等指标，设定并发控制算法，计算服务请求的最大并发数，并根据当前的请求量进行限流。

## 拓展延伸

其实限流保护的策略不仅仅在后端服务上，有时候在产品本身的设计上也可以考虑。比如 12306 在抢票购票设计时，会将不同车次的购买时间分隔开，以及提前一定时间前购票。从产品的角度，避免了某一个时间突然的高并发，反而将流量相对的均匀开，也是一种不错的在流量方面的保护策略。

并且 12306 在流量策略上，还采用了异步交易排队系统的方案。就是将 12306 网站设计成一个大超市，根据访问量变化开启多个“收银台”，将每个购票者分配到不同的“收银台”前排队，以减少拥堵。其内部猜测应该是采用了某种服务调度的策略，根据请求量大小进行了负载均衡，从而减少了流量单个堆积。

## 参考文献

- [【java 面试加分项】12306 超高并发如何保证数据一致性？](https://www.bilibili.com/list/watchlater?oid=113793128465727&bvid=BV1NTrvYnEbM&spm_id_from=333.1007.top_right_bar_window_view_later.content.click)
- [Java 实现令牌桶算法：详细讲解与代码示例](https://blog.csdn.net/qq_42055933/article/details/143694116)
- [极客时间令牌桶算法代码](https://github.com/wfnuser/Algorithms)
