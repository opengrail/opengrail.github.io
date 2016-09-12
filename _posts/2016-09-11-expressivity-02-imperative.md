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

```C
char *strcpy(char *dest, const char *src)
```

*Parameters:* 

`dest` - This is the pointer to the destination array where the content is to be copied. `src` - This is the string to be copied.

*Return Value:* 

This returns a pointer to the destination string dest.

# Expressive, not
This small example shows how confused the C library authors were about what was the correct thing to do even for the trivial task of copying strings.

There are some good things here though. Firstly that the `src` string is immutable in the function so `strcpy` cannot change it.

Secondly it returns a pointer with the copied value.

Well that would be nice, but it has overwritten `dest`, so there is never any need to use that nice return value. It would be more correct to write:

```C
void strcpy(char *dest, const char *src) /* Wrong but honest */
```

And for my taste, if we have to do it this way, I would prefer `src` come first and `dest` second but that's perhaps a modern affectation.

```C
void strcpy(const char *src, char *dest) /* Wrong but more obvious order for parameters */
```

A more expressive signature would be:

```C
char *strcpy(const char *src) /* Not good C, but definitely more ergonomic */
```


# Playing it out in client code
Here is a simple example:

```C
include <string.h>

int main()
{
   char src[40];
   char dest[100];
  
   memset(dest, '\0', sizeof(dest));
   strcpy(src, "This is an example");
   strcpy(dest, src);
}

```

You see here that the client programmer has to abide by the unwritten rules of strcpy. Amongst these are:
- ensure that the memory for `dest` array has been allocated
- ensure that the size of the `dest` array is >= the source array

# Improved C

```C
include <string.h>

int main()
{
   char *dest = strcpy("This is an example"); /* Not valid C - why? */
}

```

This does not work in C for one simple reason: C does not have a garbage collector. If the strcpy function were to allocate the memory for the `dest` array there would be nothing to clean it up so we would have a memory leak. Thus we are hobbled in our range of expressivity by the need to allocate and free memory by hand.

# Looping in C

Another area where imperative programming makes us work hard for our reward is iteration. Here is a simple for loop in C

```C
#include <stdio.h>
 
int main()
{
    const int max = 5;
    int i;
    for(i = 0; i < max; i++)
        printf("%d ", i);     // 0 1 2 3 4
}
```

Again, one nice thing here: the `max` value is constant or immutable so there's no way that it's going to change during the loop.

That `i` however is going to be changing through the use of `i++`. This is equivalent to `i = i + 1`





[imperative]: {% post_url 2016-09-11-expressivity-02-imperative %}
[oo]: {% post_url 2016-09-11-expressivity-03-oo %}
[fp]: {% post_url 2016-09-11-expressivity-04-fp %}

[expressivity]: {% post_url 2016-09-11-expressivity-01 %}

{% include disqus.html %}