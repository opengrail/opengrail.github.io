---
layout: post
title:  "Event Stream Processing - Managing state with core.async"
date:   2016-02-09 19:59:38 +0100
comments: true
categories: clojure events streams core.async
---

# Introduction

In my [last post][core-async-intro-post] I went through the code for a data flow using a product stock level tracker. It had a simple filter to ensure that the notifications occurred when a specific business rule was fired.

In that post I showed the basic mechanisms for creating and connecting channels. In this post I want to show how to combine data from multiple streams. To demonstrate this we will implement the processes around the Stock Management Tracker. 

![Simple stock management]({{ url }}/assets/Stock-Tracking-Streams-3.jpg)

In the tracker we will join the two streams to emit a running total of stock on hand vs orders at any point in time.

# Data first

Let us just quickly outline the data in our model:
{% highlight clojure %}
; Order
{:item-id A123 :description "Windscreen wiper" :quantity 2 :customer-id C234}
; Delivery
{:item-id A123 :description "Windscreen wiper" :quantity 40 :supplier-id S234}
{% endhighlight %}

Once again this is certainly a simple model but is sufficient to demonstrate the power of the solution.

To help with the simulation we will throw together some sample data and create a few small functions to generate the orders and deliveries.

# Parts data generation

{% highlight clojure %}
; Wikipedia sourced list of body and main parts
(def manifest ["Bonnet/hood" "Bonnet/hood latch" "Bumper" "Unexposed bumper" "Exposed bumper"
               "Cowl screen" "Decklid" "Fascia rear and support" "Fender (wing or mudguard)"
               "Front clip" "Front fascia and header panel" "Grille (also called grill)"
               "Pillar and hard trim" "Quarter panel" "Radiator core support"
               "Rocker" "Roof rack" "Spoiler" "Front spoiler (air dam)" "Rear spoiler (wing)" "Rims" "Hubcap"
               "Tire/Tyre" "Trim package" "Trunk/boot/hatch" "Trunk/boot latch" "Valance" "Welded assembly"
               "Outer door handle" "Inner door handle" "Door control module" "Door seal"
               "Door watershield" "Hinge" "Door latch" "Door lock and power door locks" "Center-locking"
               "Fuel tank (or fuel filler) door" "Window glass" "Sunroof" "Sunroof motor" "Window motor"
               "Window regulator" "Windshield (also called windscreen)" "Windshield washer motor"
               "Window seal"])

; magic up 460 items with variants for each part (adjust the range to create more / less)
(def parts (mapcat (fn [part]
                     (map #(assoc {} :id (gensym) :description (str part %)) (range 10)))
                   manifest))
{% endhighlight %}

# Order and delivery generation

{% highlight clojure %}
(defn gen-timed-orders [frequency-millis coll]
  (let [out (chan)]
    (go-loop []
      (do
        (>! out [(t/now) (rand-nth coll)])
        (<! (timeout frequency-millis))
        (recur)))
    out))

(defn gen-random-deliveries [max-wait-millis coll]
  (let [out (chan)]
    (go-loop []
      (do
        (>! out [(t/now) (rand-nth coll)])
        (<! (timeout (rand-int max-wait-millis)))
        (recur)))
    out))
{% endhighlight %}

Here we generate infinite streams of data with time stamps. That will be used in later examples to aggregate data into time series.

They differ only in the way that orders are consistent and deliveries are random. This is not a perfect model of the real world (yeah, I know) but is good enough for this purpose.

# Short demo

{% highlight clojure %}
(def stock-levels-ch (stock-levels orders deliveries))

(finite-printer (stop-after-n-seconds 2) stock-levels-ch)

=> #'core-async-examples.stock-management-tracker/stock-levels-ch
=>
#object[clojure.core.async.impl.channels.ManyToManyChannel
        0x6e923694
        "clojure.core.async.impl.channels.ManyToManyChannel@6e923694"]
{:id G__11226, :count -1}
{:id G__11584, :count 1}
{:id G__11422, :count -1}
{:id G__11408, :count -1}
{:id G__11579, :count 1}
{:id G__11223, :count -1}
{:id G__11288, :count -1}
{:id G__11656, :count -1}
{:id G__11333, :count 1}
{:id G__11425, :count -1}
{:id G__11666, :count -1}
{:id G__11561, :count -1}
{:id G__11642, :count -1}
{:id G__11584, :count 1}
:stop
{% endhighlight %}

# Blocking without Thread/sleep == SCAAAAALE!

One aspect to note in the above code is the `timeout` function. It reads from a virtual channel and effects a pause on the generation like Thread/sleep but does not block.

Not blocking means that it is OK to start 100s or 1000s of go blocks with very little CPU or RAM overhead. This is similar in effect to the way that **NGINX** is architected.

# Don't change that channel

I'm going to show this next function in two forms. The first form shows how to use conditionals based on channel

{% highlight clojure %}
(defn stock-levels [orders deliveries]
  "Stock levels - also includes demand (negative stock numbers)"
  (let [out (chan)
        dec-stock (partial modify-stock dec)
        inc-stock (partial modify-stock inc)]
    (go-loop [stock {}]
      (if-let [[data chan] (alts! [orders deliveries])]
        (let [[_ item] data
              update-operation (condp = chan
                                 orders dec-stock
                                 deliveries inc-stock)]
          (do
            (>! out (update-operation stock item))          ; does not work - need to manage 'stock'
            (recur stock)))
        (close! out)))
    out))
{% endhighlight %}
    
In this first showing of the code we see the ability to read many channels at once using `alts!` which receives the data and the channel that was read.

In this example we use `condp` to run differing code based on the channel that was read. In this case we set the method to adjust stock.

# Re-Stating our intentions

The gap we have with the above code is that it will only ever report 0 or 1. 

That might might be useful if have an external aggregator but we can also aggregate in the go-loop directly:

{% highlight clojure %}
(defn stock-levels [orders deliveries]
  "Stock levels - also includes demand (negative stock numbers)"
  (let [out (chan)
        dec-stock (partial modify-stock dec)
        inc-stock (partial modify-stock inc)]
    (go-loop [stock (into #{} (map #(assoc {} :id (:id %) :count 0) parts))]
      (if-let [[data chan] (alts! [orders deliveries])]
        (let [[_ item] data
              update-operation (condp = chan
                                 orders dec-stock
                                 deliveries inc-stock)]
          (if-let [[modified-stock updated-item] (update-operation stock item)]
            (do
              (>! out updated-item)
              (recur modified-stock))))
        (close! out)))
    out))
{% endhighlight %}

Using the parameters to the `go-loop` we can initiate state and then maintain it via `recur`. 

In this case we create a set of parts and set each stock item count to 0 when our function starts. One can imagine more complex initialisations!

In the loop we then pass the stock collection to the `modify-stock` function which returns a new version of the stock collection.

Here we exploit the fact that Clojure collections are **very efficient** with respect to minimising the costs of each new version. 

This stock list could easily be scaled to tens of thousands of parts without adding any latency costs (barring side effects of swapping although luckily we are no longer limited to 32k, 32Mb or even 32Gb of memory like in the good old days). For more information [see this rich visual explanation][data-structures] from  Jean Niklas L'orange.

# Short Demo Two

{% highlight clojure %}
(def order-chan (async/mult (gen-timed-orders 10 parts)))  ; use mult to allow many readers

(def deliveries-chan (gen-random-deliveries 1000 parts))

; show what's happening with stock
(def stock-chan (stock-levels (async/tap order-chan (chan)) deliveries-chan))
(def backlogs-chan (detect-backlogs -4 stock-chan))
(finite-printer (stop-after-n-seconds 20) backlogs-chan) ; 20k orders will be processed

=> #'core-async-examples.stock-management-tracker/order-chan
=> #'core-async-examples.stock-management-tracker/deliveries-chan
=> #'core-async-examples.stock-management-tracker/stock-chan
=> #'core-async-examples.stock-management-tracker/backlogs-chan
=>
#object[clojure.core.async.impl.channels.ManyToManyChannel
        0x694db469
        "clojure.core.async.impl.channels.ManyToManyChannel@694db469"]
{:id G__11288, :count -5}
{:id G__11338, :count -5}
{:id G__11603, :count -5}
{:id G__11338, :count -5}
:stop
{% endhighlight %}

# Modify without side effects

To finalise the example here is the code for `modify-stock` which runs the provided function to obtain the new value using the existing count on the item.

{% highlight clojure %}
(defn modify-stock [count-adjust-fn stock item]
  (let [current-value (first (clojure.set/select #(= (:id %) (:id item)) stock))
        new-value (assoc current-value :count (count-adjust-fn (:count current-value)))
        updated-stock (conj stock new-value)]
    [updated-stock new-value]))
{% endhighlight %}

One nice aspect of using Clojure's `set` data structure is that we can use `conj` as a succinct upsert.

# Summary

In this example we saw how to implement a stock level tracker. To achieve this functionality we read multiple channels in one `go-loop` and distinguished data from those channels. Further we maintained aggregations and state around the `go-loop` to increase power and simplicity.

The code is all on [my GitHub][ray-repo].

# Next next next

In the [final example][core-async-time] of this short series I will show how to track stream data within series of time.

# And finally - Thanks!
 
Thanks for making it through. I have a better understanding of core.async after writing this and I hope that's true for you too!
 
Zing me or ping me if this was useful via Twitter or in the comments below.

[core-async-intro-post]: {% post_url 2016-02-06-event-streams-core-async %}
[data-structures]: http://hypirion.com/musings/understanding-persistent-vector-pt-1
[core-async-time]: {% post_url 2016-02-11-event-streams-core-async-time-series %}
[ray-repo]: https://github.com/raymcdermott/core-async-examples

{% include disqus.html %}
