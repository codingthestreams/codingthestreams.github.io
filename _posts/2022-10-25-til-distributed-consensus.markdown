---
layout: post
title:  "TIL about Distributed Consensus and Systems design"
date:   2022-10-25 15:50:12 +0300
categories: til
---

## Context

I'm working on experimentation for finding ways how to deal with distributed consensus in the experiment I'm doing for [rearchitecting Apache Pulsar for 100 million topics]({% post_url 2022-10-24-rearchitecting-pulsar-part-2 %}). 

## Today I Learned (TIL)

### TIL about distributed consensus

The Google SRE book has a very good overview of distributed consensus in ["Chapter 23 - Managing Critical State: Distributed Consensus for Reliability", written by Laura Nolan, edited by Tim Harvey](https://sre.google/sre-book/managing-critical-state/).

It contains an in-depth description about algorithms, protocols and implementations and the different tradeoffs, challenges and optimizations for addressing challenges. For example, throughput and latency can be challenges in distributed consensus. 
The article describes various ways how this has been optimized in some distributed consensus implementations.

### TIL about distributed systems design

At InfoQ, there's a recorded presentation ["Essential Complexity in Systems Architecture" by Laura Nolan](https://www.infoq.com/presentations/complexity-distributed-behavior/) which is the author of the book chapter.
It provides interesting views of how distributed systems could be designed. The author believes that there are two major styles for distributed systems architecture: command & control and peer-to-peer. Google Bigtable and Dynamo are compared. 
The presentation brought up important aspects of maintainability and operability and how the system architecture design could impact this be improving the understandability and predictability of the system.

Referenced papers:
* [Google Photon paper](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/44686.pdf)
  * [Commentary of Photon on "the morning paper"](https://blog.acolyer.org/2014/12/04/photon-fault-tolerant-and-scalable-joining-of-continuous-data-streams/)
* [Google Bigtable paper](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)
* [Amazon Dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)


More distributed consensus papers, compiled by Heidi Howard:
[Distributed Consensus Reading List](https://github.com/heidihoward/distributed-consensus-reading-list)

"the morning paper" 10 part series about consensus and replication, by Adrian Colyer:
[Can’t we all just agree?](https://blog.acolyer.org/2015/03/01/cant-we-all-just-agree/)


### TIL about herd effect with leader election

I was reading papers about distributed consensus and replication in the "the morning paper". There's also a few blog posts about Zookeeper, for example https://blog.acolyer.org/2015/01/27/zookeeper-wait-free-coordination-for-internet-scale-systems/ .

What caught my eye was the way leader election was described:
"The leader election algorithm is not actually shown in the paper, but you can find it at the [ZooKeeper Recipes and Solutions page](https://zookeeper.apache.org/doc/r3.8.0/recipes.html#sc_leaderElection). Since it’s a commonly cited use case for ZooKeeper I give a quick outline here. The basic idea is similar to the lock use case: create a parent node ‘/election’, and then have each candidate create an ephemeral sequential child node. The node that gets the lowest sequence id wins and is the new leader. You also need to watch the leader so as to be able to elect a new one if it fails – details for doing this without causing a herd effect are given at the referenced page, the same ‘watch the next lowest sequence id node’ idea as we saw above is used."

I compared this description to the implementation we have in Pulsar:
[pulsar-metadata/src/main/java/org/apache/pulsar/metadata/coordination/impl/LeaderElectionImpl.java](https://github.com/apache/pulsar/blob/82237d3684fe506bcb6426b3b23f413422e6e4fb/pulsar-metadata/src/main/java/org/apache/pulsar/metadata/coordination/impl/LeaderElectionImpl.java#L175)

I don't see that there isn't any mitigation for the herd effect in Pulsar's LeaderElectionImpl. I guess the herd effect will amplify with the growth of the participants in the leader election since when the leader is lost. With a large number of brokers this is start to matter. In general, this seems to be a minor issue for scalability compared to the fact that [all Zookeeper changes are broadcasted to all nodes in Pulsar](https://github.com/apache/pulsar/pull/11198).


### TIL about availability

While browsing "the morning paper" blog, my eye caught on ["Meaningful availability"](https://blog.acolyer.org/2020/02/26/meaningful-availability/) blog post. 

The blog post explains that "count-based" availability is a common approach for addressing the issues around partial failures in a system. It also lists some problems with count based availability metrics. There's also details around different ways to measure availability from individual's user's perspective.

I was thinking about this in the blog post ["How do you define high availability in your event driven system?"]({% post_url 2022-10-14-high-availability-for-event-driven-systems %}).


### TIL about Apache Ratis

I started looking into [Apache Ratis](https://ratis.apache.org/) which is an open source Java implemention for Raft consensus protocol. 
In my previous blog post [I shared the goal that I have with experimenting with Ratis]({% post_url 2022-10-24-rearchitecting-pulsar-part-2 %}#experimenting-with-apache-ratis-and-the-sharding-model).

As the first step, I am looking for inspiration in the LogService implementation that is part of Ratis examples. The source code for ratis-logservice is available at [https://github.com/apache/ratis-hadoop-projects/tree/main/ratis-logservice](https://github.com/apache/ratis-hadoop-projects/tree/main/ratis-logservice). It's a distributed log implemented on top of Apache Ratis.
I'm learning about the Ratis APIs and how something like the LogService could be implemented with Ratis. Instead of using and copying LogService as-is, I'm planning to implement the prototype for the "topic inventory" service and "metadata shard" (concepts explained in the previous blog post) in a minimalistic way so that I learn the use of Ratis from the basics.


### What's next

Tomorrow I'll continue learning and experimenting with Apache Ratis. I'll keep on sharing what I learn on the way. Stay tuned!
