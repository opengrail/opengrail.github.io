---
layout: post
title:  "Nil points"
date:   2017-02-21 18:00:00 +0100
categories: Clojure ClojureScript Functional
---

<link rel="stylesheet" type="text/css"  
  href="http://app.klipse.tech/css/codemirror.css">

<script>  
  window.klipse_settings = {
        selector: '.language-klipse'
      };
</script>

![Nil points]({{ url }}/assets/nil-points.jpg)

<sub style="color:gray">Eurovision memories from back in the day</sub>
<sub style="color:lightgray"> (image source [bbc.co.uk][nil-points]) </sub>

# Visualising Clojure

I recently visited a [splendid blog by Joseph Wilk][patterns] that explains Clojure sequence functions by visualising their effects.

After being impressed by the visuals I was struck by a couple of the functions (`nthnext` and `nthrest`) that are shown to have the same effect on a non-empty sequence.

This post is supported by live code snippets **and that you can live edit too**:

~~~klipse
(nthnext [0 1 2 3] 2)
~~~

~~~klipse
(nthrest [0 1 2 3] 2)
~~~

I was not familiar with them so I started digging a bit deeper. They are based, possibly obviously, on `next` and `rest`. The only difference is their return value when the sequence is nil or empty.

~~~klipse
(let [nil-rest-gives (rest nil)
      empty-rest-gives (rest [])]
  [nil-rest-gives empty-rest-gives])
~~~
~~~klipse
(let [nil-next-gives (next nil)
      empty-next-gives (next [])]
  [nil-next-gives empty-next-gives])
~~~

# Seq and ye shall find

In turn the docs say they are both are based on `seq`.

Looking at the code for `next` and `rest` we see how this is achieved: they are wrapped in method calls to native code.

Here are the Clojure definitions, stripped of most metadata for brevity:

~~~clojure
(def
 ^{:doc "Returns a seq of the items after the first. Calls seq on its
  argument.  If there are no more items, returns nil."}  
 next (fn ^:static next [x] (. clojure.lang.RT (next x))))

(def
 ^{:doc "Returns a possibly empty seq of the items after the first.
 Calls seq on its argument."}  
 rest (fn ^:static rest [x] (. clojure.lang.RT (more x))))
~~~

# Swan dive

Diving deeper, `next` and `more` are written in Java, so let's sneak a view behind the curtain:

~~~java
public static ISeq next(Object x) {
  if(x instanceof ISeq) {
    return ((ISeq)x).next();
  } else {
    ISeq seq = seq(x);
    return seq == null? null : seq.next();
  }
}

public static ISeq more(Object x) {
  if(x instanceof ISeq) {
    return ((ISeq)x).more();
  } else {
    ISeq seq = seq(x);
    return (ISeq)(seq == null? PersistentList.EMPTY : seq.more());
  }
}
~~~

The significant difference is that when `rest` calls into `more`, it will return `PersistentList.EMPTY` if the call to `seq` is a Java null. The `next` implementation instead returns the null.

Well, that's cool: having drilled right the way down we can see precisely why the behaviour differs.

# There maybe reason

So what does it mean to Clojure programmer? Having a variant that returns nil might seem strange in the FP world where [Tony Hoare's nillion dollar mistake][nillion] has been, ahem, fixed.

In other languages there are no more accidental nils once we have the monadic finery afforded by types like `Option`, `Maybe`, `Coulda`, `Shoulda` and `Woulda`. Yes, I did invent those last three to see if you're still with me.

# Functionil Streaming

We don't have `Maybe` or `Option` (or other static) types in Clojure so let's see how this works in practice.

Let's start with a trivial - even silly - chain of map / filter over a sequence.

~~~klipse
(map inc (nthnext (filter even? (range 200)) 90))
~~~

Now lets write a function that does not do well with nils

~~~klipse
(defn next-nil-fail [x] (if (= 0 x) 0 (/ x x)))
; (next-nil-fail 10)
(next-nil-fail nil)
~~~

Now, let's throw that function in a working chain and see what happens

~~~klipse
(defn next-nil-fail [x] (if (= 0 x) 0 (/ x x)))
(map next-nil-fail (nthnext (filter even? (range 200)) 90))
~~~

And then let's see what happens when we start throwing nils

~~~klipse
(defn next-nil-fail [x] (if (= 0 x) 0 (/ x x)))
; filter returns 100 values so 101 > 100 and nthnext will return a nil
(map next-nil-fail (nthnext (filter even? (range 200)) 101))
~~~

# Nothing will come of nothing

**Say what?** How in the Lear didn't it generate an error?

~~~klipse
(map next-nil-fail [nil])
~~~

We can see that `nil-fail` will fail if it is passed a nil or a collection of nils.

So something funny going is on here! Let's look at how `map` works:

~~~clojure
; relevant code snippet from map
([f coll]
 (lazy-seq
  (when-let [s (seq coll)]
    (if (chunked-seq? s)
      (let [c (chunk-first s)
            size (int (count c))
            b (chunk-buffer size)]
        (dotimes [i size]
            (chunk-append b (f (.nth c i))))
        (chunk-cons (chunk b) (map f (chunk-rest s))))
      (cons (f (first s)) (map f (rest s)))))))
~~~

And now we see: the last line shows that the map implementation uses `rest` to **avoid producing nils on the stream when the seq is empty or nil**.

But it does not inspect the content of each element so cannot stop us from hitting all fail cases.

~~~klipse
(interpose nil [ 1 2 3])
~~~

The core functions generally make it possible to ignore the nil values

~~~klipse
(every? int? (interpose nil [ 1 2 3]))
~~~

In other cases, you can just clean them out

~~~klipse
(remove nil? (interpose nil [ 1 2 3]))
~~~

Or take the idiomatic route and fix up your code

~~~klipse
(defn div-nil-ok [x]
  (cond (nil? x) x
        (= 0 x) 0
        :else (/ x x)))
(div-nil-ok nil)
~~~

# Monadic puns

Is nil a seq in Clojure?

~~~klipse
(seq? nil)
~~~

No, the implementation of `map` uses a monadic approach to ensure that nils are not propagated through the chain.

Other `seq` functions take the same approach to ensure that functions which generate nil instead of empty lists are treated equally.

This is also called [nil-punning][punning]. Oh and one small niggle with that excellent article: Eric claims

> first has nothing to return if the seq is empty, and so it returns nil. Because nil is a seq, first works on nil, and returns nil. rest works on nil as well, because nil is a seq.

Naughty Eric ;-) We see that `nil` is *not a seq* but instead, the operations on sequences work to provide that effect.

# But but but next

Ah, yes in the case of `next` and its derivatives, the sequence functions make no such concessions.

So why would such functions exist?

# Functional in a Recursive style

Nil can be used to good effect as a terminal condition when writing recursive programs.

~~~klipse
(loop [nums (range 10)
       results []]
  (if nums
    (recur (nthnext nums 2) (conj results (first nums)))
    results))
~~~

**Do not** change `nthnext` for `nthrest` ;-)

# Conclusion

Clojure has made working with nil pain free in general due to the design of the sequence operations.

There are however still some edge cases that need to be addressed where nils can be inserted into sequences.

There are also many options (or enough rope) for adventurous programmers to use them where they have value.

# Credits

This post was written using the power of [KLIPSE][klipse] to enable the interactive code blocks.

[nil-points]: http://news.bbc.co.uk/2/hi/entertainment/2935874.stm
[punning]: http://www.lispcast.com/nil-punning
[nillion]: https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare
[patterns]: http://blog.josephwilk.net/clojure/functions-explained-through-patterns.html
[klipse]: https://github.com/viebel/klipse

<script src="http://app.klipse.tech/plugin/js/klipse_plugin.js?"></script>



{% include disqus.html %}
