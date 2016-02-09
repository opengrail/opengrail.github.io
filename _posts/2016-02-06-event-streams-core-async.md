---
layout: post
title:  "Event Stream Processing with core.async"
date:   2016-02-06 19:59:38 +0100
comments: true
categories: clojure events streams core.async
---

# Introduction

In my [initial post][overview-post] I outlined a small bare-bones order management system that could act as a motivation for Event Stream Processing. 

In this post I want to show some code to implement how we may implement the processes around the Stock Tracker.

![Simple stock management]({{ url }}/assets/Stock-Tracking-Streams-2.jpg)

# Data first

Let us just quickly outline the data in our model:
{% highlight clojure %}
{:id A123 :supplier S345 :quantity 100}
{% endhighlight %}

This is certainly a simple model but not necessarily useless and reduces the clutter around the examples.

# Start with a printer

At the REPL we define a generic printer for data...

{% highlight clojure %}
(defn printer [data-ch]
  (let []
    (go-loop []
      (if-let [data (<! data-ch)]
        (do
          (clojure.pprint/pprint data)
          (recur))))))
=> #'stock.core/printer
{% endhighlight %}

The `go-loop` construct is to establish a go block with a forever loop. The loop will recur until the `if-let` condition fails while running a read `<!` on the channel.

Any attempt to read from a closed channel will result in a nil and cause the read to fail. So once we close off the channel the program will quietly end.

{% highlight clojure %}
(def data-channel (chan))
=> #'stock.core/data-channel
(printer data-channel)
=>
#object[clojure.core.async.impl.channels.ManyToManyChannel
        0x6b17235
        "clojure.core.async.impl.channels.ManyToManyChannel@6b17235"]
{% endhighlight %}

Next we create a channel with the `chan` function and pass that to create a running instance of the printer.

The printer is now sitting in the background waiting on data from that open channel and we have our first core.async program up and running.

Let's play with it a little at the REPL and then shut it down:

{% highlight clojure %}
(async/put! data-channel "Hello")
=> true
"Hello"
(async/close! data-channel)
=> nil
(async/put! data-channel "Goodbye")
=> false
{% endhighlight %}

`put!` and `take!` are used for interacting with the channels outside of a `go` block or a `go-loop`.

OK, so far so good. We have a basic example so let's look at the next program that will do something useful in the system...

# Next - the end
The most trivial consumer in the diagram is the Supplier Notifier which is at the end of the stream process flow in this case.
 
That process listens on the Supplier Order channel and calls the supplier API.

Let's fake up a call to the service...

{% highlight clojure %}
(defn supplier-api-call [order]
  (println "Calling REST API with order")
  (clojure.pprint/pprint order))
=> #'stock.core/supplier-api-call
{% endhighlight %}

Next let's get our Supplier Order channel consumer defined so that it can invoke the API...

{% highlight clojure %}
(defn supplier-notify [order-ch]
  (let []
    (go-loop []
      (if-let [order (<! order-ch)]
        (do
          (supplier-api-call order)
          (recur))))))
{% endhighlight %}

Nothing suprising there. We substitute the printing function with the call to the API. 

Let's get the channels created and hooked up:

{% highlight clojure %}
=> #'stock.core/supplier-notify
(def supplier-order-ch (chan))
(def supplier-order-ch-mult (async/mult supplier-order-ch))
(supplier-notify (async/tap supplier-order-ch-mult (chan)))

=> #'stock.core/supplier-order-ch
=> #'stock.core/supplier-order-ch-mult
=>
#object[clojure.core.async.impl.channels.ManyToManyChannel
        0x2e68af05
        "clojure.core.async.impl.channels.ManyToManyChannel@2e68af05"]
{% endhighlight %}

The incredibly observant amongst you will have noticed that Order Reconciliation will also listen to the supplier order channel. So this time we have used `mult` to create a multiple of the `supplier-order-ch`. 
                                                                                                                                    
`mult` enables several clients to listen to the same channel when they call the `tap` function on a new channel. `tap` as the name implies copies the data verbatim from the original channel onto the new channel that we are using for our notifier.

Let's give that a quick spin:

{% highlight clojure %}
(put! supplier-order-ch {:id (gensym) :supplier "Acme Supplies" :quantity 100 })

=> true
Calling REST API with order
{:id G__12827, :supplier "Acme Supplies", :quantity 100}
{% endhighlight %}

# Warm up complete

So that's the first edge of the system complete. 

The interesting general point so far is that we have been able to develop and test this all out in the REPL.

We want to keep that up as we go forward to model the Stock Tracker:

{% highlight clojure %}
(defn stock-tracker [low-water-mark stock-levels]
  "Show where stock levels fall for an item below a low water mark"
  (let [out (chan 1 (filter (fn [part-level]
                              (let [backlog (:quantity part-level)]
                                (< backlog low-water-mark)))))]
    (go-loop []
      (if-let [stock-level-for-part (<! stock-levels)]
        (do
          (>! out stock-level-for-part)
          (recur))
        (close! out)))
    out))

=> #'stock.core/stock-tracker
{% endhighlight %}

In this case see our first transducer - the filter function that is attached to the channel. The filter here acts as a simple barrier on the output channel.

That's it. Our Stock Tracker is complete. Let's test it out...

{% highlight clojure %}
(def stock-tracker-ch (chan))
(def stock-tracker-ch-mult (async/mult stock-tracker-ch))
(def orders-ch (stock-tracker 50 (async/tap stock-tracker-ch-mult (chan))))

(channel-printer orders-ch)

=> #'stock.core/stock-tracker-ch
=> #'stock.core/stock-tracker-ch-mult
=> #'stock.core/orders-ch
=>
#object[clojure.core.async.impl.channels.ManyToManyChannel
        0x7618699
        "clojure.core.async.impl.channels.ManyToManyChannel@7618699"]
{% endhighlight %}

Here we set up the channels in the same manner as our last case with the addition of our printer to have eyes on whether the filter is doing its job.

Let's run a couple of test cases through:

{% highlight clojure %}
(put! stock-tracker-ch {:id (gensym) :supplier "Acme Supplies" :quantity 10 })

=> true
{:id G__13669, :supplier "Acme Supplies", :quantity 10}
(put! stock-tracker-ch {:id (gensym) :supplier "Acme Supplies" :quantity 100 })

=> true
{% endhighlight %}

In the second case our low water mark has not been breached so the data is dropped.

# Streaming hot

Finally let's hook up all of the components and run through the same test cases:

{% highlight clojure %}
(def stock-tracker-ch (chan))

(def stock-tracker-ch-mult (async/mult stock-tracker-ch))

(def supplier-order-ch (stock-tracker 50 (async/tap stock-tracker-ch-mult (chan))))

(def supplier-order-ch-mult (async/mult supplier-order-ch))

(supplier-notify (async/tap supplier-order-ch-mult (chan)))

=> #'stock.core/stock-tracker-ch
=> #'stock.core/stock-tracker-ch-mult
=> #'stock.core/supplier-order-ch
=> #'stock.core/supplier-order-ch-mult
=>
#object[clojure.core.async.impl.channels.ManyToManyChannel
        0xcc08681
        "clojure.core.async.impl.channels.ManyToManyChannel@cc08681"]
{% endhighlight %}

We don't need the printer this time around since the notify function will report is invocation.

And here we see it flow through:

{% highlight clojure %}
(put! stock-tracker-ch {:id (gensym) :supplier "Acme Supplies" :quantity 10 })

=> true
Calling REST API with order
{:id G__13741, :supplier "Acme Supplies", :quantity 10}

(put! supplier-order-ch {:id (gensym) :supplier "Acme Supplies" :quantity 100 })

=> true
{% endhighlight %}

# Summary

We have seen the stock data flow through a tracker which had a simple filter to ensure that the notifications occurred when a specific business rule was fired.

And this all happened in real time with minimal resource usage. Scaling down is often as equally important as scaling up.

# Next next next

In the [next post][core-async-state] I show the Stock Management Tracker to combine data from more than one channel and maintain local state within the go block.

# And finally - Thanks!
 
Thanks for making it through. I have a better understanding of core.async after writing this and I hope that's true for you too!
 
Zing me or ping me if this was useful via Twitter or in the comments below.

[overview-post]: {% post_url 2016-02-06-event-stream-intro %}
[core-async-state]: {% post_url 2016-02-09-event-streams-core-async-state %}

{% include disqus.html %}
