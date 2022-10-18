---
layout: post
title:  "Pulsar's namespace bundle centric architecture limits future options"
date:   2022-10-17 14:18:38 +0300
categories: pulsar
---

This blog post is a continuation of the previous blog post ["Pulsar's high
availability promise and its blind spot"]({% post_url 2022-10-14-pulsars-promise
%}), which explained Pulsar load balancing at a high level and one of the
current challenges with Pulsar load balancing from the perspective of high
availability.

### What is a "namespace bundle"

"Namespace bundles" are defined in the [Pulsar reference
documentation](https://pulsar.apache.org/docs/administration-load-balance#dynamic-assignments)
in this way:
> "The assignment of topics to brokers is not done at the topic level but at the
> bundle level (a higher level). Instead of individual topic assignments, each
> broker takes ownership of a subset of the topics for a namespace. This subset
> is called a bundle and effectively this subset is a sharding mechanism."

> "Topics are assigned to a particular bundle by taking the hash of the topic
> name and checking in which bundle the hash falls."

There has been a clear reason in the current design. With namespace bundles,
topic load balancing operations can be handled as groups (batches) to minimize
the overhead of coordination and load balancing operations.

### Implications of namespace bundles centric design

However, this design also causes essential limitations: a topic cannot be freely
assigned to a bundle or a broker. In the current namespace bundles design, a
namespace bundle is determined by calculating a hash of the topic's name and
there are hash ranges to define bundle boundaries.

There is no explicit way to assign a topic to a bundle. Splitting a bundle is
the only way to impact bundle assignment. More choices are added to how the
bundle split is determined. That is a sign of a design smell when the end users
of the system have to start working around an internal implementation detail of
the system.

The current Pulsar load balancing makes load balancing decisions dynamically,
although there are features where placement with predefined rules is used
(broker isolation, anti-affinity namespaces across failure domains). The bundle
centric design adds unnecessary complexity to load balancing decisions and
prevents optimal topic placement.

### Alternative approach: free form placement of topics to brokers

Before introducing a replacement solution for namespace bundles in Apache
Pulsar, let's first examine the benefits of letting go of namespace bundle
centric architecture.

#### Free form placement of topics provides more flexibility

There might be future requirements of having affinity and anti-affinity rules
for topic placement. It might be useful to be able to have placement rules to
take availability zones (or data center racks) into account.

Free form topic placement would also be useful for capacity management. Rate
limits could be used as part of load balancing decisions, and there could be
rules for how much total possible throughput is allowed on a specific broker
when considering the rate limits of assigned topics. Rule based placement could
also be useful when it is known that the topic traffic pattern is bursty (common
in batch oriented processing) and maximum throughput is desired while the
traffic burst is being processed. 

Dynamic load balancing is simply too slow in reacting to such changes in traffic
patterns. In these cases, it might be useful to be able to pre-assign the
different partitions of a partitioned topic to be balanced across available
brokers in the cluster with anti-affinity rules instead of making this decision
dynamically after the traffic is running. The traffic burst could be over when
the load balancer reacts.

#### Advanced form of broker isolation and capacity management

Broker isolation is another use case for free form topic placement. In the case
of capacity issues or noisy neighbor performance issues in achieving SLAs, it
would be useful to be able to assign a specific topic to a dedicate set of
brokers. Broker isolation is currently possible at namespace level, but having
this possibility at topic level would increase flexibility

On a multi-tenant SaaS platform, this would give more possibilities in meeting
QoS/SLA by having better ways for ensuring guaranteed throughput and latency
with better capacity management.

Achieving autoscaling requires better capacity management. Unless there's a way
to run the existing capacity at a certain level and measure and control this,
there aren't ways to make relevant scale-out and scale-in decisions. These
requirements could also be considered in Pulsar load balancing improvements.

#### Enabling a seamless handover of topic from one broker to another

The [previous blog post]({% post_url 2022-10-14-pulsars-promise %}) covered
issues with unloading topics and how that causes short durations of
unavailability to Pulsar producers and consumers.

In a graceful broker shutdown, the namespace bundles that are owned by the
broker are unloaded and released. There will be a short interruption in message
delivery because of this operation. Similar interruptions happen when Pulsar
load balancer decides to move a namespace bundle from one broker to another.

The benefit of free form topic placement would be such that in a graceful broker
shutdown or when load balancing decides to move topics, producers and consumers
could be migrated from one broker to another in a seamless handover,
independently of any "namespace bundle". Each topic can start serving producers
and consumers on the other broker immediately, unlike how it happens currently
that all topics must be unloaded in the previous broker before topics can be
served in the new location.

This is the key to prevent downtime and service interruptions in graceful broker
shutdown or Pulsar broker load balancing.

#### Improving availability and reliability by reducing service disruptions

When it's cheap and non-intrusive to migrate topics across brokers, Pulsar load
balancing will become more effective.

Effective Pulsar load balancing will help meet SLAs and the defined QoS levels
in a cost-effective way. Effective load balancing will also make autoscaling an
option without the risk of causing service interruptions or SLA violations.

### Redesigning Pulsar architecture

The key change to make is to replace the namespace bundle centric design with a
solution that allows free form placement of topics on brokers in flexible ways.
This all must be efficient, performant, scalable, reliable and consistent. This
is based on the assumption that the namespace bundle centric design is a
limiting factor for Pulsar.

Major changes in the Apache Pulsar project are proposed with Pulsar Improvement
Proposals (PIPs).

Before drafting a set of PIPs for replacing the namespace bundles, the approach
going forward is to make a proof-of-concept change where there would be a
significant improvement in Pulsar's operations at production time and
maintainability at development time. The PoC should demonstrate the value of
making a change to the design. It will also be a validation of the assumption
that the namespace bundle centric design is a limiting factor. 

Another key requirement for the new design is to be able to introduce a new type
of protocol for handing over a topic between brokers with the goal of minimizing
service disruptions. At high level this means, that the coordination should
involve a state machine that handles this efficiently and in a consistent
manner.

There are also multiple other reasons why an architecture redesign is needed.
Current Pulsar Admin APIs don't handle large amounts of data: there is no
pagination support. Making the Pulsar Admin APIs support pagination would
trigger a lot of changes to the existing interfaces and solutions.

#### Existing Pulsar Improvement Proposals (PIPs) related to this area

Implemented:
* [PIP-45: Pluggable metadata
  interface](https://github.com/apache/pulsar/wiki/PIP-45%3A-Pluggable-metadata-interface)
* [PIP-39: Namespace Change
  Events](https://github.com/apache/pulsar/wiki/PIP-39%3A-Namespace-Change-Events)

In progress:
* [PIP-157: Bucketing topic metadata to allow more topics per
  namespace](https://github.com/apache/pulsar/issues/15254)
* [PIP-192: New Pulsar Broker Load
  Balancer](https://github.com/apache/pulsar/issues/16691)
* [PIP-180: Shadow Topic, an alternative way to support readonly topic
  ownership](https://github.com/apache/pulsar/issues/16153)

The redesign would have to consider at least these areas where there are
existing PIPs. The above PIPs show that metadata handling and Pulsar load
balancing are very closely related topics in Pulsar and while reconsidering the
architecture choices, they cannot be separated.

For example, the design for "PIP-192: New Pulsar Broker Load Balancer", would 
be completely different if Pulsar didn't have the namespace bundle centric design
for Pulsar broker load balancing. 


### Key takeaways

* The current design of namespace bundles is becoming a limitation for future
  Pulsar improvements.

* The assumption is that if we replace "namespace bundles" in the Pulsar load
  balancing design, there will be a better way forward for Pulsar in the future.

* Pulsar load balancing design depends on the Pulsar metadata solution. These
  two cannot be separated in a performant, reliable and cost-efficient solution.
  The Pulsar metadata solution will be greatly impacted by the removal of
  "namespace bundles".

* A proof-of-concept (PoC) solution will be built to validate the assumption and
  show the value of the redesigned Pulsar architecture.

This blog post series will continue to chart the way forward. Please provide
your feedback on the Pulsar dev mailing list or by commenting on this blog post.
There will be upcoming blog posts about the high level solution architecture.
I'm sure that many are doubting whether that could be solved at all and how it is
solved. Stay tuned!
