---
layout: post
title: "Handling Overload With Adaptive Concurrency Control - Part 4"
author:
- Vikas Kumar
---

<img src='/assets/pr.jpg' width='940' />

So far in this series, we have discussed how the adaptive concurrency control works and what are the different ways to implement it in your system. In this last article, we'll discuss some important techniques that can make load shedding more effective for your business and implications of load shedding on other parts of your system (e.g. autoscaling).

# Load Shedding Based On Request Priorities

We have seen how concurrency control and load shedding can help a service remain responsive under load. But the real power of this mechanism is realised when we take request priorities into account. A service typically processes different types of requests. Some requests are more critical than others. The definition of criticality is business-specific, for example:

* A request to create a user account is likely to be more critical than a request to display user profile
* A request to purchase an item is likely to be more critical than showing a user her order history

Shedding requests based on their priorities can allow a service to process critical requests even when it's performance is degraded while shedding non-critical requests.

<h2 class='mildh2'>Defining Criticality</h2>

There can be a number of ways to define request criticality for your service. The simplest mechanism could be:

* *CRITICAL:* Request is critical and should always be given priority
* *NON_CRITICAL:* Request is non-critical and can be dropped when service is not performing optimally  

There could also be multiple levels of criticality such as *P0*, *P1*, *P2* and so on.

A useful reference is Google. In the SRE book. They define the levels of criticality as follows:

<div class='highlighter-gray'>
<p><div class='qtitle'>CRITICAL_PLUS</div>
Reserved for the most critical requests, those that will result in serious user-visible impact if they fail.</p>
<p><div class='qtitle'>CRITICAL</div>
The default value for requests sent from production jobs. These requests will result in user-visible impact, but the impact may be less severe than those of CRITICAL_PLUS. Services are expected to provision enough capacity for all expected CRITICAL and CRITICAL_PLUS traffic.</p>
<p><div class='qtitle'>SHEDDABLE_PLUS</div>
Traffic for which partial unavailability is expected. This is the default for batch jobs, which can retry requests minutes or even hours later.</p>
<p><div class='qtitle'>SHEDDABLE</div>
Traffic for which frequent partial unavailability and occasional full unavailability is expected.</p>
</div>

<h2 class='mildh2'>Identifying Request Criticality</h2>

Once we define the criticality levels, we need to identify the criticality of a given request. There can be various approaches to do this:

##### Service-Defined Criticality

A particular service can assign criticality levels to the requests it serves either by HTTP method (GET/POST/PUT) or by URI paths or a combination of two. For example, a user service can mark `POST /user` and `PUT /user` as critical while `GET /user` and `GET /users` can be marked as non-critical. When a request arrives, the service (or it's proxy) can match the request HTTP method and URI against defined rules and identify the criticality level.  

<div class='imgwrapper'><img src='/assets/ServiceDefinedPriority.png'  /></div>

##### Business Use Case-Defined Criticality

Instead of individual services defining their own request priorities, we can assign a criticality level to the top-level request at the edge (e.g. API Gateway) and then propagate it with all the sub-requests to different services in request header.

<div class='imgwrapper'><img src='/assets/PriorityPropagation.png' width='850' /></div>

The advantages of this approach is that priorities are tied to the business use cases rather than each service having it's own interpretation of criticality. For example, a service may not see a particular request as critical but it might be called by another service in a business-critical use case, which makes this request critical for that particular use case. It allows a request to be critical for some business use cases and non-critical for others.


<h2  class='mildh2'>Implementing Priority-Based Load Shedding</h2>

Unfortunately, as far as I know, none of the solutions mentioned in this series provide support for this. So it's something you have to build on your own. Here are some rough ideas:

##### Ignore critical requests in limit calculations

One simple solution is to not consider critical requests in limit calculations. Critical requests are always allowed and do not count towards the current in-flight requests. When concurrency is limited, only non-critical requests will be rejected.

A variant of this approach is to count critical requests towards in-flight requests, but to not reject when limiting is done.

##### Soft & Hard Limits

Instead of a single limit, we use soft limit (at which we start rejecting excess non-critical requests) and hard limit (at which we start rejecting all excess requests). One way to define soft and hard limits is:


* Hard Limit = Current Limit
* Soft Limit = X% of Current Limit

For example, if we defines soft limit as 10% of the current limit and the current limit is 500, then we start rejecting non-critical requests as current IFR is 450. If IFR goes up to 500, we start rejecting all requests.

##### Separate Limit Calculations

We can track IFRs and adjust limits separately for different request types.

# Autoscaling

Another important thing to consider along with load shedding is autoscaling. Autoscaling is another useful mechanism of handling overload by dynamically increasing the capacity, typically by adding more instances of the service (or external dependencies such as databases) so that the excess traffic is distributed and the traffic that a single instance of the service (or dependency) handles remains manageable.

Usually autoscaling is done by measuring some signal, like average CPU usage, and triggering the autoscaling process when signal value exceeds some pre-configured value (e.g. 70%).

The problem with autoscaling is that usually it's not instant. It takes some time for new instances of the service or database to come up and start serving traffic. If the traffic surge is high, it might exhaust the resources of the service/database before the autoscaling process is able to spin up new instances. If one or more service/database instances go down, the load balancer will redistribute the traffic among remaining instances which can strain resources on those instances as well causing a cascading failure.

<h2 class='mildh2'>Interplay of Autoscaling & Load Shedding</h2>

The autoscaling and load shedding can complement each other. Load shedding can protect the service from temporary spikes in traffic by shedding excess load. But if the increase in traffic is sustained for longer period, it's better to increase capacity rather than just keep shedding requests. Load shedding can give autoscaling process time to spin up new instances while keeping the current service alive and responsive.

However, if not considered carefully, load shedding and autoscaling can be at odd with each other. When the service starts experiencing load, it's resource usage will increase and start nearing the autoscaling trigger. In the meantime, load shedding will also trigger and start sheddding excess requests. It will reduce strain on the service and it's resource usage will go down causing autoscaling to not trigger. As the resource usage goes down and service starts becoming more responsive, load shedding will relax and stop rejecting requests. It'll again increase traffic on service causing it to become unresponsive again. This cycle will continue and we'll not be able to add more capacity via autoscaling to handle the increase in traffic.

<div class='imgwrapper'><img src='/assets/Autoscaling.png' /></div>

In order to prevent this, we need to carefully design the autoscaling conditions. In addition to resource usage, number of requests being throttled can also be taken as a signal for autoscaling. However, you should test your autoscaling setup so that autoscaling and load shedding work in conjunction.

# Conclusion & Further Reading

This concludes our series on adaptive concurrency control. It is quite a fascinating topic and I have not seen it being widely adopted in the industry except the giants like Netflix, Google and AWS. Also there is a lot more to this topics than we have covered here. I'll suggest you to go through the references I have shared with each article to deep dive further.

Thanks for reading through.

* [https://landing.google.com/sre/workbook/chapters/managing-load](https://landing.google.com/sre/workbook/chapters/managing-load)
* [Load Shedding at Google: https://www.youtube.com/watch?v=XNEIkivvaV4](https://www.youtube.com/watch?v=XNEIkivvaV4)
* [https://blog.acolyer.org/2018/11/16/overload-control-for-scaling-wechat-microservices](https://blog.acolyer.org/2018/11/16/overload-control-for-scaling-wechat-microservices)
* [https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload)
* [https://cloud.google.com/blog/products/gcp/using-load-shedding-to-survive-a-success-disaster-cre-life-lessons](https://cloud.google.com/blog/products/gcp/using-load-shedding-to-survive-a-success-disaster-cre-life-lessons)
