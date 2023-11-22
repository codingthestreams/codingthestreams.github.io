---
layout: post
title: "Apache Pulsar service level objectives and rate limiting"
date: 2023-11-22 16:37:10 +0200
categories: pulsar
tags: pulsar-dev slo
toc: true
---

## Context

This blog post is based on the discussion about improving Apache Pulsar's rate limiting. The discussion was initiated by a Pulsar Improvement Proposal (PIP) titled "PIP-310: Support Custom Publish Rate Limiters". The detailed email discussion can be found on the [Pulsar dev mailing list](https://www.mail-archive.com/search?l=dev@pulsar.apache.org&q=subject:%22%5C%5BDISCUSS%5C%5D+PIP%5C-310%5C%3A+Support+custom+publish+rate+limiters%22&o=oldest&f=1){:target="_blank"}.

I wrote this blog post to summarize my viewpoints expressed in this discussion and to provide more context around rate limiting in Pulsar. The PIP has a limited scope of "publish rate limiters". I believe it's necessary to take a holistic view and examine the purpose of rate limiters and message throttling in Pulsar.

## Delivering Improvements in Apache Pulsar for Messaging-as-a-Service Platform Teams

While visiting the [Pulsar Summit North America 2023 in San Francisco](https://pulsar-summit.org/event/north-america-2023){:target="_blank"} last month, I observed a very positive trend: an increasing number of messaging-as-a-service platform teams are adopting Apache Pulsar as their main building block for providing messaging services across their organizations.
For example, in one of the keynote talks, "A Journey to Deploy Pulsar on Cisco's Cloud Native IoT Platform," [Sr. Director Chandra Ganguly explains how they are migrating to use Pulsar in Cisco's IoT platform that serves over 245M devices, including over 84M connected cars](https://www.youtube.com/watch?app=desktop&v=YGImIkY6ymo&t=43m30s){:target="_blank"}.

![](https://www.youtube.com/embed/YGImIkY6ymo?start=2610)

This is clear validation that the value of Apache Pulsar's truly multi-tenant architecture is delivering results, making Apache Pulsar a cost-efficient and reliable solution for messaging-as-a-service platform teams in very demanding application environments.

In the Apache Pulsar project, we are committed to delivering further improvements to the existing multi-tenancy features. One area of improvement is the service level management and capacity management of a large Pulsar system. This is also a key concern of messaging-as-a-service platform teams.

This blog post focuses on rate limiting and throttling in Apache Pulsar and delivering improvements to that. The proposed improvements are prerequisites for expanding the work to cover broader aspects in delivering improvements for messaging-as-a-service platform teams that rely on OSS Apache Pulsar.

## Why Do We Need Rate Limiting in Pulsar?

When operating a multi-tenant platform service, it's necessary to define and provide a specific level of service to the system's users (see [Google SRE book's SLO chapter](https://sre.google/sre-book/service-level-objectives/){:target="_blank"} for more details). The primary reason for implementing rate limiting in Pulsar is to manage the service level and system capacity.

Defining explicit Service Level Objectives (SLOs) allows service providers to set proper performance expectations for consumers in a multi-tenant system like Apache Pulsar. Without clearly communicated targets, clients may develop unrealistic assumptions about latency, throughput, and reliability.

For example, in an empty Pulsar cluster with ample idle resources, end-to-end latency is extremely low. Clients could come to depend on this undefined "peak performance" as the expected service level. However, as more tenants and traffic utilize the system, resource contention increases queues and latency.

Suddenly, previously functional applications might begin failing catastrophically when backpressured if the applications are needlessly dependent on very low latency and high throughput where backpressure hasn't been applied. One concrete example of this is an application that uses Pulsar producer's asynchronous sending but lacks proper logic to handle errors in sending. When the application is backpressured, it's possible that the send queue fills up on the client side, and in the case of asynchronous sending with non-blocking behavior (Pulsar Java client: `blockIfQueueFull` option with the default value of `false`; Pulsar Go client: `DisableBlockIfQueueFull: true`), that results in the message not being sent and instead reported as failed asynchronously. It's possible that the application developer forgets to handle this type of errors, since they are rare. When this eventually happens, it might lead to application developers filing tickets where this type of behavior is reported as a message loss by the Pulsar messaging service, although the problem is in the application that isn't resilient to failures or degraded performance. When rate limiting is applied, it is more likely that such applications would fail already during testing and the problem would be resolved before it causes issues in production where the messaging-as-a-service platform team is blamed. (Besides this, there is a need to make application developer's life easier so that such details in developing messaging applications would be properly covered in Pulsar documentation and code samples. Currently you learn such details by experience which means making a lot of mistakes and dealing with the consequences. :smile:)

Rate limiting constrains tenant usage to specified quotas, enabling accurate planning and over-commitment of resources. This helps to achieve cost optimizations in operating the service across multiple tenants.

When no SLOs are defined, a certain service level becomes the implicit service level. Tenants might perceive degradation in service performance in very different ways than the service providers. The dilemma is that the service provider hasn't committed to the service level that the tenant is expecting, leading to many unnecessary conflicts.

Rate limiting is the first step in Pulsar for managing consumer expectations and preventing unrealistic reliance on peak performance.

### The Problem of Exceeding Service Level Expectations

There's a good example in the [SRE book](https://sre.google/sre-book/service-level-objectives/){:target="_blank"} about Google's Chubby lock service. The high availability of Chubby led some teams to add unreasonable dependencies, assuming it would never fail. However, when service degradations occurred, despite their rarity, dependent services were impacted.

To address this, the Chubby SRE team ensured that availability met, but did not significantly exceed, the SLO target. If no natural outages occurred in a quarter to anchor expectations, the team would deliberately take the system down. This approach forced service owners to handle Chubby unavailability, preventing an unrealistic reliance on its availability.

Explicit SLOs allow providers to properly set consumer expectations. Meeting, but not dramatically exceeding, SLOs, along with controlled failures, helps avoid dependencies that assume a level of reliability not guaranteed.

In the context of Pulsar, rate limiting helps to properly set consumer expectations about the performance that is sustainable.

## Current Rate Limiting Challenges in Pulsar

The existing rate limiters in Pulsar have several problems:

* The default publishing rate limiter is ineffective in practice due to its high CPU overhead when operating at scale, making it unusable.

* Additionally, the default rate limiter is inaccurate, and the rate limiting behavior is inconsistent. This inconsistency is one of the reasons why the "precise" rate limiter was introduced.

* The "precise" rate limiter, while not exhibiting CPU issues, lacks the ability to configure an average rate limit over longer time periods. This limitation hinders the handling of traffic spikes by allowing bursting.

* The "precise" rate limiter impacts the latencies of other topics that have not reached the rate limit. This impact is due to heavy lock contention on Netty IO threads that should optimally run non-blocking code. Lock contention causes slowdowns on all traffic served on the shared IO thread. This problem is reported as [issue #21442](https://github.com/apache/pulsar/issues/21442){:target="_blank"}.

* Having multiple rate limiting implementations and using "precise" as terminology exposes unnecessary internal details through the configuration interface. It is bad architectural practice to expose implementation details when that doesn't provide additional value.

* Pulsar relies on TCP/IP connection flow control for applying backpressure. However, single TCP connections get shared across multiple Pulsar producers and consumers multiplexed together.

There is a clear need to improve Pulsar's out-of-the-box rate limiting capabilities.

### Pulsar TCP/IP Connection Multiplexing Challenge

Pulsar relies on TCP/IP connection flow control for applying backpressure. Backpressure is achieved by pausing to read new messages from a connection, which eventually leads to a situation where the TCP/IP connection flow control "backpressures" all the way to the client. This type of backpressure solution is common in networking applications. However, the challenge in Pulsar is that a single TCP connection is commonly shared across multiple Pulsar producers and consumers, which are multiplexed together by the Pulsar client.

This creates a significant complication - all producers over that connection are throttled as one, rather than being individually rate limited. Similarly, rate limiters interact across streams unpredictably, impacting behavior. One rate limiter would throttle the connection, and another rate limiter would immediately unblock it.

The Apache Flink community encountered related issues from multiplexing backpressure on shared TCP connections. Their [2019 blog post explains the problem and solution](https://flink.apache.org/2019/06/05/flink-network-stack.html#inflicting-backpressure-1){:target="_blank"} in Flink 1.5 to introduce explicit stream-specific flow control.

Pulsar could implement similar flow control changes to enable reliable per-producer rate limiting in the multiplexing connection scenario. One possible solution would be to have Pulsar producers use "permits" for flow control, in a similar way as [Pulsar consumers use "permits" for flow control](https://pulsar.apache.org/docs/next/developing-binary-protocol/#flow-control){:target="_blank"}.

It was also highlighted in the discussion that Kafka quotas have a way to communicate from the broker to the client that it is running over quota. There was also an example from HTTP where quota exhaustion can be communicated with the 429 (Too Many Requests) status code.

Alternatively, client configuration options could isolate specific producers or consumers on dedicated, non-shared connections to avoid contention. This solution could be implemented without any changes to the Pulsar binary protocol and could be delivered quickly. It would be a way to mitigate possible issues caused by connection multiplexing before the changes are delivered in the Pulsar binary protocol.

This highlights how Pulsar's backpressure design impacts capabilities like rate limiting and would also need improvements.

## Arguments against "PIP-310: Support Custom Publish Rate Limiters"

Introducing pluggable external rate limiters has concerning implications:

* It increases the overall system complexity without corresponding gains. Apache Pulsar already supports many plugin extensions, but further fragmentation should be avoided without clear benefit.
* Public interfaces must be maintained long-term, which can hinder refactoring and slow down future development.
  * Implementation details could leak into the public interfaces, making it harder to change these details.
* No other messaging systems support custom rate limiters, so it is unclear why Pulsar would require such unique extensibility.
* Requiring external libraries just to make rate limiting usable in Pulsar would go against the good out-of-the-box experience of using Pulsar.

Here are some additional arguments against custom publish rate limiters:

* There is no evidence that custom rate limiters are necessary to solve concrete problems faced when operating Pulsar at scale. The current built-in rate limiters likely can be enhanced to handle typical use cases related to rate limiting.

* Introducing custom rate limiters narrows the focus, while capacity management and service levels require a broader, system-wide perspective.

* Merely fine-tuning custom rate limits cannot prevent overall system overload when operating a large Pulsar cluster. Custom limiters operate on a topic or namespace, ignoring cluster-wide resource usage and contention. Intelligent global capacity management capabilities are required to handle dynamic workloads.

* While custom rate limiters could provide more flexibility in modeling tenant usage, this does not inherently help meet tenant SLAs in a shared environment. Custom limiters introduce additional fragmentation, control loops, and components that must cooperate correctly at large scale.

* So far, there haven't been concrete examples of what problems a custom rate limiter could resolve and how this would happen. 
  * A concrete example has been about bursting, and that is a clear gap in the current functionality of rate limiters. Bursting support should be implemented in core Pulsar rate limiting while improving and fixing the issues of the existing Pulsar rate limiting solution.
  
In summary, custom rate limiters seem like an intricate solution lacking supporting evidence of real-world necessity. A better solution for all Pulsar users is fixing the issues in the current rate limiting and putting more effort into improving Pulsar's multi-tenancy, service level management, and capacity management capabilities since that seems to be the main reason why custom publish rate limiters are proposed.

I'd like to mention that the discussion around PIP-310 has been very valuable, and it has brought up challenges in operating Pulsar at scale. Without these discussions, the important feedback loop for Pulsar development would be missing. For example, one of the identified gaps was related to protecting against misbehaving applications and denial-of-service attacks. Pulsar doesn't currently have many capabilities in this area. It would be a useful feature for Pulsar operators to be able to block individual clients, consumers and producers when addressing certain situations such as misbehaving applications. 

We all seem to share the same mission of improving OSS Apache Pulsar to meet the broader needs of messaging-as-a-service platform teams. By working together, we can achieve a lot, even in a relatively short time span.

## Proposal for Pulsar Rate Limiting Enhancements

### Problems to address as the next step

Rather than custom pluggability, effort should go towards overhauling the built-in rate limiting:

* Consolidate the multiple existing rate limiting options into a single, configurable rate limiting solution
  * Remove the separate "precise" rate limiter. It is bad practice to leak implementation details to users when this is unnecessary.
  * Add configurable options for allowing traffic bursting
* Address these problems in existing Pulsar rate limiting:
  * High CPU overhead
  * High lock contention that impacts shared Netty IO threads and adds latency to unrelated Pulsar topic producers
  * Code maintainability
    * There are very few Pulsar core developers that understand how the current Pulsar rate limiting and backpressure work end-to-end.
    * Improve understandability of code
        * Introduce clear concepts and abstractions that are reflected in the code to ease this (example: token bucket)
  * Inconsistent behavior:
    * Connection Multiplexing currently causes a problem where multiple rate limiters operate on the same connection without knowing about each other. One rate limiter might throttle a connection and another immediately unthrottles at a different rate.
    * Connections are throttled at multiple levels in Pulsar. For example, there's a broker-level publisher rate limiting option. The multiple levels of throttling should be properly coordinated in the implementation. The current solution is not maintainable and some inconsistent behavior is caused by the lack of a proper model for coordinating throttling. The different levels of throttling might be competing on a single connection and this is another source of inconsistency besides the connection multiplexing.

### Solution proposal

One of the concrete solution plans is to use a token bucket-based algorithm to implement the rate limiter calculations.

Benefits of using a token bucket-based implementation:
* The conceptual model of the token bucket is well established in the domain of network QoS traffic policies and multi-tenant SaaS QoS.
  * Examples:
    * Wikipedia: [Token bucket](https://en.wikipedia.org/wiki/Token_bucket){:target="_blank"}
    * Cisco IoS documentation: [What Is a Token Bucket](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/qos_plcshp/configuration/15-mt/qos-plcshp-15-mt-book/qos-plcshp-oview.html#GUID-93BCD445-CC99-4808-BAE9-0527A57E671C){:target="_blank"}
    * Amazon DynamoDB: ["Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service"](https://www.usenix.org/conference/atc22/presentation/elhemali){:target="_blank"}
    * Jack Vanlightly's [The Architecture of Serverless Data Systems](https://jack-vanlightly.com/blog/2023/11/14/the-architecture-of-serverless-data-systems){:target="_blank"} explains details of many SaaS services. Token bucket-based rate limiting is widely used in SaaS data systems. 
* Implementing the token bucket algorithm is extremely simple.
* The token bucket concept and abstraction makes it easier to understand the implementation and maintain the code over time.
* The token bucket algorithm is well suited for Netty's asynchronous programming model.

Since the internals of Pulsar's rate limiting aren't currently exposed, it's possible to refactor the internals of Pulsar to work well with an asynchronous non-blocking rate limiter implementation. This is a necessity in order to solve problems like [[Bug] RateLimiter lock contention when use precise publish rate limiter #21442](https://github.com/apache/pulsar/issues/21442){:target="_blank"}.

Additionally, the proposed solution includes adding a new option to Pulsar clients. This option would allow for the configuration of Pulsar producers and consumers to isolate the specified producer or consumer at the TCP/IP connection level. This would help mitigate potential multiplexing issues when necessary.

#### Avoiding breaking changes

The only external change for Pulsar users would be the addition of configuring the bursting behavior and absolute maximum rate for each rate limiter. The default values could be selected in a way that provides a good usage experience for most users. Most users wouldn't have to tune the bursting behavior.

This is to say that changing the current Pulsar rate limiting can be done without causing breaking changes. However, according to [Hyrum's law](https://www.hyrumslaw.com/){:target="_blank"}, this might be impossible. :smile:

![XKCD: Every change breaks someone's workflow](https://imgs.xkcd.com/comics/workflow.png)

#### Asynchronous non-blocking rate limiter implementation: Proof-of-concept results

The token bucket algorithm is well-suited for Netty's asynchronous programming model. It can be implemented in a lockless, non-blocking way, which is also extremely performant. I have implemented a proof of concept (PoC) at [https://github.com/lhotari/async-tokenbucket](https://github.com/lhotari/async-tokenbucket){:target="_blank"} to provide sufficient confidence in taking the next steps in this direction.

[AsyncTokenBucket](https://github.com/lhotari/async-tokenbucket/blob/master/src/main/java/com/github/lhotari/asynctokenbucket/AsyncTokenBucket.java){:target="_blank"} is an asynchronous, non-blocking, lockless token bucket algorithm implementation optimized for performance with highly concurrent use. It doesn't use synchronization or blocking. Instead, it uses compare-and-swap (CAS) operations. The use of CAS fields could cause contention in the form of CAS loops, but this problem is addressed in the solution with multiple levels of CAS fields, so multiple threads don't frequently compete to update a CAS field in a CAS loop. The JVM's LongAdder class is used in the hot path to hold the sum of consumed tokens. The LongAdder class is a proven solution for addressing the CAS loop contention problem for a counter.

The main usage flow of the AsyncTokenBucket class is as follows:

1. Tokens are consumed by calling the `consumeTokens` method.
2. The `calculatePauseNanos` method is called to calculate the duration of a possible pause when the tokens are fully consumed.

The `AsyncTokenBucket` class doesn't have external side effects. It's like a stateful function, just like a counter function. Indeed, it is just a sophisticated counter. It can be used as a building block for implementing higher-level asynchronous rate limiter implementations, which do need side effects.

As part of the proof of concept, the performance was validated. This particular test measures the overhead of maintaining the token bucket calculations across 1, 10, and 100 threads. Here is an example output of running the [AsyncTokenBucketTest](https://github.com/lhotari/async-tokenbucket/blob/9c807bedec18a0e968113bdd0394ecebe6e4cf32/src/test/java/com/github/lhotari/asynctokenbucket/AsyncTokenBucketTest.java#L66-L100){:target="_blank"} on a Dell XPS 2019 i9 laptop:

```
❯ ./gradlew performanceTest

> Task :performanceTest

AsyncTokenBucketTest > shouldPerformanceOfConsumeTokensBeSufficient(int) > [1] 1 STANDARD_OUT
    Consuming for 10 seconds...
    Counter value 125128028 tokens:199941612
    Achieved rate: 12,512,802 ops per second with 1 threads
    Consuming for 10 seconds...
    Counter value 126059043 tokens:199920507
    Achieved rate: 12,605,904 ops per second with 1 threads

AsyncTokenBucketTest > shouldPerformanceOfConsumeTokensBeSufficient(int) > [1] 1 PASSED

AsyncTokenBucketTest > shouldPerformanceOfConsumeTokensBeSufficient(int) > [2] 10 STANDARD_OUT
    Consuming for 10 seconds...
    Counter value 1150055476 tokens:45309290
    Achieved rate: 115,005,547 ops per second with 10 threads
    Consuming for 10 seconds...
    Counter value 1152924215 tokens:45692611
    Achieved rate: 115,292,421 ops per second with 10 threads

AsyncTokenBucketTest > shouldPerformanceOfConsumeTokensBeSufficient(int) > [2] 10 PASSED

AsyncTokenBucketTest > shouldPerformanceOfConsumeTokensBeSufficient(int) > [3] 100 STANDARD_OUT
    Consuming for 10 seconds...
    Counter value 1650149177 tokens:-451095706
    Achieved rate: 165,014,917 ops per second with 100 threads
    Consuming for 10 seconds...
    Counter value 1664288687 tokens:-462912837
    Achieved rate: 166,428,868 ops per second with 100 threads

AsyncTokenBucketTest > shouldPerformanceOfConsumeTokensBeSufficient(int) > [3] 100 PASSED

BUILD SUCCESSFUL in 1m 15s
3 actionable tasks: 3 executed
```

With 100 threads, about 165M token bucket ops/second are achieved. On a single thread, it's about 12.5M token bucket ops/second.
That's a promising result in order to meet the sufficient performance characteristics for Pulsar rate limiting.

### Some implementation details of the proposed solution

In addition to the [asynchronous non-blocking token bucket implementation PoC](https://github.com/lhotari/async-tokenbucket){:target="_blank"}, I made a quick few hour development spike within the Pulsar code base to gain confidence about the implementation direction to start with.

One possible implementation strategy is to focus on replacing the existing rate limiter with [AsyncTokenBucket](https://github.com/lhotari/pulsar/blob/lh-rate-limiter-spike-2023-11-22/pulsar-common/src/main/java/org/apache/pulsar/common/util/AsyncTokenBucket.java){:target="_blank"} while simultaneously addressing the throttling coordination issue and integrating all of this in the existing code base. The remaining parts of the solution would emerge while working on the implementation iteratively. I believe that the spike provides a sufficient level of confidence that this implementation strategy could be viable.

In the spike, I added a ["ThrottleTracker" class](https://github.com/lhotari/pulsar/blob/lh-rate-limiter-spike-2023-11-22/pulsar-broker/src/main/java/org/apache/pulsar/broker/service/ThrottleTracker.java){:target="_blank"} to handle tracking of the connection's throttling status. There's a subclass, [ServerCnxThrottleTracker](https://github.com/lhotari/pulsar/blob/lh-rate-limiter-spike-2023-11-22/pulsar-broker/src/main/java/org/apache/pulsar/broker/service/ServerCnx.java#L270-L304){:target="_blank"}, where the intention is to interface deeper with the ServerCnx class while keeping the core throttling coordination logic in the ThrottleTracker class. Please note that the spike was developed in a few hours and doesn't contain a workable end-to-end solution where these parts are fully integrated with required changes.

There's a need to coordinate all various reasons why a connection is throttled and only if all throttling reasons are cleared, the connection should be unblocked. This is what the ThrottleTracker achieves with a somewhat similar approach as Netty's reference counting works for buffer allocations. The different reasons to throttle are tracked with a counter. Only when the counter reaches back to 0 will the connection get unblocked by setting autoread back to true. This would solve the problem with inconsistency with throttling across multiple rate limiters.

As mentioned above, the exact internal changes in different parts of Pulsar would emerge while integrating the [AsyncTokenBucket](https://github.com/lhotari/pulsar/blob/lh-rate-limiter-spike-2023-11-22/pulsar-common/src/main/java/org/apache/pulsar/common/util/AsyncTokenBucket.java){:target="_blank"} and ThrottleTracker/ServerCnxThrottleTracker classes into the Pulsar code base to perform rate limiting.
For example, this will directly impact the [PublishRateLimiter](https://github.com/lhotari/pulsar/blob/lh-rate-limiter-spike-2023-11-22/pulsar-broker/src/main/java/org/apache/pulsar/broker/service/PublishRateLimiter.java){:target="_blank"} interface among the other rate limiter related interfaces and implementations.

The goal is to change all throttling decisions into delegated calls to ServerCnxThrottleTracker. There will be 2 ways to throttle a connection with `ServerCnxThrottleTracker`: 
* setting a flag and unsetting it. (used by throttling based on configured `maxMessagePublishBufferSizeInMB` or `maxPendingPublishRequestsPerConnection` value)
* throttling for a specified duration and resuming after that (will be used for various rate limiters)

The publish rate limiting will be using the throttling for a specified duration. The PublishRateLimiter implementation won't be responsible for handling the throttling. Its role will be simply about:
- tracking consumed bytes and messages against the configured limits (delegates this to `AsyncTokenBucket`)
- if the limit is exceeded, return a pause duration to the caller (calculated with `AsyncTokenBucket`)

The integration code will pass the pause duration to the ServerCnxThrottleTracker, and that's how the connection will get throttled. Concurrent calls coming from various rate limiters are all handled in the ServerCnxThrottleTracker in a thread-safe and performant way, ensuring consistency in all cases.

### Proposed timeline

This proposed solution isn't far off. It's realistic to say that the proposed improvements could be part of a Pulsar feature release in less than three months and would be ready for production use.

Let's make it happen! :rocket:

After this first step, it would be time to start focusing on Pulsar cluster wide capacity management to meet the needs of messaging-as-a-service platform teams dealing with demanding service level management and capacity management requirements. There's already the Resource Group concept introduced in ["PIP 82: Tenant and namespace level rate limiting"](https://github.com/apache/pulsar/wiki/PIP-82:-Tenant-and-namespace-level-rate-limiting){:target="_blank"}. That's a good starting point for further improvements.

[Please join the Pulsar developer mailing list](https://pulsar.apache.org/community/#section-discussions){:target="_blank"} to keep updated of the developments also in this area of Pulsar. You are also welcome [join the discussions on the mailing list and on Pulsar Slack](https://pulsar.apache.org/community/#section-discussions){:target="_blank"}. You can also contact me directly by email [lhotari@apache.org](mailto:lhotari@apache.org){:target="_blank"}. 

Tomorrow on November 23rd, this blog post will be presented and discussed in the [Pulsar community meeting](https://github.com/apache/pulsar/wiki/Community-Meetings){:target="_blank"} that happens online. You are more than welcome to join [this Zoom meeting](https://github.com/apache/pulsar/wiki/Community-Meetings){:target="_blank"}.

I'm looking forward to more participation in Pulsar development from messaging-as-a-service platform teams!
