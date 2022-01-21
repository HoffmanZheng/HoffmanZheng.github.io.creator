---
title: "分布式：[译] 基于 Redis 的分布式锁 "
author: "Chenghao Zheng"
tags: ["Distributed"]
categories: ["Reading notes"]
date: 2022-01-19T03:19:47+01:00
draft: false
---

原文地址：[Distributed locks with Redis](https://redis.io/topics/distlock)

当不同的进程必须以互斥的方式操作共享的资源时，分布式锁是一个非常实用的原始功能。

有很多组件库和博客都描述了如何去实现一个基于 Redis 的分布式锁管理器，但是每个库使用的实现方式不尽相同，其中许多库使用了一种简单的方式，它相比稍微复杂的设计只能提供更少的保障。

本篇尝试提供一种更加规范的算法来实现基于 Redis 的分布式锁。我们提出了一种叫做 `Redlock` 的算法，它可以实现一个分布式锁管理器（我们相信它比普通的单实例方法更加安全）。我们希望社区将分析它，提供反馈，并且将它作为更复杂/替代设计的基础。

### 实现

在描述这个算法之前，现在已经有一些可用的实现链接供参考：  

- [Redlock-rb](https://github.com/antirez/redlock-rb) (Ruby 实现). 还有一个 [Redlock-rb](https://github.com/leandromoreira/redlock-rb) 的分支，它为了方便分发新增了一个 gem 包，也许还有其他的。
- [Redlock-py](https://github.com/SPSCommerce/redlock-py) (Python 实现).
- [Pottery](https://github.com/brainix/pottery#redlock) (Python 实现).
- [Aioredlock](https://github.com/joanvila/aioredlock) (Asyncio Python 实现).
- [Redlock-php](https://github.com/ronnylt/redlock-php) (PHP 实现).
- [PHPRedisMutex](https://github.com/malkusch/lock#phpredismutex) (further PHP 实现)
- [cheprasov/php-redis-lock](https://github.com/cheprasov/php-redis-lock) (PHP 锁的库)
- [rtckit/react-redlock](https://github.com/rtckit/reactphp-redlock) (Async PHP 实现)
- [Redsync](https://github.com/go-redsync/redsync) (Go 实现).
- [Redisson](https://github.com/mrniko/redisson) (Java 实现).
- [Redis::DistLock](https://github.com/sbertrang/redis-distlock) (Perl 实现).
- [Redlock-cpp](https://github.com/jacket-code/redlock-cpp) (C++ 实现).
- [Redlock-cs](https://github.com/kidfashion/redlock-cs) (C#/.NET 实现).
- [RedLock.net](https://github.com/samcook/RedLock.net) (C#/.NET 实现). 包含异步和锁的扩展支持
- [ScarletLock](https://github.com/psibernetic/scarletlock) (C# .NET 实现 可配置数据源)
- [Redlock4Net](https://github.com/LiZhenNet/Redlock4Net) (C# .NET 实现)
- [node-redlock](https://github.com/mike-marcacci/node-redlock) (NodeJS 实现). 包含锁扩展的支持

### 安全和可用保障

我们将使用三个特性模型化我们的设计，在我们看来，它们是高效地使用分布式锁的最低保障。

1. 安全性：互斥，任一时刻只有一个客户端能够持有锁
2. 可用性 A：无死锁，即使锁住资源的客户端崩溃了或者被隔离分区，（其他客户端）也要能够获取锁
3. 可用性 B：容错，只要大部分 Redis 节点存活，客户端就能够获取和释放锁

### 基于故障转移的实现的不足

为了理解我们想要改善的东西，让我们先分析下大多数基于 Redis 分布式锁的库的现状。

使用 Redis 锁住一个资源最简单的方式是在实例中创建一个 key。利用 Redis 的过期特性，这个 key 会在一个有限的时间内存活，所以它最终会被释放（特性二）。当客户端想要释放资源的时候，它会删除这个 key。

表面上这样可以正常工作，但是存在一个问题：系统的单点故障。如果 Redis 的主节点宕机会发生什么？当然我们可以增加一个从节点，并且在主节点不可用时使用它。不幸的是，因为 Redis 的复制是 **异步** 的，这样做我们无法保证互斥的安全性。

这个模型存在一个明显的竞态条件：

1. 客户端 A 获取主节点的锁
2. 主节点在将 key 传播给从节点之前宕机
3. 从节点被提升为主节点
4. 客户端 B 获取之前被 A 锁住的资源对应的锁，**违反了安全规定**！

如果多个客户端在特殊情况下，比如故障期间，同时持有一个锁是完全可以的。那么你可以使用基于从节点的解决方案。否则我们建议实施本文描述的解决方案。

### 单实例的正确实现

在突破上文描述的单实例限制之前，让我们看看在这个简单的情况下该如何正确地做。因为这对于可以接受时不时出现竞态条件的应用实际上是一个可行的解决方案，并且着眼于单实例也是我们讨论分布式算法的基础。

下述是一个获取锁的方法：

```shell
SET resource_name my_random_value NX PX 30000
```

这个命令将会在 key 不存在时（NX 选项）设置这个 key，伴随着 30000 毫秒的过期时间（PX 选项）。这个键的值将被设置为随机值，并且在所有的客户端和锁请求中保持唯一性。

本质上随机值是用来安全地释放锁的，伴随着脚本告诉 Redis：只有当键存在并且它的值与存储的值相符时才移除这个键。通过下述的 Lua 脚本来实现： 

```shell
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

避免移除一个被其他客户端创建的锁是重要的。比如一个客户端可能获取锁，被阻塞的时长超过锁的有效（过期）时长，随后清除这些已经被其他客户端获取的锁。只使用 `DEL` 是 **不安全** 的，客户端可能清除其他客户端的锁。用上文的脚本并且每个锁都用一个随机字符串签名，这样锁就只能被当初设置它的客户端清除。

这个随机字符串应该是什么？我假设它是从 /dev/urandom 来的 20 字符，但是你可以用更高效的方式来保证它对于你的任务的唯一性。比如一个安全的选择是使用 /dev/urandom 作为 RC4 的中止，从中生成一个伪随机的流。一个更简单的解决方案是使用微秒级的 unix  时间，附加上客户端 ID，这不是那么安全，但可能在大多数环境中都可以胜任。

我们设置的 key 的存活时间，被称作锁的有效时间。它既是自动释放的时间，也是客户端在锁被别的客户端再次获取之前，（保证技术上不违反互斥性）去执行所需操作的时间。互斥保证 **仅限于** 从锁的那一刻开始的这个给定的时间窗口。

因此现在我们有了一个好的方式去获取和释放锁。这个由始终可用的单实例组成的非分布式系统被论证为是安全的。让我们将这个概念扩展到一个没有此类保障的分布式系统。

### Redlock 算法

在算法的分布式版本中，我们假设有 N 个 Redis 主节点。这些节点完全独立，我们并没有使用复制或者其他的辅助系统。我们已经描述过如何在单个实例中安全地获取和释放锁。我们也理所当然地认为算法会使用那个方法在单个实例中获取和释放锁。在我们的例子中，我们设置 N = 5，它是一个合理的值，因此我们需要在不同的计算器或虚拟机上运行 5 个 Redis 主节点，

### 这个算法是异步的吗

### 错误重试

### 安全参数

### 可用性参数

### 性能、崩溃恢复和同步

### 更加可靠的算法：锁的扩展



