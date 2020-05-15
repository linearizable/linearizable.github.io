---
layout: post
title: "Handling Overload With Adaptive Concurrency Control - Part 3"
author:
- Vikas Kumar
---

<div class='bannercontainer'><img src='/assets/proxy.png' /></div>

The [part 2](/2020/05/15/adaptive-concurrency-control-algorithms-part-2.html) of the series, we discussed how the [concurrency limits library](https://github.com/Netflix/concurrency-limits) integrates with Spring Boot applications and explored various algorithms used by this library. In this article, we will look at how concurrency limiting works in practice and discuss issues with a library based approach. Finally we'll look at an alternative approach which uses a proxy in front of the application to control its concurrency.

# Concurrency Limits Library In Action

In the [first article](/2020/05/15/adaptive-concurrency-control-algorithms-part-2.html) of the series, we discussed how a Spring Boot application with Tomcat web server works. When doing an adaptive concurrency control, we usually start with an initial concurrency value and then keep changing this value based on system behaviour. Let's define:

* `CL`: Current concurrency limit
* `W` : Maximum number of worker threads (Let's use default value of 200)

There are 2 cases to consider:

##### CL &nbsp;<span class='symbol'>&#8805;</span>&nbsp; W

In this case, no request will be rejected. Tomcat will send 200 requests to the 200 worker threads and while the worker threads are processing, Tomcat will keep additional requests in the queue (up to <code class="highlighter-rouge">max-connections</code>).

One important thing to note is that increasing limit beyond `W` does not increase the service capacity. Service capacity is defined by `W` and can not go beyond that. In that sense, the concurrency limit of 1000 is essentially same as 200.

> ###### Why Increase Limit Beyond W?
>
> An interesting question is then why should we increase CL beyond W, if it doesn't change anything?
>
> As we'll see below, the library will start rejecting requests as soon as CL < W. Increasing limit beyond W can provide some level of protection against temporary overload. If CL is high (>> W) and the service starts experiencing load, the library will start decreasing CL, but will not reject requests unless CL < W. The service might recover before that condition is met. So the service will be able to handle temporary issues without rejecting requests.
>
>The speed with which CL decreases depends on the algorithm and parameters used.

##### CL &nbsp;<span class='symbol2'>&lt;</span>&nbsp; W

In this case, the maximum in-flight requests (i.e. requests being worked upon by worker threads) can not exceed `CL`. That means, at any time, only `CL` number of worker threads will be busy processing requests. The rest (`W - CL`) will just be rejecting requests.

The overall result of this is that no queuing will happen in Tomcat as the worker threads rejecting requests will become free again very quickly and Tomcat will keep sending more requests to them, only to be rejected. Also a limit of 199 is essentially the same as 100 in terms of rejecting requests, since one thread can roughly reject requests at the same rate as 100 threads.

> ###### Why Decrease Limit Below W-1?
>
> Again, an interesting question. For W = 200, a limit of 199 is essentially the same as 100 or 10. Then is there any benefit of allowing the limit to fall below 199?
>
> Well that depends on the application workload. If application is mostly IO-bound, then maybe not. The concurrency limiting is most likely because of a degraded external dependency. Allowing the limit to decrease from 200 to 199 will help as it'll prevent Tomcat queuing, but below 199 will not help.
>
> If the application has mixed worked i.e. both IO and CPU-bound, then the application might be constrained by CPU resources. Maybe there is a sudden surge in CPU-intensive tasks. In such cases, reducing the concurrency may help prevent contention and context-switching overhead.

# Issues With This Approach

### It Does Not Measure True Latency

For this library, the system output, based on which it adjusts the concurrency limits, is the time worker threads take to process the request. It does not take into account the time request spends waiting in the queue, hence it does not measure the true latency as observed by clients.

<div class='imgwrapper'><img src='/assets/MRT.png' /></div>

The library only measures `T1`, whereas the latency from client's perspective is `T1 + T2` (ignoring any network delay). This can cause the library to observe that the service is fine when there is a queue building (due to excess traffic) and not trigger load shedding.

### It Almost Makes Queuing An All-or-Nothing Proposition

With this library, either the queuing is full-on (`CL >= W`) or none at all (`CL < W`). This is something I did not expect from a concurrency control system. Ideally queuing changes should be gradual i.e. more queuing when service is running fine, less queuing when service starts experiencing overload and no queuing when service is heavily overloaded.

# An Alternative Proxy Based Approach

Another approach that can overcome the limitations of concurrency limits library is to stick a proxy (NGINX, HAProxy, Envoy) in front of the service and move all concurrency control logic inside the proxy.

<div class='imgwrapper'><img src='/assets/MRTProxy.png' width='860' /></div>


With this approach, the definition of some of the terms we have been using changes. For example:

* ###### In-Flight Requests (IFR) or Current Concurrency

  In the earlier approach, the IFR was the number of requests that are currently being processed by worker threads. So IFR is always less than or equal to the maximum number of worker threads (W).

  In the proxy based approach, IFR is the number of requests sent to the service by proxy and are currently pending. It includes requests being processed by worker threads as well as requests waiting in the queue. For example, if W = 200, then IFR can be 500, which means that 200 requests are being processed and rest 300 are waiting in queue.

* ###### Measured Response Time (MRT) or Round Trip Time (RTT)

  In the library based approach, the MRT or RTT was the time taken by worker threads to process requests i.e. T1. Using the proxy, MRT also includes the time requests spend in queue i.e. `T1 + T2`. Hence it measures the actual latency experienced by callers.

### How Proxy Based Approach Would Work

The proxy starts with a configured `initialConcurrency`. For example, if `initialConcurrency` is configured as 500, then proxy will allow 500 in-flight requests to the service at any time. Some of these will be immediately picked up by service's worker threads, while others will wait in service's internal queue. Excess requests will be rejected by the proxy.

The concurrency control algorithm employed by the proxy (it could be AIMD, Vegas or Gradient) will keep track of RTT of requests using which it'll adjust the concurrency limits. If the limit is increased, more requests will wait in service's internal queue and RTTs will increase which will cause the concurrency control algorithm to decrease the limit. Over time, the algorithm will find the right limit.

### Advantages

The proxy based approach has some clear advantages over the library based approach. Increasing/decreasing limits has a smoother control effect. If concurrency limit is increased, it allows more requests to wait in queue. The number of requests that wait in the queue is directly proportional to the current limit. If limit is decreased, less request will have to wait in queue. It limit goes below `W`, no queuing will happen within the service.

|  Concurrency Limit | Max Requests That Can Be Concurrently Processed By Worker Threads | Max Requests That Can Wait In Queue
| -------- | -------- | | -------- |
| 300     | 200     |100|
| 500     | 200     |300|
| 1000     | 200     |800|
| 200     | 200     |0|
| 50     | 50     |0|

### Options Available For Proxy-Based Approach

As far as I know, popular proxies (NGINX, HAProxy etc.) do not provide support for adaptive concurrency control out-of-the-box or even via plugins. Presently (May 2020), there are two options available that I'm aware of:

#### Envoy

The first mention (that I know of) of Envoy supporting the adaptive concurrency control is this [InfoQ article](https://www.infoq.com/articles/envoy-service-mesh-cascading-failure/) in which they mention the concurrency limits library and also that they will be working with the team at Netflix to bring this functionality to Envoy.

And the amazing team behind Envoy has actually brought this feature to Envoy via the [adaptive concurrency filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/adaptive_concurrency_filter). This feature is currently experimental and under active development. If you are already using Envoy or planning to use this in future, this is your best bet.

#### Kanaloa

Another option is the [Kanaloa proxy](http://iheartradio.github.io/kanaloa/) by IHeartRadio. Kanaloa is developed in Scala using Akka and take a slightly different approach than what we have discussed here. I would suggest reading this [detailed post](http://iheartradio.github.io/kanaloa/docs/theories.html) on the approach used by this proxy. It does not seem to be in active development (last commit was 3 year ago). If you are comfortable with Scala/Akka, it might be worth a try as the approach here is quite good.

# Conclusion & Further Reading

With this, we conclude our section on the caveats of the concurrency limits library that you should be aware of and the alternative proxy-based approach.

<a class='nextlink' href='/2020/05/15/adaptive-concurrency-control-algorithms-part-4.html'>
<b>Next</b><br />
In part 4 of the series, we will discuss some important concerns related to load shedding specifically load shedding based on the criticality of requests and the interplay of load shedding and autoscaling..
</a>

* [https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/adaptive_concurrency_filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/adaptive_concurrency_filter)
* [http://iheartradio.github.io/kanaloa](http://iheartradio.github.io/kanaloa)
