---
layout: post
title:  "About comprehensions, BigInts, Interfaces, Iterators and Tasks in Julia"
date:   2015-12-05 11:30
categories: [julia]
permalink: /julia/primes/
tags:
  - julia
  - parallel
  - algorithms
  - notebook

---

This post is the continuation of [the intro to Julia]({% post_url 2015-10-15-prime-number-sieves-with-julia %}).


# The notebook

The ipython notebook used to generate this post is available on [github][workspace].
You can clone it in local, or sync a local folder to it on [JuliaBox][juliabox].

#  BigInts and array comprehensions: Fermat numbers

A [Fermat number][fermat] is a positive integer of the form


$$F_n = 2 ^ {2^n} + 1$$

The only known Fermat number that are also primes are $$F_0$$, $$F_1$$, $$F_2$$, $$F_3$$, and $$F_4$$.
In the following sections, I test two algorithms for prime decomposition on sequences of Fermat numbers.

Julia does not promote automatically from Integer to BigInt.
Thus, I prefer to explicitly handle overflows:
if needed, I promote the argument $$n$$ to `BigInt`, to propagate the BigInt type everywhere.

Note that in the following cell I build the array using an **array comprehension**.

**In [1]:**

{% highlight julia %}
function fermat(n::Integer)
    if n >= 6
        n = BigInt(n)
    end
    2 ^ (2^n) + 1
end

fermat(from, to) = [fermat(i) for i in from:to]

fermat(0, 6)
{% endhighlight %}




    7-element Array{Union{BigInt,Int64},1}:
                        3
                        5
                       17
                      257
                    65537
               4294967297
     18446744073709551617



# Iteration interface and Tasks: Trial Division

The most naive algorithm for integers factorization is [Trial Division][trial]: to find
the prime factors of $$N$$, we try to divide it by all prime numbers smaller than  $$\sqrt{N}$$.

To generate the prime numbers, I use the Sundaram sieve (Cf. [the first post]{% post_url 2015-10-15-prime-number-sieves-with-julia %}), but I exploit the iteration
interface of Julia to have a lazy implementation of it.


## Iteration Interface: a lazy implementation of Sundaram Sieve

Julia can be extended with new user defined types.
which can be integrated seamlessly into the language by implementing the
right [interfaces][interfaces].

Consider for example the _for_ loop

{% highlight python %}
for i in obj
    do something with i
end
{% endhighlight %}

It can work with any object implementing the iteration interface, i.e. the methods
`start`, `next` and `done`.
That loop is syntactic sugar equivalent to

{% highlight python %}
state = start(obj)
while !done(obj, state)
    i, state = next(obj, state)
end
{% endhighlight %}

To define a lazy version of the Sundaram sieve, I create a new [composite type][composite-types],
containing the `is_prime` array of booleans (a bit array in this implementation), and the upper bound $$ub$$.

The type is parametric, since $$ub$$ can be `Integer` or `BigInt`.

**In [2]:**

{% highlight julia %}
immutable SundaramSieve{T}
    is_prime::BitArray
    ub::T
end
{% endhighlight %}

The iteration interface let me track the status of the iteration via the `state` object, which can be anything, and which is passed around by the three methods.

I use as state the last index checked by the sieve.

I start at 1, so `start` returns 1.

If $$p$$ is the current state, then `done` is true if there are no more primes between $$p$$ and $$ub$$. But to know that, the sieve must have run until $$ub$$. Thus, we maintain the sieving process one step beyond the prime returned at current iteration:
it computes until 7 but it returns 5; the next step it computes until 11 but it returns 7, and so on.

`next` receives the state, which allows to compute the prime it has to return, but it runs one step of the sieve until the next prime, or $$ub$$.

**In [3]:**

{% highlight julia %}
sundaram_sieve(n::Integer) = SundaramSieve(trues(n), div(n, 2)+1)

# the first state. In the state keep the last checked integer
Base.start(s::SundaramSieve) = 1

function Base.next(s::SundaramSieve, state)

    for i = state:s.ub
        step = i * 2 + 1
        for j=i+i*step:step:s.ub
            s.is_prime[j] = false
        end
        if s.is_prime[i]
            return (max(2, (state-1)*2+1), i+1)
        end
    end
    (max(2, (state-1)*2+1), s.ub)
end

Base.done(s::SundaramSieve, state) = state >= s.ub
{% endhighlight %}




    done (generic function with 51 methods)



**In [4]:**

{% highlight julia %}
for i in sundaram_sieve(10)
    print("$i ")
end

collect(sundaram_sieve(10))
{% endhighlight %}

    2 3




    4-element Array{Any,1}:
     2
     3
     5
     7



    5 7

Another way of implementing lazy evaluated sequences are [coroutines][tasks].


## Intro to Coroutines

Coroutines are functions whose evaluation can be suspended and resumed.
To create a coroutine, pass your function to the `Task` function.

To start, and resume, the execution of a coroutine use the method `consume`.
The function wrapped inside, produces data, and returns the control to the caller of `consume`,
using the `produce` function.

Task implements also the iteration interface, and iterating on it is equivalent to calling
`consume`.

Julia's runtime provides a scheduler to coordinate the execution of multiple coroutines.
Julia's coroutines always run in a single process, and they are scheduled with a cooperative
multitasking strategy: each coroutine has the responsibility
to release the control to other coroutines, by executing blocking operations (like reading from a Channel)
or by calling `yield` or `produce`. For the moment we will use them just as iterators,
but in the following sections we will see how to actually exploit them to coordinate
parallel tasks execution.

**In [5]:**

{% highlight julia %}
function trial_division(n)
    function _f()
        if n < 2
            return
        end
        ub::Integer = ceil(sqrt(n)) + 1

        for prime in sundaram_sieve(ub)
            while n % prime == 0
                produce(prime)
                n = div(n, prime)
            end
        end
        if n > 1
            produce(n)
        end
    end
    Task(_f)
end
{% endhighlight %}




    trial_division (generic function with 1 method)



**In [6]:**

{% highlight julia %}
f5 = fermat(5)
print("$f5 = 1")
for factor in trial_division(f5)
    print(" * $factor")
end
{% endhighlight %}

    4294967297 = 1 * 641 * 6700417

Unfortunately we cannot run this algorithm for bigger $$F$$s, since we are not able to store the sieve.

**In [7]:**

{% highlight julia %}
collect(trial_division(fermat(6)))
{% endhighlight %}


    LoadError: MethodError: `convert` has no method matching convert(::Type{BitArray{N}}, ::BigInt)
    This may have arisen from a call to the constructor BitArray{N}(...),
    since type constructors fall back to convert methods.
    Closest candidates are:
      call{T}(::Type{T}, ::Any)
      convert{T,N}(::Type{BitArray{N}}, !Matched::AbstractArray{T,N})
      convert{T}(::Type{T}, !Matched::T)
      ...
    while loading In[7], in expression starting on line 1



     in schedule_and_wait at task.jl:343

     in consume at task.jl:259

     in collect at array.jl:260

     in collect at array.jl:272


# Julia parallel computing: Pollard's rho algorithm

The function `pollard_rho` is almost a copy&paste from the wikipedia [pseudo-code][rho].

`pollard_rho` finds a divisor for the argument $$n$$. If it returns $$x$$ and $$y$$, s.t. $$n = x * y$$.

**In [8]:**

{% highlight julia %}
g(x, n) = (x^2 + 1) % n

function pollard_rho(n)
    x, y, d = 2, 2, 1

    if n % 2 == 0
        d = 2
    elseif n % 3 == 0
        d = 3
    else
        while d == 1
            x = g(x, n)
            y = g(g(y, n), n)
            d = gcd(abs(x-y), n)
        end
    end
    d, div(n, d)
end
{% endhighlight %}




    pollard_rho (generic function with 1 method)



It is already able to do more things than our `trial_division`!

**In [9]:**

{% highlight julia %}
f6 = fermat(6)
x, y = pollard_rho(f6)
"$$f6 = $$x * $$y"
{% endhighlight %}




    "18446744073709551617 = 274177 * 67280421310721"



What is not able to do is returning all the factors of $$n$$. To do that, we must recursively apply
`pollard_rho` to the factors of $$n$$, if they are different from $$n$$ and 1.

Those operations can be easily parallelized. In Julia, parallelism starts with [cluster managers][parallel].

## Cluster managers

`ClusterManager`s allow to create, manage, destroy sets of related processes.

Julia provides two of them:

1.    `LocalManager`, used to manage processes on a single machine
2.    `SSHManager`, to spawn processes on remote machines


Only the initial process, called `master`, has the right to spawn new processes.

To ask to the `LocalManager` to run 3 processes, we use

**In [11]:**

{% highlight julia %}
missing_procs = 3 - nprocs() # nnprocs() returns the number of running processes
if missing_procs > 0
    addprocs(missing_procs)
end
{% endhighlight %}

Now we can ask to the ClusterManager to run functions on those processes.
There is a reach API to do that, but the basic flow is

1.    you run a remote call, getting immediately a `RemoteRef`
2.    later on, you can `fetch` from a `RemoteRef` to get the actual result

A `RemoteRef` is an implementation of a more general interface, called `Channel`.
`Channel`s and shared memory (via `mmap`) are the basic building blocks for
communication among tasks.

**In [12]:**

{% highlight julia %}
f = @spawn rand(10) # remote call
fetch(f)            # fetch
{% endhighlight %}




    10-element Array{Float64,1}:
     0.975851
     0.757001
     0.327111
     0.852844
     0.150347
     0.57112  
     0.420745
     0.0726167
     0.406822
     0.196387



To run a remote call, the called function must be known to all the processes.
This is not the case for `pollard_rho`.

In general, we should run the cluster at the startup, and load our functions from a module.
Here I will re-evaluate that function using the `everywhere` macro, which runs its argument everywhere.

**In [13]:**

{% highlight julia %}
@everywhere g(x, n) = (x^2 - 1) % n

@everywhere function pollard_rho(n)
    x, y, d = 2, 2, 1

    if n % 2 == 0
        d = 2
    elseif n % 3 == 0
        d = 3
    else
        while d == 1
            x = g(x, n)
            y = g(g(y, n), n)
            d = gcd(abs(x-y), n)
        end
    end
    d, div(n, d)
end
{% endhighlight %}

**In [14]:**

{% highlight julia %}
f = @spawn pollard_rho(100)
fetch(f)
{% endhighlight %}




    (2,50)



The API we are going to implement is the same of `trial_division`: from the point of view of the user, it produces lazily the results. Under the hood, it will run in parallel the factorization.

To do this we need a scheduler function, which manages the execution of `pollard_rho` on new factors, as soon as they
are found. This is where coroutines show their power.

## Channels, and more on Coroutines

The basic data structure to handle communication and syncronization among tasks is **Channel**.
Channels are shared queues that can be accessed by multiple readers, via `fetch` or `take!`,
and by multiple writers, via `put!`.

The size of Channels is finite. If there are no objects available, or if the channel is full,
reading or writing are blocking operation.

A `RemoteRef` is a Channel of size 1.


To coordinate the remote execution of pollard_rho, we will use again coroutines. Remember? Julia runtime
provides a scheduler to coordinate the execution of multiple coroutines. The control flow switches from a coroutine
to another in case of a blocking operation, like reading from a channel, or by using `yield` or `produce`.

Our scheduler has the responsibility of running `pollard_rho`, to find new factors, and synchronizing the execution
of the tasks.

`pollard_rho_factorization` works in this way:

1.    It creates and returns a Channel `primes`. Coroutines will push on it new prime numbers. When everything is done, it will be closed. The user will get primes iterating over it

2.    It creates a task to run `factor`. The macro `@schedule` wraps the statement it receives in a Task, and schedules it. Once `factor` is done, it closes the `primes` channel. At that point the loop will finish

3.    `factor` creates a new task, using `@sync`. `@sync` is like `@schedule`, but it terminates once all tasks created in it are done

4.    this task runs remotely `pollard_rho`.  `fetch` is blocking, so at that point the scheduler resumes another coroutine. Depending on the result, it decides whether to push the result to the channel, whether to create and schedule new coroutines running `factor` on it. `@async` creates and schedules coroutines, but it does not wait for their termination. Here we actually creates and runs two parallel tasks.

**In [15]:**

{% highlight julia %}
function pollard_rho_factorization(n)

    function factor(n, primes)
        @sync begin
            ref = @spawn pollard_rho(n)
            x, y = fetch(ref)
            if x == 1
                put!(primes, y)
            elseif y == 1
                put!(primes, x)
            else
                @async factor(x, primes)
                @async factor(y, primes)
            end
        end
    end

    primes = Channel()
    @schedule begin
        factor(n, primes)
        close(primes)
    end
    primes
end
{% endhighlight %}




    pollard_rho_factorization (generic function with 1 method)



**In [16]:**

{% highlight julia %}
test = reduce(*, map(fermat, 1:6))
print("$test = 1")
for factor in pollard_rho_factorization(test)
    print(" * $factor")
end
{% endhighlight %}

    113427455640312821154458202477256070485 = 1 * 5 * 17 * 257 * 641 * 274177 * 65537 * 6700417 * 67280421310721

[interfaces]: http://docs.julialang.org/en/release-0.4/manual/interfaces/
[composite-types]: http://docs.julialang.org/en/release-0.4/manual/types/#composite-types
[trial]: https://en.wikipedia.org/wiki/Trial_division
[tasks]: http://docs.julialang.org/en/release-0.4/manual/control-flow/#man-tasks
[rho]: https://en.wikipedia.org/wiki/Pollard%27s_rho_algorithm
[fermat]: https://en.wikipedia.org/wiki/Fermat_number
[parallel]: http://docs.julialang.org/en/release-0.4/manual/parallel-computing/
[lenstra]: https://en.wikipedia.org/wiki/Lenstra_elliptic_curve_factorization
[elliptic_group]: https://en.wikipedia.org/wiki/Elliptic_curve#The_group_law
[elliptic_mult]: https://en.m.wikipedia.org/wiki/Elliptic_curve_point_multiplication
[workspace]: https://github.com/vdelia/juliabox_workspace
[juliabox]: https://www.juliabox.org/
