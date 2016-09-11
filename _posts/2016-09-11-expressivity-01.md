---
layout: post
title:  "Expressivity ascending"
date:   2016-09-11 19:59:38 +0100
comments: false
categories: Clojure ClojureScript Imperative
---

# Expressivity in code
What does it mean to be more expressive? How can I convince you that one or other form of programming is more expressive than others? 

Going back to first principles, consider expressivity in general. in English we can write:

```
One plus one equals two.
```

That works but it's more expressive in a mathematical form:

```
1 + 1 = 2
```

The mathematical form is more concise than the long hand so there is a win in productivity but that's not what makes it more expressive. [ And by the way I'm not saying that English is not expressive. I'm not totally crazy. ]

The increase in expressivity comes from the **meaning** behind the mathematical syntax. English grammar is open whereas the grammar of mathematical expressions is closed or at least constrained to some well defined semantics.

Parsing the mathematical expression is easier than the English version if you are familiar with those semantics but otherwise it's just a pile of symbolic gibberish. And so, talking of gibberish let's take the maths up a level (to about as far as I can safely go to be honest) and compare arithmetic against algebra

```
1 + 1 = 2  ; arithmetic

a + a = b  ; algebra
```


The algebraic form and the arithmetic forms are equally concise so we don't gain anything in the effort required to write the expression. So that feels like a productivity bummer.

In fact, we lose something of the directness of the arithmetic form when we compare it to the algebraic form. The algebraic form requires yet another set of semantics - a further abstraction - that sits over the semantics of arithmetic. 

# Plug and play
We can use substitution rules to plug in any numbers and the arithmetic works.

But that's not the where the value lies - instead it's the fact that we have not just made some arithmetic we have asserted a truth. That in this expression, for that is what we call it, hahaha, a + a will always equal b.

And that's in the end what algebra buys us the ability to assert expressions that can in some way be proven by plugging in some numbers and playing out the arithmetic. So the arithmetic becomes a lowly servant of the algebra.

I want to finish with a woolly model to test expressivity - does it enable us to say more in our programs and yet remain precise by being open to a transparent and predictable form of substitution?

# The Challenge
That challenge is the basis of functional programming.

There is a body of working software in the world, so why do I say that we have been failed by other categories of programming languages? Luckily, I'm not alone in this opinion. The next posts will form a short, biased survey of facts and opinions that have been carefully curated chosen to support my arguments.

- [Imperative code][imperative] considered, harmful?
- [Object-oriented][oo] composting
- [Functional programming][fp] as a New Hope

[imperative]: {% post_url 2016-09-11-expressivity-02-imperative %}
[oo]: {% post_url 2016-09-11-expressivity-03-oo %}
[fp]: {% post_url 2016-09-11-expressivity-04-fp %}


{% include disqus.html %}
