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

That works but it's more expressive (to programmers at least!) in a mathematical form:

```
1 + 1 = 2
```

The mathematical form is more concise than the long hand so there is a win in productivity but that's not what makes it more expressive for this context.

The increase in expressivity comes from the **meaning** behind the mathematical syntax. English grammar is open whereas the grammar of mathematical expressions is closed or at least constrained to some well defined semantics.

Parsing the mathematical expression is easier than the English version if you are familiar with the syntax and semantics but otherwise it's just a pile of symbolic gibberish. The need to understand this extra level of symbology has a price - so it has to be worth paying. And not everybody, as we recall from school, wanted to pay it. Whenever we increase expressivity - be it in maths or poetry or music we lose some people along the way.

And so, talking of losing people, let's take the maths up a level (to about as far as I can safely go to be honest) and compare arithmetic against algebra

```
1 + 1 = 2  ; arithmetic

a + a = b  ; algebra
```

The algebraic form and the arithmetic forms are equally concise so we don't gain anything in the effort required to write the expression. That feels like a productivity bummer.

In fact, we lose something of the directness of the arithmetic form when we compare it to the algebraic form. The algebraic form requires yet another set of semantics - a further abstraction - that sits over the semantics of arithmetic. So we lose some more people at this stage.

# Plug and play
We can use substitution rules to plug in any numbers and the arithmetic works.

But that's not the where the value lies - instead it's the fact that we have not just made some arithmetic. 

We have asserted a truth: in this expression a + a will always equal b.

And that's in the end what algebra buys us the ability to assert expressions and then play with them. By playing, I mean we can introduce logic. For example we can deduce that a and b cannot be equal unless they are 0. We also know that b is always greater than a. 

Many more things are implied and can be deduced from this increase in abstraction. And abstraction that gives us new ways to think is a genuine improvement in expressivity.
 
Finally, and more prosaically, we can plug in some numbers and play out the arithmetic. Arithmetic becomes the dull, lowly servant of the algebra.

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
