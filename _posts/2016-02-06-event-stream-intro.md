---
layout: post
title:  "Event Stream Processing"
date:   2016-02-06 19:59:38 +0100
comments: false
categories: clojure events streams
---

#Event Stream Processing - Motivation

Event Stream Processing (ESP) is an interesting way in which we can model our systems. 

If we think of actions in the world as events (and we should!) then we would like to process them in way that befits this understanding.

Traditionally we have been unable to do this for three major reasons:

- **lack of cheap and available processing capacity.** This led to an unhealthy split between tools for batch and online processing. It did however allow the expensive hardware to be exploited close to 24/7 rather than just during the online day.
- **lack of good programming models.** A callback based module is objectively harder to program, test and debug that a sequential module. Functional languages support lazy evaluation to enable support for (real or imagined) infinite streams. 
- **lack of motivation.** We could afford to be 24 hours, 12 hours or 1 hour behind and the users either wouldn't care or wouldn't have a better choice since that was the IT industry standard.

These reasons have been relentlessly crushed:

- commodity hardware and cloud rental makes it straightforward to process as much as we would like in real-time.
- concurrent programming models are now built into languages (Go, C#, Erlang, ES6) and provided as macros in Clojure and other languages in the form of Reactive Extensions (RX).
- mobile devices and IoT motivate the need to provide up to date versions of data for users at all times, everywhere. Companies in every sector are now providing this level of quality so it is quickly becoming a cost of doing business.

#CSP for ESP. No thanks JSP.

Modern concurrency models are based on Communicating Sequential Processes that is a formal model for allowing independent sequential processes to communicate using channels. I'm not going to repeat the theory here but if you are hungry for more, see [Rob Pike's presentation on concurrency and parallelism][go-video].

CSP is a powerful enabler for Event Stream Processing. Furthermore, with the advent of microservices, ESP has become an interesting model to glue together events that occur between the independent microservices. Likewise mcroservices provide a natural model to expose stream data in casual yet powerful ways. Casual because stream consumers can be created aside from the stream producer and powerful because they can join multiple streams or process time series windows of events to create new categories applications built on high-order time based events.

#Streams everywhere?

In this opening post I am asserting that this vision displaces the database as the centre piece of online data processing. 

There does however, remain a place for these technologies. Databases and search engines offer facilities that cannot be matched or replicated by event streams (ACID transactions and complex data mining). Database themselves can be the source of events as new transactions are applied. This is evidenced by the Transaction Queue in Datomic, the replication log of CouchDB and the oplog of MongoDB. Likewise SQL Server and DB2 can be queried through Change Data Capture interfaces to obtain deltas from a given point in time.

#Tooling choices

There is a huge range of tools and technologies to ingest, store and process events: from cloud services such as [Amazon's Kinesis Streams][aws-kinesis], powerful frameworks such as [Apache Kakfa][apache-kafka], [Apache Flink][apache-flink] and all the way down to individual language features.

I always want to start off with something concrete that doesn't require a lot of infrastructure but can get demonstrate the benefits so I don't want to rely on the cloud or big frameworks - not just yet.

I could have picked Go or JS but I'm a big advocate for Clojure so [core.async][core-async] is a natural choice for me to start my investigations. 

#Full DisClojure

To be honest, I know (and my colleagues insist) that there are other languages and frameworks that are richer by default than core.async. But my other goal here is to research understand the basic building blocks on which this model operates.

In my experience the abstractions in Clojure are tastefully chosen. So I decided to start there on the basis that I would not get lost either learning a new language (like Go) or in the intricacies of distributed persistence addressed by more complex frameworks (like Flink). 

If I can also provide some examples that are useful in the Clojure community: bonus.

Oh and it's my own time and I wanted to have some fun, so shut up.

#Event Stream Processing - example scenarios

When dreaming up some scenarios I wanted to make something easy for my fellow <del>corporate drones</del> developers to grok so I figured that a small subset of an order management system would fit the bill. 

This type of system is ripe with opportunities to demonstrate events throughout and I have made an overview illustration of a subset of a bare bones system here:

![Simple stock management]({{ url }}/assets/Stock-Tracking-Streams-1.jpg)

In this series of posts I will show the following examples in code:

- **Reading events** from streams (Stock Tracker in the above drawing)
- **Joining streams** to combine data (Stock Management Tracker)
- **Process time series events** in the form of sliding intervals (Sliding Window Tracker)

The code is all on my GitHub and I will point you to the relevant code in each post.

**Thanks**


[go-video]: https://www.youtube.com/watch?v=cN_DpYBzKso
[aws-kinesis]: https://aws.amazon.com/kinesis/streams/
[apache-kafka]: http://kafka.apache.org/
[apache-flink]: https://flink.apache.org/
[core-async]: https://github.com/clojure/core.async/
