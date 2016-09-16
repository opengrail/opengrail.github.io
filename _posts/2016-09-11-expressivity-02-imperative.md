---
layout: post
title:  "Expressivity: Imperative considered, harmful?"
date:   2016-09-11 19:59:38 +0100
categories: Clojure ClojureScript Functional
---

# Expressivity in code
In the [initial post][expressivity] I gave an example of how we see expressivity increase in maths as we progress from words to symbolic arithmetic to algebra.

I introduced a woolly model to test expressivity - does it enable us to say more in our programs and yet remain precise by being open to a transparent and predictable form of substitution?

I am going to look back at some of the imperative style code that I used to write back in the 90s and run it through the test. It's a style that's very much alive today so this is not a pure nostalgia trip.

# Imperative
Taking C as a canonical example, the standard C function for copying strings is `strcpy`

```c
char *strcpy(char *dest, const char *src)
```

* `dest` - pointer to the destination array where the content is to be copied. 
* `src` - The string to be copied.
* `returns` - a pointer to the destination string dest.

# Expressive, not
This small example shows how confused the C library authors were about what was the correct thing to do even for the trivial task of copying strings.

There are some good things here though: firstly that the `src` string is immutable so `strcpy` cannot change it. Secondly it returns a pointer with the copied value.

Well that *would have been nice*, but the copied value is `dest`, so there is never any need to use that return value. It would have been more honest to write:

```c
void strcpy(char *dest, const char *src) /* Honest */
```

And for my taste, if we have to do it this way, I would prefer `src` come first and `dest` second so that it reads more normally (at least for an English reader)

```c
void strcpy(const char *src, char *dest) /* More 'obvious'? */
```

# Playing it out in client code
Here is a simple example using the *standard library method*:

```c
include <string.h>

int main() {
   char src[40];
   char dest[100];
   memset(dest, '\0', sizeof(dest));

   strcpy(src, "This is an example");
   strcpy(dest, src);
}
```

The programmer has to abide by the unwritten rules of `strcpy`, or maybe it's better said, the conventions of C. Amongst these are:

* ensure that the memory for `dest` array has been allocated
* ensure that the size of the `dest` array is >= the source array, otherwise you will get less than you gave!

Imperative code like this is awkward because it combines in detail what needs to be done and how exactly to do it.

# Improved C

A tweak here and there and we can have simpler and more expressive function:

```c
char *strcpy(const char *src) /* Not good C, but definitely more ergonomic */
```

```c
include <string.h>

int main()
{
   char *dest = strcpy("This is an example"); /* Not valid C - why? */
}
```

It's simpler because there are fewer names to grok and there is no confusion between the input and the output. These small gains in expressivity add up.

*Or they would* but this does not work in C for one simple reason: **C does not have a garbage collector**.

If `strcpy` allocated memory for the return value, there would be no way for `strcpy` to clean it up and the client code couldn't either. So we would have a memory leak / hydrant. Not having a GC (or any form of automatic memory management) hobbles our range of expressivity by forcing the developer to allocate and free memory by hand. As they say these days, like an animal.

Modern GC implementations, like those on the Java Virtual Machine, are often as fast if not faster than hand tuned solutions and *much less prone to error*.

# Functions do not a functional language make

Working with a function like `strcpy` is a good example why most of the C standard library is not functional in its design. In a functional language the goal is a pure function - an input takes parameters and returns a result without any side effects. The one side effect, that memory is allocated in both cases, is managed by the language in a functional environment.

# Looping in C

Another area where imperative programming makes us work hard for a small reward is iteration. Here is a simple for loop in C

```c
#include <stdio.h>
 
int main()
{
    const char nums[] = "01234"
    const int max = strlen(nums); /* strlen is a pure function */
    int i;
    for(i = 0; i < max; i++)
        printf("%d ", nums[i]);     // 0 1 2 3 4
}
```

One small simplifying thing here: the `max` and `nums` values are constant or immutable so there's no way they will change during the loop. Optimising C compilers, and of course those in higher level languages too, can simply substitute the values directly where they are used.

That `i` however is going to be changing through the use of `i++`. FYI this is the C equivalent to `i = i + 1`. 

# High cost of small defects

If you look carefully at the C code you will notice a small defect. I'm not gonna call it a bug but it's not 100% correct either. The last iteration of the `for` loop outputs an extraneous space. It's not a biggie here but it really does require an additional piece of logic to obtain a more *correct* solution.

```c
#include <stdio.h>

int main()
{
    const char nums[] = "01234"
    const int max = strlen(nums); /* strlen is good :) too */
    int i;
    for(i = 0; i < max; i++)
        if (i == (max - 1))
            printf("%d\n", nums[i]);
        else
            printf("%d ", nums[i]);
}
```

I reckon more than one developer, like me did not know immediately which iteration of `i` is the correct `i`. The moment where the `i++` operation take place is not obvious in the syntax. In fact, it is after the loop has executed but before the next test. So the `i++` is effectively moved to be the final expression of the loop no matter which branch is taken. Or if you prefer, it is moved to be the expression before the test, except for the first time around the loop. And would it make a difference, if you wrote `++i`?
 
To be honest, I have not ran this version of the code so please send a comment to correct the it if you know better and help me to make my point!

# Hard work

This is hard work: we need two variables and a loop to print out the values of an array, with the numbers joined by spaces. In a more expressive language we would be able to state our intention and allow the machinery of the language implementation to relieve us from this book-keeping. 

Like this perhaps:

```clojure
(println (string/join " " "1234"))
1 2 3 4
```

Trust me, there is no trailing space :)

I'm going to go into more depth about getting rid of loops in the Functional Programming post but I think even the most die in the wool C coder would agree that this one liner is more expressive.

# Is it harmful?

It makes us work harder and we can cut ourselves - or more importantly our users - more easily. For most common programming problems, yes I would consider C harmful.

The Swift language designers recently decided to remove support for the `++` and `--` operators from of the language (v3 and onwards). [They argue][swift-ref] that they are the source of too many errors. :thumbsup:

Manual memory allocation is also a common form of security exploit, for example when it is not [allocated or deallocated properly.][dealloc].

On the other hand C has its place in device drivers and other low level system code where OO and functional programming have traditionally struggled to add value. 

That crown may be at risk however as we continue to see the rise of non-C system level programming languages such as Rust, Go and even JavaScript (see the 64Kb [Kinoma][kinoma])

# Next OO

The [next post will show how OO][oo] - in Java at least - imposes limits on expression, **by design**.

[imperative]: {% post_url 2016-09-11-expressivity-02-imperative %}
[oo]: {% post_url 2016-09-11-expressivity-03-oo %}
[fp]: {% post_url 2016-09-11-expressivity-04-fp %}

[expressivity]: {% post_url 2016-09-11-expressivity-01 %}
[kinoma]: http://kinoma.com
[swift-ref]: https://github.com/apple/swift-evolution/blob/master/proposals/0004-remove-pre-post-inc-decrement.md
[dealloc]: https://www.securecoding.cert.org/confluence/pages/viewpage.action?pageId=437

{% include disqus.html %}