---
layout: post
title:  "Numbers as text"
date:   2017-04-17 01:00:00 +0100
categories: Clojure ClojureScript Functional
---

<link rel="stylesheet" type="text/css" href="http://app.klipse.tech/css/codemirror.css">

<script>  
  window.klipse_settings = {
        selector: '.language-klipse'
      };
</script>


## Random Beyonce based headlines, as if you care

I was chatting to a friend about job interviews and he mentioned that he always gives a small challenge to the candidates that they can do at home.

The goal is to spell out numbers as sentences, similar to how text might appear on a bankers cheque (or check if you're in the US and - bonus - it's still relevant for you).

Here is the behaviour that the program should provide as a minimum:

~~~shell
(num->text 123456789)
=> "One hundred and twenty three million four hundred and fifty six thousand seven hundred and eighty nine"
~~~

The challenge is meant to be relatively simple and serve as a worked example to centre discussions around design and development approaches.

I figured that addressing this problem with Clojure could make an interesting blog post for the same reasons. The code is on [Github][ray-github]

This post is enhanced using [KLIPSE][klipse] live code snippets meaning **you** can edit the code and see immediate results

## All the single ladies

Hard code the single numbers you say. No way. Hard code the names at least you say. Again no way. Beyonce would not approve.

Let use key words, sequences and combinations to get straight into it the good FP stuff right from the start...

~~~klipse
(def x1 [:one :two :three :four :five :six :seven :eight :nine
         :ten :eleven :twelve :thirteen :fourteen :fifteen :sixteen :seventeen :eighteen :nineteen])
(def singles (zipmap (range 1 20) x1))
; simple interactive test, try it yourself...
(get singles 7)
~~~

Clojure make it easy to generating numbers (**`range`**) and :key-words are easy to test.

**`zipmap`** is a simple way to create a map from two sequences - using the first as keys and the second as values.

So we have the computer doing it's share of the work already. Neat!

## Now put your hands up

Next we deal with the other fixed numbers >= 20 and < 100

<pre><code class="language-klipse" data-loop-msec="2000">
(def x10 [:twenty :thirty :forty :fifty :sixty :seventy :eighty :ninety])
(def tens (zipmap (range 20 99 10) x10))
(get tens (rand-nth (keys tens)))
</code></pre>

Here we use another form of the **`range`** function again to produce a range that starts at 20, ends at 99 and is stepped by 10.

The interactive sample introduces a little randomisation for the LOLs and we will use more of this as we go along.

> The refresh on the randomisation is a little magic from `Klipse` that I use to set a loop every 2 seconds in the HTML mark up.

## I'm doing my own little thing

My solution is composed from three logical parts: numbers to 100, numbers between 100 and 1000 and numbers above 1000.

<pre><code class="language-klipse" data-loop-msec="2000">
(defn nums-lt-100
  [num]
  {:pre [(pos? num)]}
  (if-let [answer (or (get singles num)
                      (get tens num))]
    [answer]
    (let [ten-part  (quot num 10)
          ten-whole (* 10 ten-part)]
      [(get tens ten-whole) (nums-lt-100 (- num ten-whole))])))

; Generate a random sample of numbers to test it out
(map (fn [n] [n (nums-lt-100 n)])
     (take 2 (random-sample 0.05 (range 1 100))))
</code></pre>

You can tweak the above code to adjust the number of samples (take N). By the way, I only take a small sample so that the page doesn't reflow every two seconds ;-)

The first part of the code (if-let) tests if the answer is present in the singles or tens collection - there is no need to take the risk of testing for a sentinel value. Such values can easily get out of sync with the size of the vector or require maintenance. Also, because it's a map no need to test for length - just look it up and find it or move along.

If we don't find it, we take the modulus and look up the answer in one or both sets.

Finally, we use a some Clojure syntax sugar to coerce results into a vector by surrounding the responses with square brackets **`[ ]`**

Building the answer as a sequence is a huge simplification compared to the other solutions I have seen out there. This will become more evident as the solution is further composed.

## You decided to dip and now you wanna trip

The second part deals with the hundreds...

<pre><code class="language-klipse" data-loop-msec="2000">
(defn nums-gt-100-lt-1000
  [num]
  {:pre [(>= num 100) (< num 1000)]}
  (let [hundreds  (quot num 100)
        remainder (- num (* hundreds 100))
        result    [(get singles hundreds) :hundred
                   (if-not (zero? remainder)
                     (nums-lt-100 remainder))]]
    (remove nil? result)))

(map #(nums-gt-100-lt-1000 %)
     (take 3 (filter odd? (random-sample 0.5 (range 921 1000)))))
</code></pre>

The main function does some trivial maths to separate out the hundreds and its remainder. The hundreds component is added to the vector and then we use the earlier function on any remainder.

>There is a small annoyance to my eye with this solution ... I have to use `(quot num 100)` rather than `(/ num 100)` as the default answer in Clojure is a rational number. You can coerce it an integer but I'm not a fan of using such type trickery unless it's absolutely unavoidable.

## I got gloss on my lips, a man on my hips

And now I'm ready to deal with the big boys: thousands, millions and so on... to quattuordecillion. We can go higher with a longer list but this makes the samples too awkward. [You can find a longer list up to millinillion on the GitHub repo.]

This is the third part of the solution and is also based on sequences of generated numbers.

~~~klipse
(defn units [start step count]
  (reduce (fn [a b] (conj a (* (last a) step)))
          [start] (range count)))

(units 1 2 4)
~~~

## Got me tighter than my Dereon jeans

The units function provides a way to generate lists of numbers at certain boundaries and we can will use that to establish the various 'large number' magnitudes.

<pre><code class="language-klipse" data-loop-msec="2000">
(def large-numbers-text [:thousand :million :billion :trillion
                         :quadrillion :quintillion :sextillion
                         :septillion :octillion :nonillion
                         :decillion :undecillion :duodecillion
                         :tredecillion :quattuordecillion])

(def unitable (units 1000N 1000N (dec (count large-numbers-text))))

(def large-number-map (zipmap unitable large-numbers-text))
(def inverse-large-numbers-map (into {} (map (fn [[a b]] [b a])) large-number-map))

(def unit-boundaries (map (fn [u] [u (* u 1000N)]) unitable))

(let [rand (rand-nth (keys large-number-map))]
  [rand (get large-number-map rand)])
</code></pre>

We use the same `zipmap` pattern for large number texts as before...

In addition to the previous pattern, we also generate an inverse look up map that we will need later.

The interesting part of this approach is that we generate all of the boundary conditions for all supported units.

## Acting up, drink in my cup

~~~klipse
(defn which-bounds? [num unit-bounds]
  (remove nil? (map (fn [[lower upper]]
                      (if (and (>= num lower)
                               (< num upper))
                        lower)) unit-bounds)))

(defn which-unit? [num]
  (let [unit (which-bounds? num unit-boundaries)]
    (get large-number-map (first unit))))

(def various (units 1000 7 7))

(map #(which-bounds? % unit-boundaries) various)
~~~

The code to test for the boundaries, and to map it into the correct unit, is a straightforward mapping check over the boundaries.

We use the units function to generate a sequence to test it out. You can edit the code above to see the test data and validate it.

## I can care less what you think

With this in place we can present the final form that composes it together.

~~~klipse
(defn inject-and
  [num-vec]
  (if (<= (count num-vec) 2)
    num-vec
    (let [check-map (assoc inverse-large-numbers-map :hundred 100)]
      (cond
        (= :and (last (drop-last 1 num-vec))) num-vec
        (get check-map (last num-vec)) num-vec
        (get check-map (last (drop-last 1 num-vec)))
            (concat (drop-last 1 num-vec) [:and] (take-last 1 num-vec))
        (get check-map (last (drop-last 2 num-vec)))
            (concat (drop-last 2 num-vec) [:and] (take-last 2 num-vec))
        :else num-vec))))

(defn num-representation
  [num]
  (inject-and
    (flatten
      (cond
        (< num 100) (nums-lt-100 num)
        (< num 1000) (nums-gt-100-lt-1000 num)
        :else (let [unit           (which-unit? num)
                    divisor        (unit inverse-large-numbers-map)
                    first-part     (quot num divisor)
                    remainder      (- num (* divisor first-part))
                    representation (if (zero? remainder)
                                     [(num-representation first-part) unit]
                                     [(num-representation first-part)
                                      unit
                                      (num-representation remainder)])]
                representation)))))

(num-representation 100101)
~~~

**`inject-and`** finds the right spot in the vector to inject the and. Since the call site is recursive, it applies properly across all units. Such baroqueness is not needed in the US version so this function could be easily removed.

The trick in the `num-representation` function is to always have the larger numbers feed back through the function so that each part is processed - from left to right - into the appropriate single, ten or 100 multiplier.

## I need no permission, did I mention?

There is one final part at the edge to morph the data into strings...

~~~klipse
(defn num-vec->text
  [num-vec]
  (clojure.string/capitalize
    (apply str (interpose " " (map name num-vec)))))

(defn num->text
  [num]
  (num-vec->text (num-representation num)))

(num->text 56789)
~~~

## If you liked it, then you should have put a ring on it

That's just the beginning though. This problem is one where the range of answers is so huge that we cannot possibly test all of them. Moreover tests on strings presume a human checker.

So we can do better than that. Next time I want to go into using spec on this problem but I will just finish with a few data samples for tests that I have concocted. You can see more on the GitHub repo.

## Oh, oh, oh

CONSISTENCY check: Do all of the first numbers, across all units start with :one ?

~~~klipse
(let [sample (units 1000N 1000N (dec (count large-numbers-text)))]
  (distinct (map #(first (num-representation %)) sample)))
~~~

## Oh, oh, oh, oh, oh, oh

CONSISTENCY check: Is the answer for all of the first numbers, across all units, two words ?

~~~klipse
(let [sample (units 1000N 1000N (dec (count large-numbers-text)))]
  (distinct(map #(count (num-representation %)) sample)))
~~~

## Oh, oh, oh

What does it mean?

- Stay with data - it's easier to work with and to test

- Layer on infra - to enable testing on different layers

- Use the latest possible transforms at the edge of the system

- Use randomness early and often in your feedback loop

- Use properties and constraints to properly test your system

Developing with data, sequences and composing functions to manipulate them is how we will finally get to leave the sand pit.

## Oh, oh, oh

US readers please note: in the US the grammar for expressing these numbers is different and simpler: there's no 'and'.

Providing this facility for all cultures and languages is not trivial and it reminds us that even such a small and simple example like this - like most any thing - is most often a subset of a much more complex domain.

## 'Cause if you liked it, then you should have put a ring on it

Disclaimer: Me != maths

Please let me know if you are a maths whizz and know a simpler way to solve this.

Also, if you have another functional way, in Clojure, JS or Haskell or Scala or Java 8.

This has been fun but if there is a 'right way', I would be happy to learn it!

# There is a right way ...

Ha ha ... and to prove this point Alex Miller (aka [@puredanger][alex]) informed me on Twitter that **Clojure** has something very close to this already - and soooooo much more - in the form of [cl-format][cl-format]. This is an implementation of output formats from Common Lisp. Who knew? Wow - it's astounding and it was honestly worth this effort to discover that it exists! This is a whole new rabbit hole people!

<pre><code class="language-klipse" data-loop-msec="3000" data-preamble="(require 'clojure.pprint)">
(clojure.pprint/cl-format true "~r" 100101)
</code></pre>


[klipse]: http://blog.klipse.tech/clojure/2016/03/30/destructuring.html
[ray-github]: https://github.com/raymcdermott/num-text
[cl-format]:http://clojure.github.io/clojure/clojure.pprint-api.html#clojure.pprint/cl-format
[alex]:https://twitter.com/puredanger

<script src="http://app.klipse.tech/plugin/js/klipse_plugin.js?"></script>



{% include disqus.html %}
