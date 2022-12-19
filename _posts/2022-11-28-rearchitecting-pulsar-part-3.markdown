---
layout: post
title:  "Rearchitecting Apache Pulsar to handle 100 million topics, Part 3"
date:   2022-11-28 17:31:07 +0200
categories: pulsar
tags: pulsar-ng
toc: true
---

## Context

Welcome to continue to read about the possible design for rearchitecting Apache Pulsar to handle 100 million topics.

The [previous blog post explains the context]({% post_url 2022-10-24-rearchitecting-pulsar-part-2 %}).

## Results from experimenting with Apache Ratis and the sharding model

The goal of the first set of experiments was about validating the feasibility of Apache Ratis for the "metadata shard" layer.
It was mainly about learning the role of distributed consensus for ensuring strong consistency and learning about the implementation tradeoffs and limitations. 

There was a goal to demonstrate the creation of 100000 topics across 5 shards. This was more like a simulation and it didn't follow the Pulsar model closely since the main goal was to learn about the characteristics and practical details of implementing the metadata layer with an event driven approach.

The first experiment with Apache Ratis was completed and the code is available in https://github.com/lhotari/pulsar-ng-experiment-1 .

### Performance in local tests

The latency of adding a new entry to the Raft log is around 2-3 milliseconds. The throughput of about 300 to 500 ops/second is significantly lower than the [10000 IOPS mentioned in a slide deck](https://www.slideshare.net/Hadoop_Summit/high-throughput-data-replication-over-raft/30). Perhaps 10000 IOPS is across multiple Raft groups?
With batching, the latency stays fairly low when increasing the batch size. Adding 100000 topics in the experiment completes in a few seconds when using batching across 5 shards. 
The throughput could also be increased with pipelining, by having multiple requests in flight at a time. This hasn't been done in the experiment.


## Next experiments / milestones on the way toward 

The main goal is to integrate the new architecture to real Pulsar broker so that we can learn about the gaps in the solution.
This is about following ["walking skeleton"](https://wiki.c2.com/?WalkingSkeleton) and ["tracer bullets"](https://wiki.c2.com/?TracerBullets) development strategies to kick start development and let the feedback guide further development.
The main focus would be on the "happy path" when implementing the walking skeleton.


### Minimal implementation to integrate the new metadata layer to Pulsar

In the new architecture, the lookup and admin layer are separated from the brokers. 
One possible way to start moving forward is to start the "lookup component" based on pulsar-discovery component which was removed by [PR #12119](https://github.com/apache/pulsar/pull/12119).

The Pulsar documentation contains a description of the [topic lookup](https://pulsar.apache.org/docs/2.10.x/developing-binary-protocol#topic-lookup). There's no need to modify the existing binary protocol to support the new architecture.
It's possible that there's no need to have a separate redirect step in the lookup.

Here are some on the essential use cases to support for the admin layer and the lookups.

* Minimal Admin API implementation against the new metadata layer
  * Create, delete, list tenants
  * Create, delete, list namespaces
  * Create, delete, list topics
* Minimal topic lookup layer so that a Pulsar client can perform lookups
  * In the new architecture, the topic lookup will trigger the topic assignment. 
    * Lookup is part of the "topic assignment flow"

What do we mean with a "flow"? This is some type of end-to-end usage flow. Externally, the topic lookup flow is visible to the client application. Internally there's a topic assignment flow that must be implemented.
When focusing on the flow, we are also following the principle of "form follows function".

### Topic assignment flow

As part of the topic lookup flow, the topic will get assigned to a particular broker.  
It will be necessary to describe the components and the interactions in the assignment flow.


### Topic handover flow 

One of the main reasons for the new architecture is to support per-topic state so that topic unloading can be handled in a way where the topic is handed over from one broker to another in a seamless way.
This is another flow to start working on together with the topic assignment flow.

## Summary

This blog post was a short update on the experiment with Ratis and it explains the high level plan forward.
Future blog posts in this series will provide more clarity and details.
