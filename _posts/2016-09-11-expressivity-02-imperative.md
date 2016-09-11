---
layout: post
title:  "Expressivity: Imperative considered, harmful?"
date:   2016-09-11 19:59:38 +0100
categories: Clojure ClojureScript Functional
---

# Expressivity in code
In the [initial post][expressivity] I gave an example of how we see expressivity increase in maths as we progress from words to symbolic arithmetic to algebra 

I introduced a woolly model to test expressivity - does it enable us to say more in our programs and yet remain precise by being open to a transparent and predictable form of substitution?

I am going to look back at some of the imperative style code that I used to write back in the 90s and run it through the test. It's a style that's very much alive today so this is not a pure nostalgia trip.

#Imperative
Imperative style, taking C as a canonical example. It's very common to see a function like this:

The standard C function for copying strings is

```
char *strcpy(char *dest, const char *src)
```

**Parameters**
dest -- This is the pointer to the destination array where the content is to be copied.

src -- This is the string to be copied.

**Return Value**
This returns a pointer to the destination string dest.

**Example**

Here is a simple example:

```

include <stdio.h>
include <string.h>

int main()
{
   char src[40];
   char dest[100];
  
   memset(dest, '\0', sizeof(dest));
   strcpy(src, "This is an example");
   strcpy(dest, src);

   printf("Final copied string : %s\n", dest);
   
   return(0);
}

```

This is a mix of the two styles. On the one hand it does the right thing and returns a value. On the other hand it returns a pointer to dest. So the return value is almost always ignored.


[imperative]: {% post_url 2016-09-11-expressivity-02-imperative %}
[oo]: {% post_url 2016-09-11-expressivity-03-oo %}
[fp]: {% post_url 2016-09-11-expressivity-04-fp %}

[expressivity]: {% post_url 2016-09-11-expressivity-01 %}

{% include disqus.html %}