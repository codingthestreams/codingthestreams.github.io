---
layout: post
title:  "Possible worlds for Apache Pulsar 3.0 and making that actual"
date:   2022-10-13 09:41:57 +0300
categories: pulsar
---

Hello all,

This is a blog dedicated to sharing learnings, thoughts and experiments
about Apache Pulsar development and future improvements. The opinions
are my own and do not represent the views of my employer DataStax or
Apache Software Foundation (ASF) or Apache Pulsar PMC.

The content is targeted for everyone who would like to participate in
Apache Pulsar 3.0 planning discussions and development.

As [Jerome Bruner (1915-2016)](https://en.wikipedia.org/wiki/Jerome_Bruner) has said [in an
interview in 2014](https://www.youtube.com/watch?v=aljvAuXqhds) when he was 99-years old:

> The main thing about teaching is that it opens up a wider range of possibility, what's possible.
>
> You teach them about something in the past or in the presence, but you hope that your teaching will have the good effect
> of leading them into the world of possibility, that's where intelligence lies.
>
> Teaching is about speculating about possibility. Particularly, in the western world, we like people to 
> stick to the factual. To what goes now. I'd say the main object of teaching and educating is get them to think and 
> share their notions about where this leads.\
> To go beyond the information given.
>
> If it's only about the past, it gets you nowhere, only back to the past.\
> It is about back and forth between the possible and the actual and learning how to make that leap.
>
> Not only learning how, but learning what is necessary to do it. To use your bean. To its fullest extent.

I hope that this blog will be teaching you and me to go beyond the information given.
The articles shared in this blog will open up a wider range of possibility for Apache Pulsar. Me and other blog 
authors will be speculating about what is possible. Our focus is in the future. As Jerome Bruner
said, "If it's only about the past, it gets you nowhere, only back to the past.". 

My first goal is to teach you about the possible worlds for Apache Pulsar and how
that could be made actual. This will include details of how the current architecture
could be revisited to help Pulsar improve the capabilities of unified streaming and messaging.

Instead of adding new features to Pulsar, my passion is to improve the
quality features of Pulsar, improving reliability and resilience over
all and also focusing on improving efficiency and scalability of Pulsar.

Reliability is about delivering consistent quality over time. Apache
Pulsar is a highly available, distributed messaging system that provides
guarantees of no message loss and strong message ordering with
predictable read and write latency. 

There are some blind spots in the current solution which must be tackled
to make progress on improving Pulsar's reliability. Adding band-aid won't be sufficient. Quick fixes and kludges add more accidental
complexity. Future blog posts will uncover more details about the
possibilities for improving and how this can be accomplished. 

The content of the blog posts could be referenced in Apache Pulsar 3.0 planning
and eventually I have plans to extract actual Pulsar Improvement Proposals (PIP)
based on the blog posts and feedback discussions that take place in this blog. I'd like to remind you that all decisions for Apache Pulsar project's future are made on the Apache Pulsar dev mailing list. 

I'm looking forward to everyone's participation in the Apache Pulsar 3.0
planning discussions on the Apache Pulsar [dev mailing
list](mailto:dev@pulsar.apache.org)
([[subscribe]](mailto:dev-subscribe@pulsar.apache.org)
[[archive]](https://lists.apache.org/list.html?dev@pulsar.apache.org)).

Some related threads:
* [Pulsar 3.0 brainstorming: Going beyond PIP-45 Pluggable metadata
  interface](https://lists.apache.org/thread/tvco1orf0hsyt59pjtfbwoq0vf6hfrcj)
* [The current design of namespace bundles is becoming a limitation for
future Pulsar
improvements](https://lists.apache.org/thread/roohoc9h2gthvmd7t81do4hfjs2gphpk)
* [Pulsar 3.0 - dreaming of the ability to rename
  objects](https://lists.apache.org/thread/vrr75rrh4trqlp14objh3snlfvmzdrp2)
* [Back pressure in Apache
  Pulsar](https://lists.apache.org/thread/v7xy57qfzbhopoqbm75s6ng8xlhbr2q6)
* [Long list of Metadata inconsistency
  issues](https://github.com/apache/pulsar/issues/12555#issuecomment-955748744)

This is the most recent thread: [Planning for Apache Pulsar
3.0](https://lists.apache.org/thread/1bofpck07fgnv118s2z9qtpz7tvd8fg9).


> Use your bean, to its fullest extent.
> — <cite>Jerome Bruner</cite>


Best Regards,

Lari Hotari
