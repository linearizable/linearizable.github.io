---
layout: post
title: "Handling Overload With Adaptive Concurrency Control - Part 1"
author:
- Vikas Kumar
---

<div class='bannercontainer'><img src='/assets/lizard2.jpg' width='940' /></div>

As the [Google SRE Workbook](https://landing.google.com/sre/workbook/chapters/managing-load/) says

<div class='highlighter'>No service is 100% available 100% of the time.</div>

There is a multitude of factors that can cause a service being unavailable or very slow ([Slowdown in the new outage](https://www.youtube.com/watch?v=-q0sd-5b40U)). One of the major factors is overload. Overload occurs when a service is unable to keep up with the amount of work it is given. Again, there can be numerous factors which can cause a service to become overloaded such as:

##### Increase in Traffic

A sudden increase in traffic can cause a service to experience overload. The traffic surge could be external such as DDOS from bad actors, increased search engine crawling activity, push notifications and marketing events or internal such as unintentional DOS (bad script) or a new service/script is deploying which starts using this service extensively.

##### Traffic Pattern Changes

A service usually serves different types of requests with different resource requirements. Some requests could be just be a lookup in cache or a small and efficient query in databases. Others could be quiet resource intensive such as complex database queries, execution of a lot of complex business logic or requiring multiple calls to external services. Service owner usually look at the traffic distribution of requests, do some load testing and plan capacity accordingly.

However sometimes the traffic pattern changes in a way that causes an increase in resource-intensive requests. Although a very small proportion of overall traffic, an increase in such requests can cause the service to become overloaded.

##### Slow upstream dependencies

As mentioned above, when upstream dependencies (databases, caches, services) become slow, the service can exhibit overload behaviour. Databases can become slow because of data growth (sudden or gradual), disproportionate  increase in slower queries due to traffic pattern changes, GC pauses etc.

In this series of articles, we'll look at load shedding as a way of effectively handling overload. We'll explore various techniques that can be used for load shedding and discuss their pros and cons. In the subsequent articles of the series, we'll focus on adaptive concurrency control as one of the most effective techniques for this purpose.

<div class='aside'>
<h2 id="concurrency-limiting-behaviour-in-spring-boot-application">A Reference Application</h2>
<p>To make the discussion in this and subsequent articles more concrete, we'll take a reference application and use it as a context. The example I want to take is that of a Spring Boot application with Apache Tomcat web server.</p>

<p>In this <a href='https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-application-properties.html#server-properties'>setup</a>, there is a fixed number of application threads defined by <code>server.tomcat.threads.max</code> property. This value is configurable and has a  default value of 200. These are the threads which are actually processing the requests (executing the application code). That means at any point of time, there are a maximum of 200 requests being concurrently processed. </p>

<p>This does not mean that at any point of time, there will be only a maximum of 200 requests being accepted by the server. Tomcat, using the NIO or APR connector, can accept more requests (defined by <code>server.tomcat.max-connections</code> property which has a default value of 10000 for NIO connector and 8192 for APR). The additional requests (apart from the 200 being processed by application threads) will wait in a queue for their turn. As application threads free up, they will pick requests from the queue and process.</p>

<p>In addition to this queue, there is also a queue at OS level (defined by <code>server.tomcat.accept-count</code> property with default value 100).</p>

<p><img src="/assets/tomcat.png" alt=""></p>

<p>The server will only start rejecting requests when all the worker threads are busy and both the queues are full. Queuing here is a useful mechanism to handle small bursts in traffic. For example, if we get 500 requests at once, then instead of rejecting excess 300, we can queue them and process when worker threads are free. However, excess queueing can lead to a lot of latency for callers as requests will spend a lot of time waiting in queue.</p>
</div>
<p></p>
Now let's take a look at different load shedding techniques.

## Rate Limiting

First technique that comes to mind when thinking about handling overload is rate limiting. Conceptually, rate limiting can be depicted as:

<div class='imgwrapper'><img src='/assets/RateLimitingOverload.png'  /></div>

The incoming traffic can be bursty, but it is converted into a constant request flow (e.g. 100 requests/second) by the rate limiter. Rate limiting can be configured at global level or on a per-client basis or both.

Although rate limiting is a useful technique and can work well in a lot of scenarios, it has it's limitations when it comes to effectively handling service overload. Specifically, service capacity (how much work it can handle) can fluctuate in a dynamic environment, but the rate limiter is usually not aware of it. It'll happily keep sending the requests as long as the flow doesn't exceed the rate limit parameters. I'll refer to this fantastic presentation to learn more about this (As a matter of fact, this presentation is the inspiration behind this article series).


<div class='iframecontainer'>
<iframe width="900" height="415" src="https://www.youtube.com/embed/m64SWl9bfvk" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

### Implementation

I have written [another post](/2020/05/01/rate-limiting-techniques.html) which summarises various options for implementing rate limiting that you can refer to.


## Concurrency Control

Another approach for load shedding is concurrency control i.e. limiting the number of requests service handles concurrently. The concurrency in this context refers to the number of in-flight requests (IFR) i.e. the requests that have been sent to the service and awaiting response. If we take our reference application as example, it includes requests being processed by worker threads as well as requests waiting in the queue.

A common approach is to limit the IFRs at the number of worker threads as shown below:

<div class='imgwrapper'><img src='/assets/ConcurrencyLimit.png'  /></div>

However, we can keep this number slightly more than the number of worker threads to allow for some queuing within the service.

### Concurrency control vs Rate limiting

With rate limiting, we send a constant flow of requests to the service irrespective of whether the previously sent requests have completed or not. With concurrency control, once the IFR reaches the limit, further requests will be throttled until one or more IFRs complete, either with success or error.

To give you an analogy, consider a restaurant with a sitting capacity of 500 and waiting area (at the bar) capacity inside the restaurant of 50. With rate limiting, we can send at most 10 persons/minute irrespective of how many people are already there inside the restaurant. If the seats don't free up fast enough, there will be too much crowd inside the restaurant causing a lot of unhappy customers and restaurant staff.

With concurrency limiting, once 550 people are inside the restaurant, we'll have to wait for some people to come out before we can send more people in.

So unlike rate limiting, concurrency control takes into account the service capacity. If all the threads are busy, then it will not send additional requests which will prevent excessive queuing and fail-fast for clients. Although in practice some queuing is desired so that we can process requests with a little delay during small traffic spikes.


### Implementation

Like rate limiting, concurreny control can also be implemented either at the service level or using a proxy in front of the service. For example, in the reference application, we can set the number of worker threads equal to the desired concurrency and the tomcat `max-connections` equals to the desired queue size.

If we want to use a proxy, the proxy can track the number of in-flight requests and keep it under the desired concurrency
by rejecting excess requests. The concurrency value at the proxy can be a bit higher than the maximum number of worker threads in the service to utilize a little bit of queuing within the service.  

## Adaptive Concurrency Control

Although concurrency control with a small queue is as good technique to avoid issues due to overload, it also has some issues, mostly because of the fixed limits on concurrency and queue size. As we discussed before, how fast the service processes requests can keep on changing depending on:

* Types of requests (specifically their resource requirements)
* Responsiveness of upstream dependencies (databases, caches, other services etc.)

If we consider a service with maximum worker threads as 200 and fixed queue size of 100, depending on the current service performance:

* Is 200 as maximum concurrency the right value? If service start processing a lot of CPU intensive tasks, you may want to reduce it.
* Is 100 the right queue size? If service is fast, you may want to queue more requests rather than shedding them as they are likely to be processed in a acceptable timeframe. If service is slow, you many want to reduce the queue size.

Adaptive concurrency control is a technique to dynamically adjust the concurrency value depending on the current service performance. If service is fast, concurrency is increased to allow more throughput. When the service performance degrades, concurrency is reduced to avoid excess queuing and allow the service to recover.

Adaptive concurreny control is type of a [closed-loop feedback control system](https://en.wikibooks.org/wiki/Control_Systems/Feedback_Loops) as depicted in the diagram below.

<div class='imgwrapper'><img src='/assets/feedback.jpeg'  /></div>

The control system measures the output of the system and uses a feedback loop to adjust the input to meet a desired output response. In this context:

* Output is the time service takes to process requests
* Input is the concurrency

### Implementation

Primarily, there are two ways to implement the adaptive concurrency control:

##### Within The Service

This approach is used by [Netflix's concurrency limits library](https://github.com/Netflix/concurrency-limits). It can integrate with GRPC or servlet based applications. For our reference application, we can integrate this library using a servlet filter. The concurrency limits library is based on TCP congestion control algorithms and is focus of [part 2](/2020/05/15/adaptive-concurrency-control-algorithms-part-2.html) of this series.

##### Using a Proxy

Another approach is to use a proxy in front of the service to implement adaptive concurrency limiting logic.   I'll discuss this approach in detail in [part 3](/2020/05/15/adaptive-concurrency-control-algorithms-part-3.html) of this series.

# Conclusion & Further Reading

<a class='nextlink' href='/2020/05/15/adaptive-concurrency-control-algorithms-part-2.html'>
<b>Next >></b><br />
In part 2 of the series, we explore the Netflix concurrency-limits library in detail and discuss the algorithms used by the library.
</a>

Here are some wonderful resources to learn more about the subject:

* [https://landing.google.com/sre/workbook/chapters/managing-load](https://landing.google.com/sre/workbook/chapters/managing-load)
* [https://www.evanjones.ca/prevent-server-overload.html](https://www.evanjones.ca/prevent-server-overload.html)
* [Load shedding lessons from Google: https://www.youtube.com/watch?v=XNEIkivvaV4](https://www.youtube.com/watch?v=XNEIkivvaV4)
* [https://blog.acolyer.org/2018/11/16/overload-control-for-scaling-wechat-microservices](https://blog.acolyer.org/2018/11/16/overload-control-for-scaling-wechat-microservices)
* [https://medium.com/@NetflixTechBlog/performance-under-load-3e6fa9a60581](https://medium.com/@NetflixTechBlog/performance-under-load-3e6fa9a60581)
* [https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload)
* [https://cloud.google.com/blog/products/gcp/using-load-shedding-to-survive-a-success-disaster-cre-life-lessons](https://cloud.google.com/blog/products/gcp/using-load-shedding-to-survive-a-success-disaster-cre-life-lessons)
* [https://ferd.ca/handling-overload.html](https://ferd.ca/handling-overload.html)
* [https://www.infoq.com/articles/anatomy-cascading-failure](https://www.infoq.com/articles/anatomy-cascading-failure)
* [https://www.infoq.com/articles/envoy-service-mesh-cascading-failure](https://www.infoq.com/articles/envoy-service-mesh-cascading-failure)
* [https://www.datadoghq.com/videos/the-anatomy-of-a-cascading-failure-n26](https://www.datadoghq.com/videos/the-anatomy-of-a-cascading-failure-n26)
