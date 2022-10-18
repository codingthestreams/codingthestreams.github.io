---
layout: post
title:  "How do you define high availability in your event driven system?"
date:   2022-10-14 08:20:00 +0300
categories: pulsar eventdriven
---

I have worked with strict latency and availability requirements in the past. In
one case, there was a general requirement for request-response and event
notification flows to have an end-to-end latency requirement of 1 second for p99
and max latency of 3 seconds. The maximum error rate was 1%.
Requests/notification flows exceeding 3 seconds were counted as errors.  These
metrics were calculated for a window of 1 minute. If p99 latency was over 1
second or error rate was more than 1% for this 1 minute window, this was
considered as downtime. In general, availability is defined as a ratio of uptime
(total time - downtime) and total time.

The availability requirement in the above case was 99.95% with a monthly
reporting interval. An outage in one month would be penalizing the availability
metrics for the whole month until it got cleared. A challenging requirement?

If you think of it, a distributed system is very rarely completely down. That is
why defining downtime based on error metrics or maximum acceptable latency is a
good way to measure partial "micro-outages" in a distributed system.

For some mission-critical systems, a failure in event delivery could be causing
life-threatening incidents, or it could cause huge financial losses. A critical
system should have sufficient safety measures to tolerate failures in event
delivery on a single channel. However, mission criticality is creeping into the
apps we build and reliability and resilience is becoming the expectation also
for mainstream event driven systems.

How do you define high availability in your event driven system? What is
considered as downtime in your case?
