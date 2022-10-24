---
layout: post
title:  "Rearchitecting Apache Pulsar to handle 100 million topics, Part 2"
date:   2022-10-21 20:23:56 +0300
categories: pulsar
toc: true
---

## Context

Welcome to read about the possible design for rearchitecting Apache Pulsar to handle 100 million topics.

The [previous blog post explains the context]({% post_url 2022-10-21-possible-high-level-architecture %}).

This blog post continues on more details and defines the scope of the first experiment.

## Reasons for switching the Pulsar metadata storage model from PIP-45 to a sharded model

The previous blog post presented a major change for the Apache Pulsar metadata storage. Let's take a closer look in the reasons to do this.

### More observations about the PIP-45 Metadata Store abstraction

Pulsar's [MetadataStore](https://github.com/apache/pulsar/blob/master/pulsar-metadata/src/main/java/org/apache/pulsar/metadata/api/MetadataStore.java) 
abstraction contains the repository pattern and in addition contains a way to register handlers for notification events and a view to the data that is cached.

Here's a compressed list of the methods in the interface:

```java
public interface MetadataStore extends AutoCloseable {
    CompletableFuture<Optional<GetResult>> get(String path);
    CompletableFuture<List<String>> getChildren(String path);
    CompletableFuture<Boolean> exists(String path);
    CompletableFuture<Stat> put(String path, byte[] value, Optional<Long> expectedVersion);
    CompletableFuture<Void> delete(String path, Optional<Long> expectedVersion);
    CompletableFuture<Void> deleteRecursive(String path);
    void registerListener(Consumer<Notification> listener);
    <T> MetadataCache<T> getMetadataCache(Class<T> clazz, MetadataCacheConfig cacheConfig);
    <T> MetadataCache<T> getMetadataCache(TypeReference<T> typeRef, MetadataCacheConfig cacheConfig);
    <T> MetadataCache<T> getMetadataCache(MetadataSerde<T> serde, MetadataCacheConfig cacheConfig);
}
```

This is a traditional CRUD interface with additional change notification and caching support.
The problem with this abstraction is that it's not optimal for a large scale distributed system such as Pulsar.

There are several problems:
* lack of support for sharding which is necessary for horizontal scalability.
* interface doesn't cover pagination and more advanced ways to search and list entities. Pagination will be necessary when there's a large amount of entities.
* all changes are broadcasted to all connected clients. This conflicts with scalable design.

The PIP-45 metadata store abstraction cannot be "fixed" or optimized to address the requirements, what there are for Pulsar for metadata handling. The interface is not complete in it's current form, since adding support for pagination or new features such as renaming of topics would cause large changes. The previous blog post covered possible new features like renaming or moving topics. 
Kafka is improving in this area with ["KIP-516 Topic Identifiers"](https://cwiki.apache.org/confluence/display/KAFKA/KIP-516%3A+Topic+Identifiers). Pulsar will need to match Kafka also in this area.

Martin Kleppmann's ["Making Sense of Stream Processing"](https://www.confluent.io/stream-processing/) ebook contains a case study "Web Application Developers Driven to Insanity" (pdf page 50, ebook page 40).
That is a case study of a web application using traditional CRUD architecture, and it explains the complexity which could arise from having the database as the source of truth.
This is explained in the chapter called "Using Logs to Build a Solid Data Infrastructure". Using logs as the source of truth can reduce overall complexity. 
Event logs as the source of truth is the model that scales well for distributed systems. This is a model that Pulsar should be leveraging internally to reduce complexity and improve reliability.

### Is the metadata shard presented in the previous blog post like a distributed database which uses Raft, such as CockroachDB or YugaByte?

Yes and no. There are similarities to distributed database design.
The main difference is the purpose of the component. The Pulsar metadata shard is not a general purpose database. Instead, it's a deployable Pulsar component that contains multiple software components that handle entities for a specific "shard" in the system.
The possible software components that run in the "metadata shard":
* topic shard inventory
* tenant shard inventory
* namespace shard inventory
* topic controller
* cluster load manager
* broker load manager

The components running in each shard will be able to act on local state. Raft will enable low latency replication and high availability of this state.
The assumption is that modeling and building stateful components for Pulsar in this way will reduce complexity and be a more efficient and effective way
to achieve the quality requirements of low cost, high performance, low latency, high availability and reliability.

Instead of thinking of a database, it is better to think about "state". What is the state that is stored in Pulsar, and what state transitions are there?
The source of truth are the events stored in the log. The Raft solution is used to replicate the event logs. Raft's replication model is based on event logs.

Raft is not the only solution to replicate state in the solution. Event logs will be available for consuming and the state of consumer position can be used to achieve consistency after state changes. This was explained partially in the previous blog post.

### Benefits of event logs as the source of truth for Pulsar metadata

The benefit of this approach is that it's possible to achieve:
* data locality and data consistency
* replicated, high available state machines
* sharding and scalability

This type of architecture will follow "single writer principle".

One inspiration for this is the ["Out of the tar pit" paper](http://curtclifton.net/papers/MoseleyMarks06a.pdf)(["Out of the tar pit" commentary in the "the morning paper" blog](https://blog.acolyer.org/2015/03/20/out-of-the-tar-pit/)).

> The biggest problem in the development and maintenance of large-scale software systems is complexity — large systems are hard to understand. We believe that the major
> contributor to this complexity in many systems is the handling of state and the burden that this adds when trying to analyse and reason about the system. 

The assumption is that the overall complexity of the system can be reduced by reducing shared mutable state in the system. 

## Scope for the first experiment

### Experimenting with Apache Ratis and the sharding model

The goal of the first set of experiments would be in validating the feasibility of Apache Ratis for the "metadata shard" layer and starting to sketch out the protocol between the 
Lookup & Admin layer components. The protocol between "metadata shard" layer components and the broker can be postponed. 

It should be possible to demonstrate the creation of 100000 topics across 5 shards. The first experiment doesn't necessarily have to follow the Pulsar model closely since the main goal is to get started. The experiment could be used to estimate throughput and latency of creating 1 million entities, 10 million entities and so on. 

The upcoming blog posts will cover the experimentation progress and any topics that come up on the way.

## Summary

* One reason to switch from Pulsar PIP-45 design to an event sourced model is that the assumption is that this results in less complexity and a more reasonable approach for meeting the scalability and reliability requirements together with the new features that will be needed in the future (such as topic renaming/moving support).
* The "metadata shard" design is an approach where there's focus on state management and replication of the state across the distributed components. 
  * Metadata isn't handled as a separate concern outside of the system. It's handled inside the system.
* The scope of the first experiment is to experiment with Apache Ratis and the sharding model.




















































