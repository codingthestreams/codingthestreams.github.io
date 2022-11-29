---
layout: post
title:  "Rearchitecting Apache Pulsar to handle 100 million topics, Part 4"
date:   2022-11-29 09:52:13 +0200
categories: pulsar
toc: true
lightbox: true
lightbox_mermaid: true
---

## Context

Welcome to continue to read about the possible design for rearchitecting Apache Pulsar to handle 100 million topics.

The [previous blog post explains the context]({% post_url 2022-11-28-rearchitecting-pulsar-part-3 %}).

## Project "Apache Pulsar NextGen"

Rearchitecting Apache Pulsar to handle 100 million topics is one of the goals of the next generation architecture for Apache Pulsar. 
This architecture is unofficial and it hasn't been approved or decided by the Apache Pulsar project. 
Before the project can decide, it is necessary to demonstrate that the suggested changes in the architecture are meaningful and address the problems that have become limiting factors for the future development of Apache Pulsar. 

### Summary of goals

* introduce a [high level architecture with a sharding model]({% post_url 2022-10-21-possible-high-level-architecture %}) that scales from 1 to 100 million topics and beyond.
* address the micro-outages issue with Pulsar topics. This was explained in the blog post ["Pulsar's high availability promise and its blind spot"]({% post_url 2022-10-14-pulsars-promise %}).
* provide a solid foundation for Pulsar future development by getting rid of the current [namespace bundle centric design]({% post_url 2022-10-17-namespace-bundle-is-a-limiting-factor %}) which causes unnecessary limitations and complexity to any improvements in Pulsar load balancing.

### The way forward

[The previous blog post explained the next milestone]({% post_url 2022-11-28-rearchitecting-pulsar-part-3 %}#next-experiments--milestones-on-the-way-toward), the minimal implementation to integrate the new metadata layer to Pulsar.
This blog post will continue with the detailed design that is necessary to get the implementation going.

## Topic lookup and assignment flow - components and interaction

The plan for the first steps in the implementation is to restore pulsar-discovery-service ([already in progress in develop branch](https://github.com/codingthestreams/pulsar/commits/develop)) and then start modifying the solution where we migrate towards the new architecture.

As explained [in the previous blog post]({% post_url 2022-11-28-rearchitecting-pulsar-part-3 %}#next-experiments--milestones-on-the-way-toward), a Pulsar client will connect to the discovery service for lookup.

![Topic lookup](https://pulsar.apache.org/assets/images/binary-protocol-topic-lookup-f013216a8dae04823eb9d39a0f2e264e.png)

This is a point where we will start the "walking skeleton" by keeping the solution "working" at least for the "happy path" of Pulsar consumers and producers.
In the new architecture, the topic lookup will trigger the topic assignment. That is the reason why it's useful to call the end-to-end flow as "topic lookup and assignment flow".

### Components participating in the topic lookup and assignment flow

There was a draft listing of possible components [in one of the previous blog posts]({% post_url 2022-10-21-possible-high-level-architecture %}#storage-model-events-are-the-source-of-truth):
* topic shard inventory
* tenant shard inventory
* namespace shard inventory
* topic controller
* cluster load manager
* broker load manager

This could be a starting point for describing the components and the roles.


```mermaid
%%{init: {'theme': 'neutral'}}%%
sequenceDiagram
    autonumber
    participant PC as Pulsar client
    participant DS as Discover service
    participant TC as Topic controller<br/>(sharded)
    participant CM as Cluster manager
    participant BM as Broker manager<br/>(sharded)
    participant B as Broker
    participant TNI as Tenant inventory<br/>(sharded)
    participant NI as Namespace inventory<br/>(sharded)
    participant TI as Topic inventory<br/>(sharded)
    B-)BM: Establish session
    BM-)B: Session id
    loop 
      B-)BM: Heartbeat
      B-)BM: Load report      
    end 
    TC->>+CM: Lease capacity units for assignments
    CM->>+BM: Lease capacity units for assignments
    BM->>-CM: Capacity units lease
    CM->>-TC: Capacity units lease
    PC->>+DS: Lookup topic
    DS->>+TC: Lookup topic
    TC->>+BM: Assign topic
    BM->>-TC: Topic assignment received (position)
    BM-)TNI: Subscribe tenant metadata & policy
    BM-)NI: Subscribe namespace metadata & policy
    BM-)TI: Subscribe topic metadata & policy
    TNI-)BM: Tenant metadata & policy
    NI-)BM: Namespace metadata & policy
    TI-)BM: Topic metadata & policy
    BM->>B: Assign topic (includes tenant, ns, topic metadata)
    Note over BM,B: Broker manager keeps track of metadata & policy data<br/>that has been sent to the broker during the session.<br/>The broker is expected to cache all metadata.
    B-)BM: Update sync position
    BM-)TC: Sync position for broker
    TC->>-DS: Lookup topic response
    DS->>-PC: Lookup topic response
```


### Component communications

In the architecture, there's a need for communication from each sharded component to components of other shards.
The initial plan is to use either gRPC or [RSocket protocol](https://rsocket.io/) for implementing this communication.

The discovery of other shards is via the "cluster manager" component. That provides services for service discovery.

The current assumption is that each shard is forms a Raft group and all components are sharded across these shards.
The cluster manager would form it's own separate Raft group.

Whenever the leader of the shard gets assigned, it will connect to the cluster manager and update the leader's endpoint address.
All nodes of all shards will keep a connection to the cluster manager and get informed of the shard leader changes.



```mermaid
%%{init: {'theme': 'neutral'}}%%
sequenceDiagram
    autonumber
    participant CM as Cluster manager's<br/>Membership manager
    participant SL as Shard Leader's<br/>Membership manager
    participant SF as Shard Follower<br/>Membership manager
    SL-)CM: Register shard leader<br/>and connect for updates
      Loop
        CM-)SL: Update shard leader end point
      end
    SF-)CM: Connect shard follower for updates
      Loop
        CM-)SF: Update shard leader end point
      end
```

The endpoint that is registered is a gRPC or a RSocket endpoint address. 
There will be a way to send messages to a sub-component that is contained in the shard. Each shard is expected to be uniform and consistent hashing is used to locate a specific key. This key will be the tenant name, namespace name or topic name in the case of tenant inventory, namespace inventory or topic inventory components. The topic name is also the key for the topic controller. 


## Summary

This blog post describes the components and interactions participating in the topic lookup and assignment flow.  
There are a lot of small details that will need to be addressed on the way to implementing this solution. 
Each sub-component interaction procotol will be specified in detail after there's some feedback from proof-of-concept implementation that is ongoing.
