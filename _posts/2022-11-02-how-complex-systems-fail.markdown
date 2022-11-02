---
layout: post
title:  "All your systems will become safety critical"
date:   2022-11-02 11:45:48 +0200
categories: til
---


"All your systems will become safety critical" is what Richard I. Cook says in his presentation ["Resilience In Complex Adaptive Systems" at 2:28, "The future is safety"](https://www.youtube.com/watch?v=PGLYEDpNu60&t=2m4s). This is why it's important to learn about resilience engineering.

One of the best resources on the topic is to read about the work by Richard I. Cook.

[Dr. Richard I. Cook (1953 – August 31, 2022)](https://biologicalsciences.uchicago.edu/news/features/richard-cook-obituary) was a system safety researcher, physician, anesthesiologist, university professor, and software engineer. Cook did research in safety, incident analysis, cognitive systems engineering, and resilience engineering across a number of fields, including critical care medicine, aviation, air traffic control, space operations, semiconductor manufacturing, and software services.
Richard I. Cook passed away on August 31, 2022. RIP.

### Resources about "How Complex Systems Fail"

* Wikipedia page ["Richard Cook (safety researcher)"](https://en.wikipedia.org/wiki/Richard_Cook_(safety_researcher))

* [Article "How Complex Systems Fail"](https://www.adaptivecapacitylabs.com/HowComplexSystemsFail.pdf)
  * ["the morning paper" commentary about this paper.](https://blog.acolyer.org/2016/02/10/how-complex-systems-fail/)

* [Velocity 2012: Richard Cook, "How Complex Systems Fail" - YouTube](https://www.youtube.com/watch?v=2S0k12uZR14)

* [Velocity 2013: Richard Cook, "Resilience In Complex Adaptive Systems" - YouTube](https://www.youtube.com/watch?v=PGLYEDpNu60)

* [Velocity conf 2016 keynote: Situation normal: All fouled up](https://www.oreilly.com/content/situation-normal-all-fouled-up/)


### Resources about applying "How Complex Systems Fail" to DevOps

* John Allspaw blog posts:
  * [2009-11-12 How Complex Systems Fail: A WebOps Perspective](https://www.kitchensoap.com/2009/11/12/how-complex-systems-fail-a-webops-perspective/)
  * [2011-04-07 Resilience Engineering: Part I](https://www.kitchensoap.com/2011/04/07/resilience-engineering-part-i/)
  * [2012-06-18 Resilience Engineering: Pair II: Lenses](https://www.kitchensoap.com/2012/06/18/resilience-engineering-part-ii-lenses/)

* [Velocity Europe, John Allspaw, "Anticipation: What Could Possibly Go Wrong?"](https://www.youtube.com/watch?v=gy2lTFD4560)

### Applying "How Complex Systems Fail" to rearchitecting Apache Pulsar

This is something that I'm working on. The design presented in ["Rearchitecting Apache Pulsar to handle 100 million topics, Part 1"]({% post_url 2022-10-21-possible-high-level-architecture %}) and ["Rearchitecting Apache Pulsar to handle 100 million topics, Part 2"]({% post_url 2022-10-24-rearchitecting-pulsar-part-2 %}) already contain many elements that are influenced by the principles of resilience engineering. Continuing to improve in this area is one of my goals.

One of the main motivations for me in applying resilience engineering to Apache Pulsar is that I believe what Richard Cook explains in the presentation ["Resilience In Complex Adaptive Systems" at 2:28, "The future is safety"](https://www.youtube.com/watch?v=PGLYEDpNu60&t=2m4s). Richard Cook makes the claim that all systems will become safety critical systems.

A system like "Apache Pulsar" is critical infrastructure that many other systems rely on. Let's say that "Apache Pulsar" is used as the communications backbone for an IoT system. In IoT, there are use cases where IoT systems are used for implementing panic button backends, for example used by elderly people. If the backend system is not available when the panic button is pressed in an emergency, that could be a life critical outage. Many IoT systems have already become safety critical systems. In my opinion, relying on "Apache Pulsar" alone for messaging in critical systems is not sufficient preparation and design. For example, the blog post ["Pulsar's high availability promise and its blind spot"]({% post_url 2022-10-14-pulsars-promise %}) explains some details about the high availability promise. 

Building safety critical systems that rely on messaging would require much more focus on resilience and reliability patterns for ensuring end-to-end service. It is not sufficient to rely on Apache Pulsar for messaging alone. However, since all systems will become safety critical systems in the future, how do we improve Apache Pulsar in this area?

The mistake in building safety critical systems is to trust any intermediate component between endpoints. This is a true dilemma with message brokers such as Apache Pulsar. This takes us to another topic which is the ["End-to-end principle"](https://en.wikipedia.org/wiki/End-to-end_principle). You can read more about that [in the commentary of "End-to-End Arguments in System Design" by "the morning paper"](https://blog.acolyer.org/2014/11/14/end-to-end-arguments-in-system-design/). 
To be continued in another blog post in the future.



