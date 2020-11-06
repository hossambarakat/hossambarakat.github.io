---
layout: post
title: Microservices Monitoring Practical Tips
tags: Microservices, Monitoring
---

During the development of microservices we keep our focus on the architecture of each microservice, integration between microservices, defining the boundery for each service,... however we tend to forget about how we are going to monitor those service after deployment to production.

I have found that two main tips helped me in my implementation:

### Centralized Structured Logging

Each microservice could be hosted on a separate node(s) and each request coming into the system may  require communicating with other microservices so when you have a bug (I know it is rare situation :P) having the logs on each node would make diagnosing of bug very difficult.

**Structured logging** is technique which is using a consistent, predetermined message format containing semantic information. So in each message we can include service name, server ip, user id,... and then query the logs using those fields so I can find all log messages related to specific user, also you build aggregates views such as number of error messages per server.

There are a lot of tools available such as [ElasticSearch](https://www.elastic.co/products/elasticsearch) and [Seq](https://getseq.net).

### Use of Correlation ID

Any request coming into the system might result in communication between different microservices that are hosted on different nodes. This means that bug investigation requires reading all logs generated from all microservices relevant to the failed request. I have found an easy solution was correlating log entries relevant to specific request using correlation id.

Once a request enters the system, I assign a correlation to it and keep passing that correlation id to all microservices, then I use that correlation id in all log entries and with the structured logging I can search for specific request using its correlation id.

But how I would know the correlation id of a request? I add the correlation Id to the response headers of my request, this was very handy specially when testers found a bug, they report it with correlation id of the failed request and thanks to structured logging I can trace it back and diagnose what went wrong.
