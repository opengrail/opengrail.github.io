---
layout: post
title:  "Event Stream Processing - Managing state with core.async"
date:   2016-02-09 19:59:38 +0100
comments: true
categories: clojure events streams core.async
---

# Introduction

In my [last post][core-async-intro-post] I went through the code for a data flow using a product stock tracker. It had a simple filter to ensure that the notifications occurred when a specific business rule was fired.

In that post I showed the basic mechanisms for creating and connecting channels. In this post I want to show how to combine data from multiple streams. To demonstrate this we will implement the processes around the Stock Management Tracker. 

![Simple stock management]({{ url }}/assets/Stock-Tracking-Streams-2.jpg)

In the tracker we will join the two streams to emit a running total of stock on hand vs orders at any point in time.

# Data first

Let us just quickly outline the data in our model:
{% highlight clojure %}
; Order
{:item-id A123 :description "Windscreen wiper" :quantity 2 :customer-id C234}
; Delivery
{:item-id A123 :description "Windscreen wiper" :quantity 2 :supplier-id S234}
{% endhighlight %}

Once again this is certainly a simple model but is sufficient to demonstrate the power of the solution. To help with the simulation we will throw together some sample data and create a few small functions to generate the orders and deliveries.

# Order generation


# Delivery generation


# Don't change that channel




# Summary

We have seen the stock data flow through a tracker which had a simple filter to ensure that the notifications occurred when a specific business rule was fired.

And this all happened in real time with minimal resource usage. Scaling down is often as equally important as scaling up.

# Next next next

In the next post I will show the Stock Tracker to combine data from more than one channel and maintain local state within the go block.

# And finally - Thanks!
 
Thanks for making it through. I have a better understanding of core.async after writing this and I hope that's true for you too!
 
Zing me or ping me if this was useful via Twitter or in the comments below.

[core-async-intro-post]: {% post_url 2016-02-06-event-streams-core-async %}

{% include disqus.html %}
