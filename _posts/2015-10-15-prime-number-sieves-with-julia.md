---
layout: post
title:  "Euler 7 problem: prime number sieves with Julia"
date:   2015-10-15 11:30
categories: [julia]
permalink: /julia/euler7/
tags:
  - julia
  - project_euler
  - algorithms
  - notebook

---

[Julia][julia] is a high level programming language designed for technical computing.

Random interesting features (language + ecosystem):


*    designed to be fast
*    syntax similar to python/ruby
*    dynamic typing
*    parametric types
*    *multiple dispatch* and *syntactic macros* (Cf. [Common Lisp][cl])
*    designed for parallelism
*    built in package manager
*    it has a [backend for ipython/jupyter][ijulia]
*    great interoperability to C/Fortran/Python/...
*    great lib for [vector graphics][compose]
*    you can create/run your notebooks online on [JuliaBox][juliabox].
     It is able to sync your directories with Google Drive or github repos.
     When you sign on it, you get some interactive tutorials.


This post contains several implementations of the [problem 7 of project Euler][euler7].

# Hello, Julia!

There are three ways to start experimenting with Julia:

1.   [install][command_line] command line tools
2.   [Juno][juno], a complete IDE
3.   [JuliaBox][juliabox].

For this post I chose the third option. You can clone my [workspace][workspace]
in local, or let JuliaBox sync a local folder to it.

# Euler 7

https://projecteuler.net/problem=7

> By listing the first six prime numbers: 2, 3, 5, 7, 11, and 13, we can see that the 6th prime is 13.

> What is the 10 001st prime number?

The fastest solution is by using a sieve.


## Upper bound

Before running a sieve, we need to determine the size of the sieve. To do
this, we can apply  https://en.wikipedia.org/wiki/Prime-counting_function#Inequalities
to determine an upper bound for the nth prime.

**In [1]:**

{% highlight julia %}
function get_upper_bound(nth_prime)
    ub::Integer = ceil(nth_prime * log(nth_prime * log(nth_prime)))
    max(ub, 7)
end
{% endhighlight %}




    get_upper_bound (generic function with 1 method)



Julia has a dynamic type system, but we can add [type annotation][type_doc] to variables, like *ub*, to
generate more efficient code.

Moreover, `get_upper_bound` is a *generic function*. We can write several implementations of it, by adding different
type specialization of its arguments ([*multimethods*][methods]). Then, Julia chooses the concrete implementation to run by implementing *multiple dispatch*.

**In [2]:**

{% highlight julia %}
target_nth_prime = 10001
ub = get_upper_bound(target_nth_prime)
{% endhighlight %}




    114320



## The Sieve data structure

The sieve algorithms work on an array of size $$n$$, s.t. $$v[i]$$ is true if $$i$$ is prime.

Julia provides a very convenient bit array for this purpose.

**Julia arrays are indexed starting from 1, not 0.**

**In [3]:**

{% highlight julia %}
v = trues(ub)
v[0]
{% endhighlight %}


    BoundsError()
    while loading In[3], in expression starting on line 2



     in getindex at ./bitarray.jl:350


## Sieve of Eratosthenes

First of all I try the [Sieve of Eratosthenes][eratosthenes].

**In [4]:**

{% highlight julia %}
function eratosthenes_sieve(target_nth)
    n = get_upper_bound(target_nth)
    root_n::Integer = floor(sqrt(n))
    sieve = trues(n)
    sieve[1] = false

    for i = 2:root_n
        if sieve[i]
            # this is faster than v[i^2:i:n] = false
            for j=i^2:i:n
                sieve[j] = false
            end
            target_nth -= 1
            if 0 == target_nth
                return i
            end
        end
    end

    # I keep going past sqrt(n) until the target_nth
    for i = root_n+1:n
        if sieve[i]
            target_nth -= 1
            if 0 == target_nth
                return i
            end
        end
    end
end

@time eratosthenes_sieve(target_nth_prime)
{% endhighlight %}

    elapsed time: 0.014452114 seconds (430144 bytes allocated)





    104743



Here we measure the elapsed time by using the macro `time`.

Macros are not functions, and this is made explicit with the `@` prefix.

## Sieve of Sundaram

The [sieve of Sundaram][sundaram] is slightly more complex. Compared to Eratosthenes's, it skips even numbers.


**In [5]:**

{% highlight julia %}
function sundaram_sieve(target_nth)
    ub = get_upper_bound(target_nth)
    sieve = trues(ub)
    n = div(ub, 2)
    target_nth -= 1 # it skips 2
    for i = 1:n
        step = i * 2 + 1
        for j=i+i*step:step:n
            sieve[j] = false
        end
        if sieve[i]
            target_nth -= 1
            if 0 == target_nth
                return i*2+1
            end
        end
    end
end


@time sundaram_sieve(target_nth_prime)
{% endhighlight %}

    elapsed time: 0.010705828 seconds (293692 bytes allocated)





    104743



## Sieve of Atkin

The [sieve of Atkin][atkin] is a fast and modern algorithm for prime generation. It **can** be faster than the others,
depending on how smart is your implementation.

For the following code, I applied ideas from [this page][implementation].


**In [6]:**

{% highlight julia %}
function get_bitmask(n, idx)
    f = falses(n)
    for i in idx
        f[i] = true
    end
    f
end

function atkin_sieve(target_nth)

    if target_nth < 4
        return eratosthenes_sieve(target_nth)
    end

    ub = get_upper_bound(target_nth)
    root_ub::Integer = floor(sqrt(ub))
    target_nth -= 3
    is_prime = get_bitmask(ub, [2, 3, 5])

    step31_filter = get_bitmask(60, [1, 13, 17, 29, 37, 41, 49, 53])
    step32_filter = get_bitmask(60, [7, 19, 31, 43])
    step33_filter = get_bitmask(60, [11, 23, 47, 59])

    x, x_square, x_step = 1, 1, 3

    while x <= root_ub

        y, y_square, y_step = 1, 1, 3
        while y <= root_ub
            # 4x^2 + y^2
            n = (x_square << 2) + y_square
            n_mod_60 = n % 60
            if n <= ub && n_mod_60 > 0 && step31_filter[n_mod_60]
                is_prime[n] = !is_prime[n]
            end

            # 3x^2 + y^2
            n -= x_square
            n_mod_60 = n % 60
            if n <= ub && n_mod_60 > 0 && step32_filter[n_mod_60]
                is_prime[n] = !is_prime[n]
            end

            if x > y
                n -= (y_square << 1)
                n_mod_60 = n % 60
                if n <= ub && n_mod_60 > 0 && step33_filter[n_mod_60]
                    is_prime[n] = !is_prime[n]
                end
            end

            y += 1
            y_square += y_step
            y_step += 2
        end
        x += 1
        x_square += x_step
        x_step += 2
    end

    for n = 7:root_ub
        if is_prime[n]
            target_nth -= 1
            if 0 == target_nth
                return n
            end

            n_squared = n^2
            for c = n_squared:n_squared:ub
                is_prime[c] = false
            end
        end
    end

    for n = root_ub+1:ub
        if is_prime[n]
            target_nth -= 1
            if 0 == target_nth
                return n
            end
        end
    end
end

@time atkin_sieve(target_nth_prime)
{% endhighlight %}

    elapsed time: 0.031512008 seconds (1497144 bytes allocated)





    104743



For the Euler 7 assignment, this implementation is the slowest of the three,
but we want to benchmark for bigger primes!


# Benchmark
The function `bench_sieve` takes an ordinal identifying the prime to generate,
the sieve to use, and the number of runs.

It runs the sieve a first time, just to be sure it is compiled by the Julia interpreter,
and then it measure the elapsed time as a mean over the specified number of runs.

**In [7]:**

{% highlight julia %}
function bench_sieve(nth_prime, generator, runs=5)
    times = zeros(runs)
    # run the first time just to ensure it is compiled
    generator(nth_prime)
    for i = 1:runs
        times[i] = @elapsed generator(nth_prime)
    end
    mean(times)
end

{% endhighlight %}




    bench_sieve (generic function with 2 methods)



Julia modules are imported with `using`.
Here, I create a dictionary whose keys are the name of the algorithms (I used symbols, i.e. [interned strings][interning]),
and whose values are the functions implementing the corresponding algorithm.

Then I run the benchmark, arranging the results in a DataFrame.

**In [8]:**

{% highlight julia %}
using DataFrames

sieves = [:Erathostenes => eratosthenes_sieve,
          :Sundaram => sundaram_sieve,
          :Atkin => atkin_sieve]

df = DataFrame(nths = map(round, logspace(1, 7, 7)))
for (name, sieve) in sieves
    df[name] = map(nth -> bench_sieve(nth, sieve), df[:nths])
end

{% endhighlight %}


**In [9]:**

{% highlight julia %}
# to install Gadfly
# Pkg.add("Gadfly")
using Gadfly

set_default_plot_size(24cm, 16cm)
p = plot(melt(df, :nths), x = :nths,
         y = :value, color=:variable,
         Guide.xlabel("nth prime"),
         Guide.ylabel("elapsed time [s]"),
         Geom.point,
         Geom.line)
display(p)
{% endhighlight %}

![julia bench]({{ site.url }}/assets/sieves/julia_bench.svg)

For larger primes, Atkin is faster than Sundaram.

[julia]: http://julialang.org/
[cl]: https://en.wikipedia.org/wiki/Common_Lisp
[generic_programming]: https://en.wikipedia.org/wiki/Generic_programming
[ijulia]: https://github.com/JuliaLang/IJulia.jl
[euler7]: https://projecteuler.net/problem=7
[command_line]: http://julialang.org/downloads/
[compose]: http://composejl.org/
[juliabox]: https://www.juliabox.org/
[juno]: http://junolab.org/
[workspace]: https://github.com/vdelia/juliabox_workspace
[type_doc]: http://julia.readthedocs.org/en/latest/manual/types/
[methods]: http://julia.readthedocs.org/en/latest/manual/methods/#man-methods
[eratosthenes]: https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes
[sundaram]: https://en.wikipedia.org/wiki/Sieve_of_Sundaram
[atkin]: https://en.wikipedia.org/wiki/Sieve_of_Atkin
[implementation]: http://www.codeproject.com/Articles/490085/Eratosthenes-Sundaram-Atkins-Sieve-Implementation
[interning]: https://en.wikipedia.org/wiki/String_interning
