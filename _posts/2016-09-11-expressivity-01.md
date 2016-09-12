---
layout: post
title:  "Expressivity ascending"
date:   2016-09-11 19:59:38 +0100
comments: false
categories: Clojure ClojureScript Imperative
---

# Expressive
What does it mean to be more expressive? How can I convince you that one or other form of programming is more expressive than others? 

Going back to first principles, consider expressivity in general. in English we can write:

```
One plus one equals two.
```

That works but it's more expressive (to programmers at least!) in a mathematical form:

```
1 + 1 = 2
```

The mathematical form is more concise than the long hand so there is a win in productivity but that's not what makes it more expressive for this context.

The increase in expressivity comes from the **meaning** behind the mathematical syntax. English grammar is open whereas the grammar of mathematical expressions is closed or at least constrained to some well defined semantics.

Parsing the mathematical expression is easier than the English version if you are familiar with the syntax and semantics but otherwise it's just a pile of symbolic gibberish. The need to understand this extra level of symbology has a price - so it has to be worth paying. And not everybody, as we recall from school, wanted to pay it. Whenever we increase expressivity - be it in maths or poetry or music we lose some people along the way.

# Algebra
And so, talking of losing people, let's take the maths up a level (to about as far as I can safely go to be honest) and compare arithmetic against algebra

```
1 + 1 = 2  ; arithmetic

a + a = b  ; algebra
```

The algebraic form and the arithmetic forms are equally concise so we don't gain anything in the effort required to write the expression. That feels like a productivity bummer.

In fact, we lose something of the directness of the arithmetic form when we compare it to the algebraic form. The algebraic form requires yet another set of semantics - a further abstraction - that sits over the semantics of arithmetic. So we lose some more people at this stage.

# Plug and play
We have asserted a truth: in this expression a + a will always equal b.

Algebra buys us the ability to assert expressions and then play with them. By playing, I mean we have introduced the power of inference and deductive logic. For example we can deduce that a and b cannot be equal unless both are 0. Further we can assert that b is always greater than a, again unless they are 0. [ Did you notice how 0 is quite annoying - maybe it's the mathematical equivalent of null ... but that's for another day)

Many more things can be deduced from this increase in abstraction. And any abstraction that gives us new ways to think is a genuine improvement in expressivity.
 
Finally, and more prosaically, we can plug in some numbers and play out the arithmetic. 

What a change! Arithmetic, once upon a time so magical, is demoted to the role of servant to the algebra.

# The Challenge
I want to finish with a woolly model to test expressivity - does it enable us to think in a materially different way and yet remain precise by being open to a transparent and predictable form of substitution?

This challenge is the basis on which we should judge functional programming.

There is a considerable body of software in the world, decent amounts of which appear to work. So do I say that we have been failed by other categories of programming languages? No, I don't. Like arithmetic, other forms of programming have value. But like algebra, we should strive to use the most powerful form of expression we can harness. 

Some will be able to fly to degrees of abstraction that I cannot follow and they will lose me along the way. But I can do the basics and am willing to learn more. If you read this far, I think you are too.

The associated posts will form a short, biased survey of facts and opinions that have been carefully curated chosen to support my arguments.

- [Imperative code][imperative] considered, harmful?
- [Object-oriented][oo] composting
- [Functional programming][fp] as a New Hope

[imperative]: {% post_url 2016-09-11-expressivity-02-imperative %}
[oo]: {% post_url 2016-09-11-expressivity-03-oo %}
[fp]: {% post_url 2016-09-11-expressivity-04-fp %}


{% include disqus.html %}
