---
layout: post
title:  "A critical view of Pulsar's Metadata store abstraction"
date:   2022-10-18 15:57:14 +0300
categories: pulsar
tags: pulsar-ng
---

This blog post is a continuation of the previous blog post ["Pulsar's namespace bundle centric architecture limits future options"]({% post_url
2022-10-17-namespace-bundle-is-a-limiting-factor %}).

## Context

### Stating the problem before describing the solution

Leslie Lamport wrote a short paper in 1978 ["State the Problem Before Describing the Solution"](https://www.microsoft.com/en-us/research/publication/state-problem-describing-solution/). I'm following this principle and continuing to state problems before describing the solution. The solution will be shared later in the blog post series.

### Assumptions

I'm making the assumption that replacing "namespace bundles" in the Pulsar load
balancing design will be an essential design change to resolve availability
problems related to broker shutdowns and broker load balancing. There is a need
to have control topics individually to minimize service disruptions that were
explained in the previous blog post. 

The Pulsar load balancing design depends on the Pulsar metadata solution. These
two cannot be separated in a performant, reliable and cost-efficient solution.
The Pulsar metadata solution will be greatly impacted by the removal of
"namespace bundles".

Because of this dependency, I'm taking a deeper look at Pulsar's metadata store
abstraction and what problems should be addressed when revisiting the design.

## What is the PIP-45 Metadata store abstraction / Pluggable metadata interface

### Background

[PIP-45 Metadata store abstraction / Pluggable metadata interface](https://github.com/apache/pulsar/wiki/PIP-45%3A-Pluggable-metadata-interface)
was introduced gradually in Pulsar versions between 2.6 and 2.10. The changes
for PIP-45 started already in Pulsar 2.6.0 release with [PR 5358, "Switch
ManagedLedger to use MetadataStore
interface"](https://github.com/apache/pulsar/pull/5358) . 

PIP-45 was considered feature complete in Pulsar 2.10. There was a blog post
describing it from the perspective of replacing Zookeeper in Pulsar, [“Moving
Toward a ZooKeeper-Less Apache Pulsar” by David
Kjerrumgaard](https://streamnative.io/blog/release/2022-01-25-moving-toward-a-zookeeperless-apache-pulsar/)
in 1/2022.

### The promised land of Zookeeper-less Pulsar: where is it?

There have been high hopes that replacing Zookeeper in Pulsar with etcd would
help solve issues with Pulsar metadata scalability. You can find this mentioned
also in the previously referenced blog post. 

PIP-45 is completed, but we aren't seeing any reported improvements with
scalability by switching from Zookeeper to etcd. This is not to say that PIP-45
didn't contain improvements. PIP-45 included major performance improvements such
as ["Transparent batching of ZK operations" #13043](https://github.com/apache/pulsar/pull/13043). 

What if etcd is slower than Zookeeper with Pulsar? There have been talks that
this might be the case. The community is waiting for the results. We didn't seem
to get to the promised land by getting rid of Zookeeper.

The original problem remains: Zookeeper is an in-memory store and that sets
limits for scalability. This is something that needs to be addressed.

### Metadata consistency issues from user's point of view

Currently, there is extensive caching within Pulsar when tenants, namespaces and
topics are created, modified and deleted. This can lead to inconsistent behavior
from the user's point of view. I'd like to highlight that this is not a new
problem that PIP-45 introduced.  

There's a summary of the consistency issues in [a GitHub issue comment written
in
10/2021](https://github.com/apache/pulsar/issues/12555#issuecomment-955748744).
Some problems have been addressed, but the root cause remains: some changes are
eventually consistent from the user's point of view, and this isn't documented
or defined.

There are use cases where it is necessary to be able to wait for the metadata
changes to take full effect before continuing. It would simplify the usage of
Pulsar if this was supported.

Some users have worked around this issue by assuming that Pulsar's metadata
changes are eventually consistent. In some cases the workaround could be to use
retries to tolerate problems cased by missing metadata or using polling for
checking that the changes are effective before continuing. This isn't great.
This is a problem that needs to be addressed.

### Metadata inconsistency within Pulsar and BookKeeper

There are also signs of issues where the state in a single broker seems to be in
a bad state as a result of consistency and concurrency issues with metadata
handling and caching. One example of such issue:
[Endless Topic HTTP Request Redirects](https://github.com/apache/pulsar/issues/13946) .

The PIP-45 abstractions also contain interfaces for leader election.

The Pulsar Metadata store abstraction is also used within BookKeeper as part of
the Pulsar distribution since Pulsar 2.10. I have seen several issues in test
environments with Pulsar 2.10 where the BookKeeper's metadata state has become
corrupted. After making changes in the configuration to by-pass Pulsar Metadata
interface for Bookkeeper client and Bookkeeper server (possible with
[#17834](https://github.com/apache/pulsar/pull/17834)), the problems went away. One of the
severe issues was [#17759](https://github.com/apache/pulsar/issues/17759) which is very
recently fixed in master branch with
[#17922](https://github.com/apache/pulsar/pull/17922).

It is possible that the incidents were caused by issues with the PIP-45 Leader
election abstraction. One of the fixes has been
[#17401](https://github.com/apache/pulsar/pull/17401) . This change hasn't yet been
cherry-picked to releases. 

I recommend disabling Pulsar Metadata driver for Bookkeeper client (in broker)
and Bookies until the recent fixes such as [PR
17401](https://github.com/apache/pulsar/pull/17401) and [PR
17922](https://github.com/apache/pulsar/pull/17922) have been released and that
there's confidence that Pulsar Metadata store is stable with Bookkeeper. It's
possible to configure Bookkeeper client in the broker and in bookies to by-pass
Pulsar Metadata store and use Zookeeper directly.

Another set of problems have emerged with ["PIP-118, Do not restart brokers when
ZooKeeper session expires"](https://github.com/apache/pulsar/issues/13304). I
would recommend setting `zookeeperSessionExpiredPolicy=shutdown` to reduce risk
of problems in this area until it is proven to be stable.

It feels like there should almost be a documented proof of correctness for the
distributed locks and leader election implementation. Pulsar coordination relies
on
[ResourceLockImpl](https://github.com/apache/pulsar/blob/master/pulsar-metadata/src/main/java/org/apache/pulsar/metadata/coordination/impl/ResourceLockImpl.java)
and
[LeaderElectionImpl](https://github.com/apache/pulsar/blob/master/pulsar-metadata/src/main/java/org/apache/pulsar/metadata/coordination/impl/LeaderElectionImpl.java)
which don't contain much documentation in this area.

This migth be a messy description of the challenges and it's just one view point.
However, there's more work to do until PIP-45 is stable.

### Scalability issue: all metadata changes are broadcasted to all brokers and bookies

A model where all changes are broadcasted to all nodes in the cluster is
problematic. This change was made in Pulsar 2.9.0 in [PR 11198, "Use ZK
persistent watches"](https://github.com/apache/pulsar/pull/1119). The global
change event broadcasting design doesn't follow typical scalable design
principles. This will pose limits on Pulsar clusters with large number of
brokers and bookies. The current metadata change notification solution doesn't
support scaling out when it's based on a design that broadcast all notifications
to every participant.

The [Scale cube](https://akfpartners.com/growth-blog/scale-cube) is one way to
define the scalability dimensions of a system. The centralized change notification solution is essentially a single queue, and
it will become an issue when the size of the system grows since there isn’t the
possibility to parallelize.

The scale cube's Z-axis is about sharding. A scalable solution for Pulsar metadata should be sharded
so that the design could eliminate solutions where all changes are broadcasted everywhere.


## Key takeaways

* replacing "namespace bundles" in the Pulsar design will also impact Pulsar metadata solution
* the current Pulsar metadata solution has several unresolved challenges:
  * PIP-45 has not delivered the promise of Zookeeper-less Pulsar in practice: Zookeeper remains the recommended and most performant option. Zookeeper has scalability limits with metadata size. 
  * Pulsar metadata solution has usability issues with it's eventual consistent change model which isn't properly described.
  * PIP-45 metadata change event broadcasting solution conflicts with scalable design.

The challenges with Pulsar metadata are an additional motivation to revisit the Pulsar architecture, besides addressing the availability issues in topic unloading. Addressing the Pulsar load balancing and metadata challenges are the problem space for the upcoming blog posts about the possible solutions to address this by changing Pulsar's architecture. Stay tuned!
