---
layout: post
title:  "Event Stream Processing - Managing time series with core.async"
date:   2016-02-11 19:59:38 +0100
comments: true
categories: clojure events streams core.async
---

# Introduction

In my [last post][core-async-state] I went through the code for a data flow using a product stock level tracker by combining channels and maintaining state around the `go-loop`

In this post I want to show how to segregate data from a stream into time series.

![Simple stock management]({{ url }}/assets/Stock-Tracking-Streams-4.jpg)

In this use case we want to notify the supplier if we have a spike in demand. This spike example is fairly trivial but we will see that core.async can handle many millions of events
per minute and provide fine grained time series with very little code.

# Data first

As usual in this series we will outline the simple data we use in our model:
{% highlight clojure %}
; Order
{:item-id A123 :description "Windscreen wiper" :quantity 2 :customer-id C234}
{% endhighlight %}

# Timing - thinking out of the box

`core.async` does not provide any models for managing time in the library. 

The addition of this support however is relatively simple and surprisingly concise. Well, maybe not that surprising by now.

The development of this model was prompted by a short conversation with @jgdavey in the core.async channel on Slack

> If you were to implement yourself, you could make a `go` that simply pulls from a channel, and adds to another as a `[timestamp item]` pair, then finally pushes into an unbounded `chan` that has a transducer that filters based on age of that timestamp.
>
> -- <cite>Joshua Davey</cite>

I couldn't get my head around the suggestion at first but I decided to give it a try - what could possibly go wrong? In the end mine was a different take on Joshua's idea but it served as inspiration.

Oh, and Joshua has [his own implementation][jgd-gist] which is interesting in its own right.

# Timing orders

{% highlight clojure %}
(defn gen-timed-orders [frequency-millis coll]
  (let [out (chan)]
    (go-loop []
      (do
        (>! out [(t/now) (rand-nth coll)])
        (<! (timeout frequency-millis))
        (recur)))
    out))
{% endhighlight %}

We saw this function in a previous example to generate infinite streams of data with time stamps so this satisfies the first part of generating pairs of data with a time / data tuple. 

# Timing windows

{% highlight clojure %}
(defn create-time-series-windows
  "Create a window for X seconds that refresh every Y seconds. 1 <= Y <= X"
  ([open-duration slide-interval]
   {:pre [(> open-duration 0) (> slide-interval 0) (>= open-duration slide-interval)]}
   (let [out (chan)]
     (go-loop [start-time (t/now)]
       (let [window (gen-window start-time open-duration)]
         (do
           (>! out window)
           (<! (timeout (* 1000 slide-interval)))
           (if (= open-duration slide-interval)
             (recur (:to window))
             (recur (t/now))))))
     out))

  ([open-duration]
   {:pre [(> open-duration 0)]}
   (create-time-series-windows open-duration open-duration)))

(finite-printer (stop-after-n-seconds 4) (create-time-series-windows 2))
=>
#object[clojure.core.async.impl.channels.ManyToManyChannel
        0x36b824c4
        "clojure.core.async.impl.channels.ManyToManyChannel@36b824c4"]
{:from
 #object[org.joda.time.DateTime 0x6b838f36 "2016-02-14T14:55:19.807Z"],
 :to
 #object[org.joda.time.DateTime 0x595b82e4 "2016-02-14T14:55:21.807Z"],
 :closed false,
 :items []}
{:from
 #object[org.joda.time.DateTime 0x595b82e4 "2016-02-14T14:55:21.807Z"],
 :to
 #object[org.joda.time.DateTime 0x18e43e29 "2016-02-14T14:55:23.807Z"],
 :closed false,
 :items []}
:stop
{% endhighlight %}

We have a multi-arity function for creating time windows. With one argument it will produce contiguous windows of X seconds. With two arguments the windows will overlap.

We use state on the `go-loop` to maintain the interval contiguousness (I checked, yes that is the right word)

We also have a few contractual checks using `{pre:` to ensure that the arguments are somewhat sane.

# Time in Windows

There are many ways to skin this particular cat. But before I go on I just want to say that no cats were actually skinned.

I went away from Joshua's concept and decided to place the data into the window by means of a vector of items.

We will see that this enables further aggregations based on the data accumulated in each window, which is a reasonably common requirement.

To complete the code, here is the function to generate the windows

{% highlight clojure %}
(defn- gen-window [start-time open-duration]
  {:pre [(> open-duration 0)]}
  (let [to-time (t/plus start-time (t/seconds open-duration))]
    (assoc {} :from start-time :to to-time :closed false :items [])))
{% endhighlight %}

# Merging time and data

{% highlight clojure %}
(defn data-in-timed-series
  "Add timed data from item-ch to the time series windows produced in the window-ch;
   emit the window once only, after it is closed"
  [item-ch window-ch]
  (let [out-ch (chan 1 (comp (map (fn [windows]
                                    (filter #(:closed %) windows)))
                             (filter (fn [windows] (> (count windows) 0)))))]
    (go-loop [active-windows ()]
      (if-let [[data chan] (alts! [item-ch window-ch])]
        (condp = chan
          window-ch (if-let [windows (maintain-active-windows (conj active-windows data))]
                      (do
                        (>! out-ch windows)
                        (recur windows)))
          item-ch (recur (add-timed-item-to-windows data active-windows)))
        (close! out-ch)))
    out-ch))
{% endhighlight %}

This function creates a `go-loop` that combines data from channels on which the windows and time-stamped data are emitted.

When the data is incoming from the `item-ch` the items are added to appropriate window(s) using `add-timed-item-to-windows` and we `recur` on the result.

When the data comes in from the `window-ch` the new window is added to the list. We will show the `maintain-active-windows` function shortly. 

Finally, the transducer limits the output to closed, non-empty windows.

# Time management

Again, no cats skinned but there are choices. In this case I chose to maintain the windows by tracking a boolean and then reaping the windows after a certain time threshold. Here is the code abstracted out into a small function:

{% highlight clojure %}
(defn- maintain-active-windows [windows]
  (let [now (t/now)
        retention-period (t/millis 500)
        retention-boundary (t/minus now retention-period)
        retained (filter #(t/after? (:to %) retention-boundary) windows)
        to-be-closed (filter #(and (t/before? (:to %) now) (false? (:closed %))) windows)
        closing (map #(assoc % :closed true) to-be-closed)]
    (concat retained closing)))
{% endhighlight %}

I chose a retention period of 500ms on the basis that 1000ms is the minimum window size in this design. This way I am always guaranteed to clean up on each new time window. 

It also means that there is some small additional time for catching stragglers although that is the limit of any effort to deal with that particular problem. In general stragglers are silently dropped.

# Data management

The code for adding items to the windows is fairly boiler plate but presented here so that you can have a more complete view:

{% highlight clojure %}
(defn- within-interval? [from to time]
  {:pre [(t/before? from to)]}
  "Check whether a time is within an interval"
  (let [interval (t/interval from to)]
    (t/within? interval time)))

(defn- add-timed-item-to-windows [timed-item windows]
  "Add an item to the windows where the time intervals match"
  (if-let [[time item] timed-item]
    (let [matching-windows (filter #(within-interval? (:from %) (:to %) time) windows)
          updated-windows (map #(assoc % :items (conj (:items %) item)) matching-windows)]
      updated-windows)))
{% endhighlight %}

The thing I like about this code is that we have avoided creating global state.

# Demo time

{% highlight clojure %}
(finite-printer (stop-after-n-seconds 10) (data-in-timed-series (gen-timed-orders 1000 parts) (create-time-series-windows 4)))
=>
#object[clojure.core.async.impl.channels.ManyToManyChannel
        0x2b730646
        "clojure.core.async.impl.channels.ManyToManyChannel@2b730646"]
({:from
  #object[org.joda.time.DateTime 0x6a21c46e "2016-02-14T20:31:28.404Z"],
  :to
  #object[org.joda.time.DateTime 0x51d383fa "2016-02-14T20:31:32.404Z"],
  :closed true,
  :items
  [{:id G__12317, :description "Door control module9"}
   {:id G__12077, :description "Decklid9"}
   {:id G__12382, :description "Fuel tank (or fuel filler) door4"}
   {:id G__12092, :description "Fender (wing or mudguard)4"}]})
({:from
  #object[org.joda.time.DateTime 0x51d383fa "2016-02-14T20:31:32.404Z"],
  :to
  #object[org.joda.time.DateTime 0x66a36cd7 "2016-02-14T20:31:36.404Z"],
  :closed true,
  :items
  [{:id G__12171, :description "Roof rack3"}
   {:id G__12055, :description "Exposed bumper7"}
   {:id G__12428, :description "Window regulator0"}
   {:id G__12167, :description "Rocker9"}]})
:stop
{% endhighlight %}

# Aggregations

Here is a small general purpose aggegrator that take a function to operate on each window:

{% highlight clojure %}
(defn interval-aggregator [aggregator in]
  "Execute the provided aggregator against the :items property from the map on the channel"
  (let [out (chan)]
    (go-loop []
      (if-let [window (first (<! in))]
        (do
          (let [results (aggregator (:items window))]
            (>! out results))
          (recur))
        (close! out)))
    out))


(finite-printer (stop-after-n-seconds 10) (interval-aggregator count (data-in-timed-series (gen-timed-orders 1000 parts) (create-time-series-windows 4))))
=>
#object[clojure.core.async.impl.channels.ManyToManyChannel
        0x77069406
        "clojure.core.async.impl.channels.ManyToManyChannel@77069406"]
3
4
:stop
{% endhighlight %}

# Summary

So that's our final experiment with aggregating over time series.

The amount of code is pleasantly small and we can see many possibilities for playing with time series data.

# Conclusions

This really has been me scraping the surface of core.async by trying to scratch a few itches. I found the model quite straightforward to use and extremely powerful. I will continue to experiment with the library and write up some samples as further revelations unfold!

# And finally - Thanks!
 
Thanks for making it through, especially of you ploughed through the series!
 
I have a better understanding of core.async after writing this and I hope that's true for you too!
 
Zing me or ping me if this was useful via Twitter or in the comments below.

[core-async-state]: {% post_url 2016-02-09-event-streams-core-async-state %}
[jgd-gist]: https://gist.github.com/jgdavey/d928136d035645bd15ec

{% include disqus.html %}
