---
layout: post
title:  "Scalable distributed system principles in the context of redesigning Apache Pulsar's metadata solution"
date:   2023-03-02 11:38:00 +0200
categories: pulsar
tags: pulsar-ng
toc: true
---

## Context

Welcome to continue to read about the possible design for rearchitecting Apache Pulsar to handle 100 million topics.

The [previous blog post]({% post_url 2022-11-29-rearchitecting-pulsar-part-4 %}) and the [first blog in the series]({% post_url 2022-10-21-possible-high-level-architecture %}) explain more about the context. 

This is the last article in the "rearchitecting Apache Pulsar to handle 100 million topics" blog post series.

## Thoughts about architecture design in the context of Apache Pulsar's metadata solution

The target system in this case is Apache Pulsar's metadata solution. This isn't a description of a holistic architecture design which would be conducted in a top-down approch. This blog post is more about sharing thoughts that what I think that is important to consider when designing the solution. 

In earlier blog posts, there has been the design approach of "form follows function". This blog post takes a different view point and describes some of the driving forces. The "form follows function" approach is helpful in focusing on the core essence of the system so that the architecture is adapted to the true purpose of the system and not the other way around. The book [System Architecture: Strategy and Product Development for Complex Systems](https://learning.oreilly.com/library/view/system-architecture-strategy/9780136462989/xhtml/fileP7000495919000000000000000000A70.xhtml#:-:text=Function%20is%20the%20activity%2C,raction%20between%20entities.) is one of the best descriptions of this design philosophy and how it can be applied.

Iterative design is about bouncing back and forth between top-down and bottom-up. What is necessary is the architecture vision that emerges as part of these exercises. The architecture is never perfect and eventually it will be about making decision decisions and tradeoffs in the design.

In the end of the day, the designed system will only be successful when it's valuable. It's valuable when the benefits it produces is more than the cost.
The Apache Pulsar metadata solution doesn't provide direct end user value. However, the benefits and value contribute to the value creation of the Apache Pulsar system. 
Holistic system thinking and system design are essential when making larger scale redesign. However, that is not the scope of this blog post.

## Thoughts about scalable distributed systems principles in the context of Apache Pulsar's metadata solution

Scalability is defined in Wikipedia as "the property of a system to handle a growing amount of work by adding resources to the system." 

["Designing Data-Intensive Applications" by Martin Kleppmann](https://learning.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/) explains that the architecture of systems that operate at large scale is usually very highly specific to the application. There is no single architecture that works for all use cases and scales universally. However, the book discusses general purpose design elements and approaches that can be adopted to create a suitable architecture.

When designing the architecture of the target system, it is very useful to reflect on the general purpose design elements and approaches of scalable distributed systems.
This will have a great impact on the architecture. The system decomposition to components and what they are distributed and what internal protocols (messaging interactions) are used to communicate between the components.

Besides the scalable distributed systems principles, the data consistency requirements and principles have a great impact on component distribution and interactions.
All of this design must happen iteratively until the implementation of a system becomes approachable. It's possible to start immediately and iterate, but the principles will help guide the way. 

### Partitioning / Sharding

The blog post ["A critical view of Pulsar's Metadata store abstraction"]({% post_url 2022-10-18-view-of-pulsar-metadata-store %}) highlighted the necessity for scalability, particularly a **partitioning (sharding) design** for the given system. Chapter 6 in the DDIA book covers partitioning extensively, which is an often omitted detail in systems with scalability requirements.

### Shared nothing & redundancy

Shared nothing architecture is a principle related to partitioning in which each node running an application is independent and there are no shared resources, such as a shared disk or file system. Besides Partitioning, replication is also necessary to add redundancy so that individual node failures can be tolerated.

### Sufficient data consistency depending on the use cases

In the context of Apache Pulsar's metadata solution, an efficient design should take into account partitioning and replication, as well as data consistency requirements.
Metadata should be handled at a sufficient consistency level depending on the use case. The assumption is that eventual consistency is less costly and more performant, and should be used when there's a way to use it. 

### Efficiency through eventual consistency

The design could also be crafted to enable proper and efficient handling of eventual consistency for metadata in the system. Optimistic locking is an often used mechanism for managing eventual consistent data. When information is sent between components, a version field can be included to indicate which version the command or event should be referring to. Conflict resolution strategies could be applied to handle version conflicts.

In the context of Pulsar, there have been some internal discussions on the design that it would be useful if the topic lookup information could also contain some versioning information so that when the client connects to the broker where the topic has been assigned, there would be a way to allow independent asynchronous transmission of topic assignments so that the broker would know that the client is ahead of the individual broker in its metadata information. 

### Enabling parallelism

It's possible to handle this also without versioning information, but it has just come up as one example of where eventual consistent delivery of data as events between components could be handled without too much synchronicity which can be preventing parallelism, which is a really important property of high performance systems.

## Thoughts about data consistency modeling that impacts the architecture

During my career as a consultant helping companies succeed with software development and build great products, I came across a common pattern that software teams struggled with: data consistency design. Today, it is very common that an agile software team is put to deliver software without much design effort and thought put into the design. "If it works, don't fix it" approach is fine for software where it's fine that data consistency is more like a best-efforts consideration of the system. However, when data consistency is essential and data inconsistency problems have a negative business impact, it's useful to do design.

### Consistency boundaries

Back in the day, the consistency boundaries of a system where designed around database transactions. The consistency design was focused on designing how database transactions are created and when committed or rolled back. In distributed systems, we must continue to do put effort in ensuring data consistency, but it requires different set of methods. The properties are also different. There is no atomicity with distributed systems as there was in traditional monolithic non-distributed systems. Distributed transactions could provide abstractions for atomicity, but that usually comes with a heavy tradeoff cost which makes it not suitable for efficient large scale systems.  

One replacement for distributed transactions for distributed systems is the [saga pattern](https://microservices.io/patterns/data/saga.html). However, that causes the focus go away from the actual consistency design in the design of a distributed system. 

A more useful practical design approach is explained in Jonas Bonér's book  "Reactive Microsystems", in Chapter 4, Event-First Domain-Driven Design:
>**Think in Terms of Consistency Boundaries**
>
>I’ve found it useful to think and design in terms of *consistency boundaries* for the services:
>1. Resist the urge to begin with thinking about the *behavior* of a service.
>1. Begin with the data—the facts—and think about how it is coupled and what dependencies it has.
>1. Identify and model the integrity constraints and what needs to be guaranteed, from a domain- and business-specific view. Interviewing domain experts and stakeholders is essential in this process.
>1. Begin with zero guarantees, for the smallest dataset possible. Then, add in the weakest level of guarantee that solves your problem while trying to keep the size of the dataset to a minimum.
>1. Let the *Single Responsibility Principle* (discussed in “Single Responsibility”) be a guiding principle.
>
>The goal is to try to minimize the dataset that needs to be *strongly consistent*. After you have defined the essential dataset for the service, *then* address the behavior and the protocols for exposing data through interacting with other services and systems—defining our *unit of consistency*.

Lightbend offers this book as a [free download](https://www.lightbend.com/blog/reactive-microsystems-the-evolution-of-microservices-at-scale-free-oreilly-report-by-jonas-boner).

The author refers to Pat Helland's paper ["Data on the Outside versus Data on the Inside"](http://cidrdb.org/cidr2005/papers/P12.pdf). Bonér references also [7 Design Patterns For Almost-Infinite Scalability](http://highscalability.com/blog/2010/12/16/7-design-patterns-for-almost-infinite-scalability.html) which has further references to Pat Helland's paper [Life beyond Distributed Transactions: an Apostate’s Opinion](https://www.ics.uci.edu/~cs223/papers/cidr07p15.pdf) and [Vaughn Vernon's articles about Effective Aggregate Design](https://www.dddcommunity.org/library/vernon_2011/).

### End-to-end principle

In addition to the "consistency boundaries", a closely related design principle is the end-to-end principle. The foundations of the internet were build on this design principle.

["End-to-end arguments in system design – Saltzer, Reed, & Clark 1984"](http://web.mit.edu/Saltzer/www/publications/endtoend/endtoend.pdf) is a classic from almost 40 years ago. [The commentary blog post about this article in the Morning Paper blog](https://blog.acolyer.org/2014/11/14/end-to-end-arguments-in-system-design/) is a good way to get familiar with this article. 

Some quotes from the commentary:

> The end-to-end argument says that many functions in a communication system can only be completely and correctly implemented with the help of the application(s) at the endpoints.

> End-to-end arguments can help with layered protocol design and “may be viewed as part of a set of rational principles for organizing such layered systems.”

Wikipedia contains more details about the [end-to-end principle](the https://en.wikipedia.org/wiki/End-to-end_principle)

Another commentary paper is [A critical review of "End-to-end arguments in system design" – Moors 2002](https://www.csd.uoc.gr/~hy435/material/moors.pdf). The conclusions of this paper:
> The end-to-end arguments are a valuable guide for placing
functionality in a communication system. End-to-end implementations  are
supported by the need for correctness of implementation, their ability
to ensure appropriate service, and to facilitate network transparency,
ease of deployment, and decentralism. Care must be taken in identifying
the endpoints, and end-to-end implementations can have a mixed impact on
performance and scalability. To determine if the end-to-end arguments
are applicable to a certain service, it is important to consider what
entity is responsible for ensuring that service, and the extent to which
that entity can trust other entities to maintain that service. The
end-to-end arguments are insufficiently compelling to outweigh other
criteria for certain functions such as routing and congestion control.

## Closing words for "rearchitecting Apache Pulsar to handle 100 million topics" 

This blog post didn't include practical actions that will take the implementation forward.
As Jerome Bruner has said "Teaching is about speculating about possibility.". This blog post series of "rearchitecting Apache Pulsar to handle 100 million topics" has been about teaching by speculating about possibility. Perhaps this possibility has also inspired others in the Apache Pulsar community. I hope so.

"If you want to build a ship,\
don't drum up the people\
to gather wood, divide the\
work, and give orders.\
Instead, teach them to yearn\
for the vast and endless sea."

(from Antoine de Saint-Exupéry: The Little Prince)

Perhaps one day, the Apache Pulsar community will build that ship that takes Apache Pulsar to the next level.
