---
layout: post
title:  "Trampoline"
date:   2015-06-26 23:58:58
categories: [functional, trampoline]
permalink: /functional/trampoline/
tags:
  - functional
  - python
  - basics
---

From [wikipedia][wiki-trampoline]:

> As used in some Lisp implementations, a trampoline is a loop that iteratively invokes thunk-returning functions (continuation-passing style).
> A single trampoline suffices to express all control transfers of a program; a program so expressed is trampolined, or in trampolined style;
> converting a program to trampolined style is trampolining. Programmers can use trampolined functions to implement tail-recursive function
> calls in stack-oriented programming languages

In this post I explain what a trampoline is and why it matters.

Consider this recursive python function

```python
def f(n):
    if n == 0:
        return
    print n
    return f(n-1)
```

This function performs `n` recursive calls when invoked with parameter `n`.
Can you use any value for `n`? In python, and in the *stack-oriented programming languages*
wikipedia talks about, the answer is no. At each recursive step, the function parameters
and the return address are pushed onto the stack, which is a limited resource.
Thus, big values of `n` break the stack.

In python there is a safety-guard for this: the maximum number of recursive steps your function
can perform is hardcoded. When this limit is exceeded, you get a polite `RuntimeError` exception.

```python
>>> f(10000)
10000
...
RuntimeError: maximum recursion depth exceeded
```

But what if you need a deeper recursion? If your function is [tail-recursive][wiki-tailrec],
you can easily write a *trampolined* version of it: you wrap the recursive call in a function.

*Wrapping a computation in a function is a very common pattern to postpone evaluation.*

```python
def f(n):
    if n == 0:
        return
    print n
    return lambda: f(n-1)
```


Now it executes the first step, but then returns immediately a function (a *thunk*).

```python
>>> f(10000)
10000
<function <lambda> at 0x7ffa2753c758>
```


How do you execute `f(10000)` now? You need a *trampoline*.

```
def trampoline(f, *args, **kwargs):
    g = lambda: f(*args, **kwargs)
    while callable(g):
        g = g()
    return g
```

The function takes as arguments the trampolined function, and it arguments.
At the beginning, it wrap it in another function, so that it can invoke it without
using parameters. Then, until its result is a callable, it invokes it.
At the end, it will have the actual result, and it returns it.

Now `f(10000)` is invoked in this way

{% highlight python %}
trampoline(f, 10000)
{% endhighlight %}

which prints all the integers between 10000 and 1.

TODO: example with factorial

[wiki-trampoline]: https://en.wikipedia.org/wiki/Trampoline_%28computing%29
[wiki-tailrec]: https://en.wikipedia.org/wiki/Tail_call
