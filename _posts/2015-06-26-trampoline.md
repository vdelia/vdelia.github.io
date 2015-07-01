---
layout: post
title:  "Trampoline"
date:   2015-06-26 23:58:58
categories: [functional, trampoline]
tags:
  - functional programming
  - python
  - tail call elimination
---


From [wikipedia][wiki-trampoline]:

> As used in some Lisp implementations, a trampoline is a loop that iteratively invokes thunk-returning functions (continuation-passing style).
> A single trampoline suffices to express all control transfers of a program; a program so expressed is trampolined, or in trampolined style;
> converting a program to trampolined style is trampolining. Programmers can use trampolined functions to implement tail-recursive function
> calls in stack-oriented programming languages

In this post I explain what a trampoline is and why it matters.

Consider this recursive python function

{% highlight python linenos=table %}
def tail_recursive_f(n):
    if n == 0:
        return "done"
    print n
    return f(n-1)
{% endhighlight %}

The argument `n` gives us the number of recursive calls performed by `tail_recursive_f`. Can you invoke `tail_recursive_f` with any value for its argument `n`? In python, and in the *stack-oriented programming languages* wikipedia talks about, the answer is no. At each recursive step, the function parameters and the return address are pushed onto the stack, which is a limited resource. Thus, big values of `n` break the stack.


{% highlight python linenos=table %}
>>> tail_recursive_f(10000)
10000
...
RuntimeError: maximum recursion depth exceeded
{% endhighlight %}

Python has a safety-guard: if the recursion depth level exceeds a hardcoded value, it triggers a `RuntimeError` exception.

What if you want to stop caring about the stack? If your function is [tail-recursive][wiki-tailrec], like `tail_recursive_f`, you can write a **trampolined** version of it.

The objective is to 1) store the recursive calls somewhere for future execution,
2) extract them from the function, and 3) run them in a loop.

To postpone a computation, we usually wrap it in a function that performs that computation
when invoked. That function is often called *thunk*.

{% highlight python linenos=table %}
def operation():
    return 6*10

def lazy_operation():
    return lambda: 6*10
{% endhighlight %}

{% highlight python linenos=table %}
>>> operation()
60
>>> lazy_operation():
<function <lambda> at 0x7ffa2753c7d0>
>>> thunk=lazy_operation()
>>> thunk()
60
{% endhighlight %}

Here, we transform `tail_recursive_f` in the same way: we wrap the recursive
call in a lambda.

{% highlight python linenos=table %}
def tail_recursive_f(n):
    if n == 0:
        return "done"
    print n
    return lambda: f(n-1)
{% endhighlight %}

This version of `tail_recursive_f` runs one step, and then it returns a function
containing the continuation.

{% highlight python linenos=table %}
>>> tail_recursive_f(10000)
10000
<function <lambda> at 0x7ffa2753c758>
{% endhighlight %}

We can store it in a variable, and call it to execute the next step and
get the next continuation.

{% highlight python linenos=table %}
>>> thunk = tail_recursive_f(10000)
10000
>>> thunk()
9999
<function <lambda> at 0x7ffa2753c578>
{% endhighlight %}

and so on.

{% highlight python linenos=table %}
>>> thunk = thunk()
9999
>>> thunk = thunk()
9998
{% endhighlight %}

At the end, the last thunk will receive `n=0`, and it won't return a callable,
but the string `"done"`.

The **tampoline** is the loop running the chain of *thunks*, until they return a callable.

{% highlight python linenos=table %}
def trampoline(f, *args, **kwargs):
    g = lambda: f(*args, **kwargs)
    while callable(g):
        g = g()
    return g
{% endhighlight %}

The function takes in input the trampolined function, and its arguments, to be able to call it.
At the beginning, it wraps it in a new function, to make it callable without parameters.
Then, it keeps calling the thunks until they return a thunk, i.e. a callable.
At the end, it will have the actual result, and it returns it.

Now `tail_recursive_f(10000)` is invoked in this way

{% highlight python linenos=table %}
>>> trampoline(tail_recursive_f, 10000)
{% endhighlight %}

which prints all the integers between 10000 and 1, before returning `"done"`.

What about a not tail-recursive function? I take the simplest recursive function: the factorial.

{% highlight python linenos=table %}
def factorial(n):
    if n == 0:
        return 1
    return n*factorial(n-1)
{% endhighlight %}
{% highlight python linenos=table %}
>>> factorial(3)
6
>>> factorial(1000)
...
RuntimeError: maximum recursion depth exceeded
{% endhighlight %}

This version of factorial is not tail-recursive: at each recursion level,
the execution of the callee must return to the caller because it multiplies its
return value by `n`.

So first of all we turn it into tail recursive. To do this, one technique is to
propagate intermediate results through the stack using the function arguments.

{% highlight python linenos=table %}
def tail_recursive_factorial(n, acc=1):
    if n == 0:
        return acc
    return tail_recursive_factorial(n-1, acc=n*acc)
{% endhighlight %}

In this way, at each recursion step `n`, we have all the information to
compute the result: there is no need to return to the caller.

Then, we  apply the same transformation as before
{% highlight python linenos=table %}
def tail_recursive_factorial(n, acc=1):
    if n == 0:
        return acc
    return lambda: tail_recursive_factorial(n-1, acc=n*acc)
{% endhighlight %}

and we are ready for our `trampoline`

{% highlight python linenos=table %}
>>> trampoline(tail_recursive_factorial, 3)
6
>>> trampoline(tail_recursive_factorial, 1000)
402387260077093773543702433923003985719374864210714632543799910429938512398629020592044208486969404800479988610197196058631666872994808558901323829669944590997424504087073759918823627727188732519779505950995276120874975462497043601418278094646496291056393887437886487337119181045825783647849977012476632889835955735432513185323958463075557409114262417474349347553428646576611667797396668820291207379143853719588249808126867838374559731746136085379534524221586593201928090878297308431392844403281231558611036976801357304216168747609675871348312025478589320767169132448426236131412508780208000261683151027341827977704784635868170164365024153691398281264810213092761244896359928705114964975419909342221566832572080821333186116811553615836546984046708975602900950537616475847728421889679646244945160765353408198901385442487984959953319101723355556602139450399736280750137837615307127761926849034352625200015888535147331611702103968175921510907788019393178114194545257223865541461062892187960223838971476088506276862967146674697562911234082439208160153780889893964518263243671616762179168909779911903754031274622289988005195444414282012187361745992642956581746628302955570299024324153181617210465832036786906117260158783520751516284225540265170483304226143974286933061690897968482590125458327168226458066526769958652682272807075781391858178889652208164348344825993266043367660176999612831860788386150279465955131156552036093988180612138558600301435694527224206344631797460594682573103790084024432438465657245014402821885252470935190620929023136493273497565513958720559654228749774011413346962715422845862377387538230483865688976461927383814900140767310446640259899490222221765904339901886018566526485061799702356193897017860040811889729918311021171229845901641921068884387121855646124960798722908519296819372388642614839657382291123125024186649353143970137428531926649875337218940694281434118520158014123344828015051399694290153483077644569099073152433278288269864602789864321139083506217095002597389863554277196742822248757586765752344220207573630569498825087968928162753848863396909959826280956121450994871701244516461260379029309120889086942028510640182154399457156805941872748998094254742173582401063677404595741785160829230135358081840096996372524230560855903700624271243416909004153690105933983835777939410970027753472000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000L
{% endhighlight %}

[wiki-trampoline]: https://en.wikipedia.org/wiki/Trampoline_%28computing%29
[wiki-tailrec]: https://en.wikipedia.org/wiki/Tail_call
