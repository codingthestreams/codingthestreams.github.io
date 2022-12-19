---
layout: post
title:  "Pulsar's high availability promise and its blind spot"
date:   2022-10-14 16:18:00 +0300
categories: pulsar
tags: pulsar-ng availability
---

This blog post is a continuation of the previous blog post ["How do you define high availability in your event driven system?"]({% post_url 2022-10-14-high-availability-for-event-driven-systems %}). It is rare that Pulsar is completely down, but there are cases where there are short downtime for topics, if downtime is defined in the form of acceptable end-to-end latency or error rate. This blog post explains more details about the blind spots.

### Pulsar's promise

“Apache Pulsar is a highly available, distributed messaging system that provides guarantees of no message loss and strong message ordering with predictable read and write latency.”

### Pulsar load balancing

The Pulsar load balancing is described in the Pulsar manual ["Load balance across brokers"](https://pulsar.apache.org/docs/administration-load-balance). There's also a very detailed blog post about Pulsar load balancing in ["Achieving Broker Load Balancing with Apache Pulsar", by Heesung Sohn and Kai Wang](https://streamnative.io/blog/engineering/2022-07-21-achieving-broker-load-balancing-with-apache-pulsar/). 

The Pulsar manual describes the load balancing:
> Topics are dynamically assigned to brokers based on the load conditions of all brokers in the cluster. The assignment of topics to brokers is not done at the topic level but at the bundle level (a higher level). Instead of individual topic assignments, each broker takes ownership of a subset of the topics for a namespace. This subset is called a bundle and effectively this subset is a sharding mechanism.

A quick summary of how Pulsar load balancing works:

* All topics belonging to the same "namespace bundle" are assigned to run on a specific broker.
* A topic is mapped to a namespace bundle by calculating a hash and finding the bundle that matches the hash range where the calculated hash falls in.
* The number of namespace bundles can change as a result of an operation called "bundle splitting". That splits the bundle to 2 parts.
* This assignment of a bundle is done centrally by the load manager on the leader broker. When a client creates a producer or a consumer to a topic, the topic is first looked up. If the topic isn't part of an active namespace bundle, the bundle will get assigned to a broker by the load manager on the leader broker.
* Each broker will report load metrics to the load manager by storing the reports in Zookeeper
* If the load manager detects that a broker is overloaded or unbalanced with load (depends on load balancing configuration), it will take action. One action could be to split the namespace bundle in 2 parts and then move the other half of the namespace bundle to another broker. Another action could be to move the namespace bundle from the broker to another broker that has less load.


### The cause of micro-outages in Pulsar load balancing or Pulsar graceful shutdown

In Pulsar there's an operation called topic "unloading". Unloading applies to namespace bundles in Pulsar load balancing. In that case, all topics belonging to the same namespace bundle are "unloaded" to reduce load on one broker that is overloaded or unbalanced. The namespace bundle will then be assigned to another broker with less load. A similar operation of unloading topics takes place when a broker is gracefully shutdown, for example for software version upgrade or when there are configuration changes that require a broker restart.

Unloading a namespace bundle is a forceful operation without a special notification to the Pulsar client in the Pulsar binary protocol. From the client's perspective, the broker closes producers and consumers without any special reason code. 
(you can lookup [NamespaceService.unloadNamespaceBundle](https://github.com/apache/pulsar/blob/466bd894fe7dbf6ecf580f4bf6b6cd7dd5be89b4/pulsar-broker/src/main/java/org/apache/pulsar/broker/namespace/NamespaceService.java#L723-L742) in the source code and drill down to details such as [BrokerService.unloadServiceUnit](https://github.com/apache/pulsar/blob/15b33a3b7da7e4d839ccf21d2106e6cfdffc2145/pulsar-broker/src/main/java/org/apache/pulsar/broker/service/BrokerService.java#L2009-L2013) and [PersistentTopic.close](https://github.com/apache/pulsar/blob/11482048d357ccb4e4f1802304a7dd0bfd7b9c26/pulsar-broker/src/main/java/org/apache/pulsar/broker/service/persistent/PersistentTopic.java#L1278-L1348).)  
The Pulsar client will attempt to reconnect consumers and producers immediately and backing off exponentially between retry attempts. The exponential backoff contains 10% jitter ([implementation](https://github.com/apache/pulsar/blob/04aa9e8e51869d1621a7e25402a656084eebfc09/pulsar-client/src/main/java/org/apache/pulsar/client/impl/Backoff.java#L57-L85)).
By default, the Pulsar client will start reconnecting after 100ms, 200ms, 400ms, 800ms, 1600ms, 3200ms, 6400ms, 12800ms.... After the initial step there will be up to -10% of random jitter.
The total time spent after each attempt is about 100ms, 300ms, 700ms, 1500ms, 3100ms, 6300ms, 12700ms, 25500ms (-10% jitter). 
In a heavily loaded system, the namespace bundle unloading takes some time to complete. All topic ledgers in the bundle will be closed before the ownership is released. During this time, all topics in the bundle won't be able to produce or consume messages.
Since the clients retry with backoff (to reduce the "herd effect"), it can result in disruption durations that are aligned with the values of total time spent after each attempt (100ms, 300ms, 700ms, 1500ms, 3100ms, 6300ms, 12700ms, 25500ms; -10% jitter).

The unloading of a topic is forceful in a way that the producers are cut off immediately without explicitly completing storing messages that are in-flight. The existing solution seems to have many chances for race conditions. My assumption is that "topicFencingTimeoutSeconds" feature introduced by the pull request [[broker] Close topics that remain fenced forcefully #8561](https://github.com/apache/pulsar/pull/8561) is needed to workaround some race conditions in the unloading solution. There are also other issue reports in this area such as [Topic is temporarily unavailable #14941](https://github.com/apache/pulsar/issues/14941)
, [Persistent topic is always temporarily unavailable #5284](https://github.com/apache/pulsar/issues/5284) or [Topic is already fenced #15451](https://github.com/apache/pulsar/issues/15451).
The forceful termination of consumers can cause message redelivery since the broker doesn't wait for client to complete delivering in-flight acknowledgements. This isn't a major problem for at-least-once messaging, but it could also be eliminated with a proper topic handover solution. 

### Related work

There's [PIP-192 New Pulsar Broker Load Balancer](https://github.com/apache/pulsar/issues/16691) in progress which contains an improvement to topic unloading by introducing a new operation "transfer" which will inform the client of the new broker. This requires changing the Pulsar protocol and the solution won't help old clients. 

#### Unresolved challenges of PIP-192?

The current [PIP-192](https://github.com/apache/pulsar/issues/16691) design doesn't seem to contain a plan to fix the various race conditions in unloading which are the actual root cause of the issues. 

In addition, when a high-volume topic with many consumers gets moved from one broker to another, it might cause the consumers to go into catch up read mode where the broker cache isn't used and this will cause a spike in read requests to BookKeeper. This is another reason for additional latency.   

### Key takeaways

* Pulsar's load balancing solution relies on the client retrying to connect after a topic gets moved from an overloaded broker or at graceful shutdown from one broker to another. 
* The current solution doesn't provide a transparent handover and will always introduce a short disruption of service for the topics that are moved. 
* The duration of the service disruption depends on the load, resources and cardinality of topics, producers and consumers.
* The current plans in PIP-192 don't cover addressing the root causes. 

A possible solution for this topic problem will be described in future blogs posts.










