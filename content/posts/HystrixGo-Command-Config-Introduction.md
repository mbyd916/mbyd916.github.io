---
title: "详解 hystrix-go Command 配置参数"
date: 2019-12-30T21:16:54+08:00
draft: false
categories:
- 基础组件
- 备忘
tags:
- "Hystrix 配置"
- 熔断降级
keywords:
- "详解 hystrix-go Command 配置参数"
---

hystrix 是 Netflix 开源的一个非常强大的熔断降级组件， 用 Java 实现， 而 hystrix-go 是其 Go 版本的一个实现。本文主要介绍 Command 配置涉及的 5 个参数。

```CommandConfig``` 结构体如下：

```golang
type CommandConfig struct {
  Timeout                int `json:"timeout"`
  MaxConcurrentRequests  int `json:"max_concurrent_requests"`
  RequestVolumeThreshold int `json:"request_volume_threshold"`
  SleepWindow            int `json:"sleep_window"`
  ErrorPercentThreshold  int `json:"error_percent_threshold"`
}
```

有 5 个参数可供配置，下面介绍他们的具体含义：

1. Timeout: 设置调用方执行超时，默认设置 1000 ms。超时后，hystrix 执行降级处理。

2. MaxConcurrentRequests: 设置调用方最大并发限制，默认值是 10。超过后，hystrix 执行降级处理。

3. RequestVolumeThreshold：在滑动窗口的采样时间内，请求次数达到这个阈值后，才开始判断是否熔断，默认值是 20。举例来说，如果该值是 20，在 10s 的滑动窗口期收到了 19 个请求，即使全都失败了，也不会触发熔断。

4. SleepWindow：休眠窗口期，默认设置 5000ms。熔断后等待试探（是否关闭熔断器）的时间。

5. ErrorPercentThreshold：错误率阈值，默认是 50。超过后，会启用熔断和降级处理。

按上述 Command 的默认配置，熔断器生效的条件是：

在 10s 内，至少需要接收到 20 个请求（RequestVolumeThreshold），并且有一定比例（ErrorPercentThreshold）的请求处理失败了，这样熔断器才会被打开。5s 后（SleepWindow），处理第 1 个请求，如果成功，熔断器将会关闭，开始正常对外提供服务。

### 参考

1. [Hystrix Configuration 官方文档](https://github.com/Netflix/Hystrix/wiki/Configuration)

2. [Hystrix Circuit Breaker — How To Set It Up Properly](https://medium.com/@darek1024/hystrix-circuit-breaker-how-to-set-it-up-properly-84c75cfbe3ee)
