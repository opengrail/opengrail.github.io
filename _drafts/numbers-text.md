---
layout: post
title:  "Numbers as text"
categories: Clojure ClojureScript Functional
---

<link rel="stylesheet" type="text/css"  
  href="http://app.klipse.tech/css/codemirror.css">

<script>  
  window.klipse_settings = {
        selector: '.language-klipse'
      };
</script>

## Friendly challenge

I was chatting to a friend about job interviews and he mentioned that he always gives a small challenge to the candidates that they can do at home.

The goal is to spell out numbers as sentences, similar to how text might appear on a bankers cheque (or check if you're in the US and - bonus - it's still relevant for you).

Here is the behaviour that the program should provide as a minimum:

|**number**|**text**|
|======|====|
| 8 | eight|
| 79 | seventy nine|
| 619 | six hundred and nineteen|
| 5 919 | five thousand, nine hundred and nineteen |
| 49 191 | forty nine thousand, one hundred and ninety one |
| 391 919 | three hundred and ninety one thousand, nine hundred and nineteen |
| 2 919 191 | two million, nine hundred and nineteen thousand, one hundred and ninety one |
| 89 191 919 | eighty nine million, one hundred and ninety one thousand, nine hundred and nineteen |
{:.mbtablestyle}

<br/>
The challenge is meant to be simple and serve mostly as a worked example to centre discussions around design and development approaches.

I figured that this small challenge could make an interesting blog post for the same reasons.

This post is enhanced using [KLIPSE][klipse] live code snippets meaning **you** can edit the code and see immediate results

## All the single ladies

Hard code the single numbers you say. No way. Hard code the names at least you say. Again no way.

Let use key words, sequences and combinations to get straight into it the good stuff right from the start...

~~~klipse
(def words [:one :two :three :four :five :six :seven :eight :nine :ten])
(def singles (zipmap (range 1 11) words))
; ask the REPL to print the definition
singles
; simple interactive test, try it yourself...
(get singles 7)
~~~

Computers are good at generating numbers and key words are easy to manipulate.

zipmap is a simple way to create a map from two sequences - using the first as keys and the second as values.

Neat, eh?

~~~klipse
(def words [:one :two :three :four :five :fix-me :seven :eight :nine :ten])
(def singles (zipmap (range 1 11) (map name words)))
; ask the REPL to print the definition
singles
(get singles 6)
~~~

Here we transform the keys to words - to show that off. But there is a bug in the form.

It is simple to detect and fix. Do it :)

## Teenage thrills

Let's complete the form for the fixed, low numbers:

~~~klipse
(def words [:one :two :three :four :five :six :seven :eight :nine :ten
            :eleven :twelve :thirteen :fourteen :fifteen :sixteen :seventeen :eighteen :nineteen])
(def singles (zipmap (range 1 20) words))
; take a random key and get the result
(get singles (rand-nth (keys singles)))
~~~

Next we deal with the other fixed numbers >= 20 and < 100

~~~klipse
(def x10 [:twenty :thirty :forty :fifty :sixty :seventy :eighty :ninety])
(def tens (zipmap (range 20 99 10) x10))
; take a random key and get the result
(get tens (rand-nth (keys tens)))
~~~

Here we use the computer to produce a range that is stepped by 10.

Bringing it together we can emit up to 100 numbers as text with ease...

~~~klipse
(defn starters
  [num]
  (if-let [answer (get singles num)]
    [answer]
    (let [modulus (mod num 10)
          tenny   (- num modulus)]
      (if (zero? modulus)
        [(get tens tenny)]
        [(get tens tenny) (get singles modulus)]))))

; Generate a random sample of numbers to test it out
(map (fn [n] [n (starters n)])
     (random-sample 0.05 (range 1 100)))
~~~

You can tweak the above code to adjust the number of samples.

The first part of the code (if-let) tests if the answer is present in the singles collection - there is no need to take the risk of testing for a sentinel value. Such values can easily get out of sync with the size of the vector or require maintenance. Also, because it's a map no need to test for length - just look it up and find it or move along.

If we don't find it, we take the modulus and look up the answer in one or both sets.

Finally, we use a some Clojure syntax sugar to coerce results into a vector by surrounding the responses with square brackets **`[ ]`**

Having the answer as a sequence makes it easy to select or compose standard functions to further manipulate the results.

## Hundreds and Thousands

So now to deal with the hundreds, thousands and so on...

~~~klipse
(defn units [start step count]
  "Generate a list of n numbers with a start and step multiplier"
  (reduce (fn [a b] (conj a (* (last a) step)))
          [start] (range count)))

(units 1 2 4)
~~~

The units function provides a way to generate lists of numbers at certain boundaries. We would like to test that numbers sit between various magnitudes.

We could hard code all of this but why should we always reach for the dull choice?

We do need the large number text though so we follow the same zipmap pattern as before...

~~~klipse
(def large-numbers-text [:thousand :million :billion :trillion
                         :quadrillion :quintillion :sextillion
                         :septillion :octillion :nonillion
                         :decillion :undecillion :duodecillion
                         :tredecillion :quattuordecillion])

(def large-number-map (zipmap (units 1000 1000 (count large-numbers-text))
                              large-numbers-text))

(let [rand (rand-nth (keys large-number-map))]
  [rand (get large-number-map rand)])
~~~

So we now have all of the big number boundaries set up and ready to go, we need the other (100s and 1000s boundaries), with at least the same range.

~~~klipse
(def ten-boundaries
  (units 10 1000 (count large-numbers-text)))
(take 3 ten-boundaries)
~~~

~~~klipse
(def hundred-boundaries
  (units 100 1000 (count large-numbers-text)))
(take 3 hundred-boundaries)
~~~

## No class

No classes have been defined so far ... we have been able to do all of this interactively and at the REPL. This is a big win with Clojure - no need to wait for the compiler!

Let's top it off by capitalising the first letter. We can cheat here by using a Clojure standard library function. I'm not convinced it really is cheating but when it's this easy, it feels a little bit like it.

~~~klipse
(let [print-sequence (interpose " " (map name (starters 33)))]
  (clojure.string/capitalize (apply str print-sequence)))
~~~

## Guard rails

Even with the advent of spec, it's sometimes still simpler to use argument assertions to get the job done.

~~~klipse
(defn num-text [n]
  {:pre [(> n 0)]}
  n)
; ERR ---> Scroll window to the right to see the error
(num-text -1)
~~~

## Conclusions

- Stay with data - easier to work with and to test

- Layer on infra - to enable testing on the infra

- Use the latest possible transforms at the edge of the system

- Use randomness early and often

- Use properties and constraints to properly test your system

Developing with data, sequences and composing functions to manipulate them is how we will finally get to leave the sand pit.

## Gotchas

US readers please note: in the US the grammar for expressing these numbers is different and simpler: there's no 'and'. Providing this facility for all cultures and languages is not trivial and it reminds us that even such a small and simple example like this - like most any thing - is most often a subset of a much more complex domain.

## Disclaimer: Me != maths

Please let me know if you are a maths whizz and know a simpler way to solve this.

Also, if you have another functional way, in JS or Haskell or Scala or Java 8.

This has been fun but if there is a 'right way', I would be happy to learn it!


[klipse]: https://github.com/viebel/klipse

<script src="http://app.klipse.tech/plugin/js/klipse_plugin.js?"></script>



{% include disqus.html %}
