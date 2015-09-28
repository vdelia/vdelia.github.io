---
layout: post
title: "microkanren with python"
tags:
    - python
    - notebook
    - logic-programming
---
# A python implementation of  $$ \mu $$ Kanren

[ $$ \mu $$ Kanren][micro] (microKanren) 
is a minimalistic relational programming language, introduced as a stripped-down implementation of [minikanren][minikanren].

What is a  **relational programming language**? In other programming paradigms, a program is a set of operations (functions, statements, etc.) which consume *inputs* to produce *outputs*.

A relational program is a set of predicates on (logic) variables. Running it means searching the values that can be assigned to logic variables so that those predicates hold. It does not return a value, but it enumerates solutions. Moreover, there is no distinction between *inputs* and *outputs* of relations.

To me relational programming is a way to extract a usable and pure logical subset of prolog, and bring it to the masses.
[Other people][lp-overrated] see it as DSLs for brute-force search. Everybody agrees that [this presentation][prez-byrd] is mind-blowing.

Where does it come from? In [The Reasoned Schemer][reasoned-schemer], the authors introduced relational programming as a **natural extension of functional programming**. 
They show how to **embed a logic interpreter** into [Scheme][racket].

While Scheme is the reference host for all the *\*kanren*s, currently there are many implementations, in many different host languages. The most succesful is probably [clojure/core.logic][core.logic].

This post is actually a ipython notebook where I implement  $$ \mu $$ Kanren and some syntactic sugar in **python**.
The notebook file is available [here]({{ site.url }}/assets/kanren/ukanren.ipynb).
It is meant to be really interactive: I redefine multiple times several functions to get a more and more friendly API.


The first section contains an implementation of  $$ \mu $$ kanren in python; in the second section I introduce some syntactic sugar to make it more similar to miniKanren; the last sections contain some examples of what kind of programs can be written with it.

##  $$ \mu $$ Kanren core

A  $$ \mu $$ Kanren program can be interpreted as a query. Given a set of relations among items and variables, we ask to the interpreter to find the variable substitutions so that those relations are valid.

From the [paper][micro-paper]

> A  $$ \mu $$ Kanren program proceeds through the application of a **goal** to a **state**. Goals are often understood by analogy to predicates. Whereas the application of a predicate to an element of its domain can be either true or false, a goal pursued in a given state can either succeed or fail.

> A **state** is a pair of a substitution (represented as a dictionary) and a non-negative integer representing a fresh variable counter.

### Logic variables

A `logic_variable` is an object. It is identified by an index: two `logic_variable`s are the same if they share the same index.

**In [1]:**

{% highlight python %}
import collections

logic_variable = collections.namedtuple("logic_variable", ["index"])

def is_logic_var(x):
    return isinstance(x, logic_variable)
{% endhighlight %}

To create new `logic_variable`s, we use the *fresh variable counter* to generate those indexes.

### Substitutions

Substitutions are represented by python dictionaries whose keys  $$ k $$  are `logic_variable`s, and whose values  $$ v $$  are items that can replace that variable.

**In [2]:**

{% highlight python %}
class SubstitutionMap(dict):
    pass
{% endhighlight %}

A `logic_variable`  $$ X $$  can be

*   free, i.e.  $$ X $$  not in SubstitionMap, and the relations are valid predicates for all  $$ X $$ 
*   bound to another `logic_variable`  $$ Y $$ :  $$ X $$  is equivalent to  $$ Y $$ , and SubstitutionMap[ $$ X $$ ] =  $$ Y $$ 
*   bound to an item  $$ i $$ : predicates hold when  $$ X $$  is replaced by  $$ i $$ 

We will see later what are the terms of the language, and so what are the items  $$ i $$ .

The **walk** method searches for a term's value. If the term is a value (i.e. not a logic variable), then it returns the value itself. Otherwise, it traverses the chain of substitutions until it finds a value.

The **ext_s** is factory method that adds a new binding to an existing SubstitutionMap.

**In [3]:**

{% highlight python %}
class SubstitutionMap(dict):
    
    def walk(self, var):
        while is_logic_var(var) and var in self:
            var = self[var]
        return var

    def ext_s(self, var, value):
        s = SubstitutionMap(self)
        s[var] = value
        return s
{% endhighlight %}

**In [4]:**

{% highlight python %}
# test SubstitutionMap
x, y = logic_variable(0), logic_variable(1)

assert x == SubstitutionMap().walk(x), "x is free"

assert 1 == SubstitutionMap({x: 1}).walk(x), "x is bound to 1"

assert "something else" == SubstitutionMap({x: y})\
                           .ext_s(y, "something else")\
                           .walk(x), "walk must traverse the chain of substitutions"
{% endhighlight %}

The *fresh variable counter* is an int starting from 0, and it is used throught the evaluation to get new unique indexes for logic variables.

Initially, there are no substitutions, and the counter is 0. That state is called `empty_state`.

**In [5]:**

{% highlight python %}
empty_state = (SubstitutionMap(), 0)
{% endhighlight %}

### The terms of the language: `unify`

The function `unify` defines which are the terms of the language.
It takes as arguments two objects,  $$ x $$  and  $$ y $$ , and a SubstitutionMap *substitutions*.

Its objective is to add new bindings to *substitutions*, so that  $$ x $$  and  $$ y $$  becomes equivalent, i.e. 
```
unifying = unify(x, y, substitution)
unifying.walk(x) == unifying.walk(y)
```

If there is no way to get  $$ x $$  equivalent to  $$ y $$ , for example because they are both already bound to terms
not equivalent, then it `unify` returns `None`. In this case we say that  $$ x $$  and  $$ y $$  do not unify under
`substitution` , i.e.

```
unify(x, y, substitution) == None
```

In this implementation, valid terms are python objects. At the beginning `unify` walks the two arguments
by using the SubstitutionMap. Then it checks the resulting terms. The two objects unify if
*   they are equals according to the `==` operator; then we don't need to add anything to *substitution*
*   one is a `logic_variable`  $$ v $$ , and in that case  $$ v $$  must be substituted with the other term to get the unification working
*   they are sequences, and in that case the unification is recursively applied term by term.

**In [6]:**

{% highlight python %}
def _concrete_unify(x, y, substitutions):
    if substitutions is None:
        return None

    x = substitutions.walk(x)
    y = substitutions.walk(y)

    if x == y:
        return substitutions
    elif is_logic_var(x):
        return substitutions.ext_s(x, y)
    elif is_logic_var(y):
        return substitutions.ext_s(y, x)
    elif is_sequence(x) and is_sequence(y):
        return unify_sequences(x, y, substitutions)
    return None

def unify(x, y, substitutions):
    #  This is because I will modify it later
    return _concrete_unify(x, y, substitutions)
{% endhighlight %}

Sequences are objects with the `__iter__` method. This is orthogonal to the rest of  $$ \mu $$ Kanren. To handle new kind of terms, we must update the `unify` function.

**In [7]:**

{% highlight python %}
import itertools as it

def is_sequence(o):
    #  I want it to fail on strings
    return not is_logic_var(o) and hasattr(o, '__iter__')

def unify_sequences(xs, ys, substitutions):
    if len(xs) != len(ys):
        return None
    for a, b in it.izip(xs, ys):
        substitutions = unify(a, b, substitutions) 
    return substitutions
{% endhighlight %}

**In [8]:**

{% highlight python %}
# Test unify
x, y = logic_variable(0), logic_variable(1)
assert unify(x, y, SubstitutionMap()) ==\
       SubstitutionMap({x: y}), "to unify x and y, substitute y to x"

assert unify(x, y, SubstitutionMap({x: "value x", y: "value y"}) ) is None, "they cannot be equivalent"
assert unify(x, 5, SubstitutionMap()) == SubstitutionMap({x: 5})
assert unify((x, 1, y), (1, 2, 3), SubstitutionMap()) is None, "sequences unify term-by-term"
assert unify((x, 1, y), (1, 1, 3), SubstitutionMap()) ==\
       SubstitutionMap({x: 1, y: 3}), "sequences unify term-by-term"
{% endhighlight %}

## Goals/Predicates builder

A *goal* is a function which takes as argument a state, i.e. a pair (`SubstitutionMap`, *fresh variable counter*), and returns a list of states which satisfy the goal.
In this implementation, *goal*s are generators yielding the valid states. If the generator is empty, then that goal
does not succeed.

 $$ \mu $$ Kanren has four primitive goals builder: `equiv`, `call_fresh`, `disj` and `conj`.

### `Equiv`
 `equiv` builds a goal which succeeds if its two arguments unify, i.e. it yields the substitutions which make its arguments unify

**In [9]:**

{% highlight python %}
def equiv(x, y):
    def _goal((substitutions, fresh_var_counter)):
        unifying = unify(x, y, substitutions)
        if unifying is not None:
            yield (unifying, fresh_var_counter)
    return _goal
{% endhighlight %}

**In [10]:**

{% highlight python %}
# Test equiv
x, y = logic_variable(0), logic_variable(1)
assert list(equiv(x, 5)(empty_state)) == [(SubstitutionMap({x: 5}), 0)]
assert list(equiv(x, y)(empty_state)) == [(SubstitutionMap({x: y}), 0)]
{% endhighlight %}

### `call_fresh`
The call/fresh goal constructor creates a new `logic_variable`.

It takes as argument a unary function  $$ f $$ , which must return a goal, and it returns
a new goal which runs  $$ f $$  by binding its argument to a new `logic_variable`.
It leaves unchanged the substitutions, but it increments the *fresh variable counter*,
to ensure unicity of variables indexes.

**In [11]:**

{% highlight python %}
def call_fresh(f):
    def _new_goal((substitutions, fresh_var_counter)):
        return f(logic_variable(fresh_var_counter))((substitutions, fresh_var_counter+1))
    return _new_goal

{% endhighlight %}

As shown in the following tests, if I want to introduce a new logic variable in a goal  $$ G $$ :
*   I wrap the goal  $$ G $$  in a unary function  $$ f $$ 
*   I use the sole argument of  $$ f $$  as logic_variable in  $$ G $$ 
*   I pass  $$ f $$  to `call_fresh` and I use the resulting goal

This is quiet verbose: the API will be simplified with the operator `fresh`. Since this is not
part of the core of  $$ \mu $$ Kanren, we will see that later.

**In [12]:**

{% highlight python %}
# test call_fresh
x, y = logic_variable(0), logic_variable(1)
def is_five(new_var):
    return equiv(new_var, 5)
goal = call_fresh(is_five)
assert list(goal(empty_state)) ==\
       [(SubstitutionMap({x: 5}), 1)], "x must be 5"

# for every new variable, I need a new call_fresh
def f(var):
    def _g(other_var):
        return equiv(var, other_var)
    return call_fresh(_g)
goal = call_fresh(f)

assert list(goal(empty_state)) ==\
       [(SubstitutionMap({x: y}), 2)], "x and y are equivalent but not bound to a term"

{% endhighlight %}

### `disj`

`disj` takes as arguments some goals, and it returns a new goal which succeeds for a given state  $$ s $$  if either succeeds for that state  $$ s $$ .

I use `ANY` as synonim of `disj`, since it is equivalent to the built-in function `any`.

**In [13]:**

{% highlight python %}
def disj(*goals):
    def _new_goal(state):
        return it.chain.from_iterable(g(state) for g in goals)
    return _new_goal

ANY = disj
{% endhighlight %}

**In [14]:**

{% highlight python %}
# test disj
x, y = logic_variable(0), logic_variable(1)

def g_any_ok(var_x, var_y):
    """
    It succeeds if x is 2, or y is 3 or x is y
    """
    return ANY(equiv(var_x, 2),
               equiv(var_y, 3),
               equiv(var_x, var_y))

## 2 variables, so 2 call_fresh(unary function)
r = list(call_fresh(lambda x: call_fresh(lambda y: g_any_ok(x, y)))(empty_state))
assert len(r) == 3 \
        and (SubstitutionMap({x: 2}), 2) in r \
        and (SubstitutionMap({y: 3}), 2) in r\
        and ({x: y}, 2) in r, "Three solutions must be returned"
{% endhighlight %}

### `conj`
`conj` takes as arguments some goals, and it returns a new goal which succeeds for a given state  $$ s $$ 
if all of them succeed for that state  $$ s $$ . 

I use `ALL` as synonim of `conj`, since it is equivalent to the built-in function `all`.

**In [15]:**

{% highlight python %}
def make_stream(goal, s):
    return it.chain.from_iterable(it.imap(goal, s))

def conj(*goals):
    """
    It returns a goal which succeeds if all the goals passed as argument succeed
    for that state.
    """
    def _new_goal(state):
        stream = goals[0](state)
        for g in goals[1:]:
            stream = make_stream(g, stream)
        return stream
    return _new_goal

ALL = conj
{% endhighlight %}

**In [16]:**

{% highlight python %}
# test disj
x, y = logic_variable(0), logic_variable(1)

def g_all_ko(var_x, var_y):
    """
    It succeeds if x is 2, or y is 3 or x is y
    """
    return ALL(equiv(var_x, 2),
               equiv(var_y, 3),
               equiv(var_x, var_y))

# fresh will fix this mess
r = list(call_fresh(lambda x: call_fresh(lambda y: g_all_ko(x, y)))(empty_state))
assert len(r) == 0, "x is not equivalent to y"

def g_all_ok(var_x, var_y):
    """
    It succeeds if x is 2 or 3, and y is 3 and y is x
    """
    return ALL(ANY(equiv(var_x, 2), equiv(var_x, 3)),
               equiv(var_y, 3),
               equiv(var_x, var_y))

r = list(call_fresh(lambda x: call_fresh(lambda y: g_all_ok(x, y)))(empty_state))
assert r == [(SubstitutionMap({x: 3, y: 3}), 2)], "x cannot be 2"
{% endhighlight %}

**That's it**. This is the core of  $$ \mu $$ Kanren.

In the next session we will add some syntactic sugar implemented in minikanren, which is very convenient to use what we implemented as an actual language.

## Minikanren sugar

Now that we have the core of the system, it is time to add some sugar to make it pleasant to use.

### `fresh`
`fresh` is a `call_fresh` without the single-variable limitation.

Technically, I use the modules `inspect` and `functools` to apply the same pattern above.

**In [17]:**

{% highlight python %}
import inspect
from functools import partial

def fresh_helper(curried_goal, nargs):
    if nargs <= 1:
        return call_fresh(curried_goal)
    else:
        return call_fresh(lambda x: fresh_helper(partial(curried_goal, x), nargs-1))
    
def fresh(variadic_goal):
    # Discover now many arguments it has, and apply the pattern
    # wrap in a function -> pass it to call_fresh
    positional_args = inspect.getargspec(variadic_goal).args
    return fresh_helper(variadic_goal, len(positional_args))
{% endhighlight %}

**In [18]:**

{% highlight python %}
# test fresh         
# I use here the same functions used to test ALL. The API here is soooo much better
assert list(fresh(g_all_ko)(empty_state)) == []
assert list(fresh(g_all_ok)(empty_state)) == \
       [(SubstitutionMap({x: 3, y: 3}), 2)], "x cannot be 2"
{% endhighlight %}

### `run`

`run` calls `fresh` for us. It takes a goal as arguments and it yields the substitutions
that make the goal succeed.

It hides `fresh` and the *fresh variable counter*.
In the next section we will redefine it to hide also the `SubstitionMap`.

**In [19]:**

{% highlight python %}
def run(goal_builder):
    for s, _ in fresh(goal_builder)(empty_state):
        yield s
        
list(run(g_any_ok))
{% endhighlight %}




    [{logic_variable(index=0): 2},
     {logic_variable(index=1): 3},
     {logic_variable(index=0): logic_variable(index=1)}]



### `reify`

`reify` translates the internal representation of variables
to a human readable form.

The reification starts by `walk`ing a variable in the `SubstitutionMap`.

If the result is a `logic_variable`, then that variable could be free,
or not bound to an item but equivalent to another `logic_variable`.
To represent this, I use the objects `free_value`s. They contain an id,
and two `free_value`s are equivalent if that id is the same.

Sequences are reified to lists, where each element is the reified version
of the item of original sequence.

Scalar items are reified to their value.

**In [20]:**

{% highlight python %}
free_value = collections.namedtuple("free_value", ["id"])

def reify_sequence(subs, objects):
    return [reify(subs, o) for o in objects]

def reify(substitutions, o):
    o = substitutions.walk(o)
    if is_logic_var(o):
        return free_value(o.index)
    elif is_sequence(o):
        return reify_sequence(substitutions, o)
    else:
        return o
{% endhighlight %}

**In [21]:**

{% highlight python %}
# test reify
x, y, z = logic_variable(0), logic_variable(1), logic_variable(2)
empty_sm = SubstitutionMap()
test_sm = SubstitutionMap({x: 2, y: z, z: 3})

assert free_value(id=0) == reify(empty_sm, x), "x is a free variable"
assert 3  == reify(test_sm, z) == reify(test_sm, y), "y and z walk to 3"
assert reify(test_sm, logic_variable(12)) != \
       reify(test_sm, logic_variable(logic_variable(2))), "free variables not equivalent"
{% endhighlight %}

### `run`, reworked

I let `run` handle the reification, and hide completely the `SubstitutionMap`.

The new `run` takes as argument a variadic function returning a goal.

It uses `fresh` to create `logic_variable`s, and then it yields all the solutions which make the goal
succeed. A solution is a tuple containing the reified variables in the same order of the goal builder arguments.

I exploit the fact that variable indexes match the order of the arguments they are bound to.

**In [22]:**

{% highlight python %}
def run(goal_builder, stop=None):
    args = inspect.getargspec(goal_builder).args
    var_indexes = range(len(args))
    substitutions = fresh(goal_builder)(empty_state)
    for s, _ in it.islice(substitutions, stop):
        # it reifies the variables in the order of corresponding
        # arguments of goal
        yield tuple(reify(s, logic_variable(idx)) for idx in var_indexes)
{% endhighlight %}

**In [23]:**

{% highlight python %}
# test run

solutions = list(sorted(run(g_any_ok)))
assert solutions == [(2, free_value(id=1)), # g_any_ok(2, whatever)
                     (free_value(id=0), 3), # g_any_ok(whatever, 3)
                     (free_value(id=1), free_value(id=1))] # g_any_ok(whatever, the same)
{% endhighlight %}

### `conde`
The last predicate I add is `conde`. It is a `if-then-else` construct. The first example shows how it works.

**In [24]:**

{% highlight python %}
def conde(*disjs):
    """
    conde is a form of if-then-else construct
    conde(
        [if-clause1 then],
        [if-clause2 then],
    )
    """
    return ANY(
        *[ALL(*conjs) for conjs in disjs]
    )
{% endhighlight %}

The language is sweet enough to run some examples!

# Example: querying Game of Thrones

In this first example, I use what we developed to run some query over a db of [Game of Thrones][got] characters.

How does the language work?
*   Combine `equiv` and `conde` (or `ALL`, `ANY`) to create new predicate.
*   If you need new `logic_variable`s, wrap the predicate in a python function, and use `fresh` or `run`
    to build them
*   Use `run` to iterate over the solutions

**In [25]:**

{% highlight python %}
got_characters = {0: ('catelyn', 'tully'),
                  1: ('eddard', 'stark'),
                  2: ('sansa', 'stark'),
                  3: ('benjen', 'stark'),
                  4: ('robb', 'stark'),
                  5: ('joffrey', 'baratheon'),
                  6: ('stannis', 'baratheon'),
                  7: ('cersei', 'lannister'),
                  8: ('tyrion', 'lannister'),
                  9: ('tommen', 'baratheon'),
                  10: ('jon', 'snow'),
                  11: ('myrcella', 'baratheon'),
                  12: ('tywin', 'lannister'),
                  13: ('jaime', 'lannister'),
                  14: ('rickon', 'stark'),
                  15: ('arya', 'stark'),
                  16: ('brandon', 'stark'),
                  17: ('renly', 'baratheon'),
                  18: ('robert', 'baratheon')}
{% endhighlight %}

We follow this **naming convention**: predicates are suffixed with the `o` character.

I write the relation `charactero(characterid, name, surname)`, which succeeds
when that triple identifies a character of Game of Thrones.

**In [26]:**

{% highlight python %}
def charactero(characterid, name, surname):
    # A simplified version
    return conde(
        # if characterid is 16, then name is 'brandon', and surname is 'stark'
        [equiv(characterid, 16), equiv(name, 'brandon'), equiv(surname, 'stark')],
        # or if characterid is 17, then name is 'renly' and surname is 'baratheon'
        [equiv(characterid, 17), equiv(name, 'renly'), equiv(surname, 'baratheon')])

# I can define it by exploiting the got_characters db
def charactero(characterid, name, surname):
    return conde(*[[equiv(characterid, k), equiv(name, n), equiv(surname, s)]
                   for (k, (n, s)) in got_characters.iteritems()])
{% endhighlight %}

Now I can `run` some queries on it. To do that, I need to pass some logic variables to the predicate, and let  $$ \mu $$ Kanren find those values. I use python functions and `run` to get new logic variables and perform the search.

*   What are name and surname of the character whose id is 16? Find  $$ X $$  and  $$ Y $$  s.t. `charactero(16, X, Y)`.

**In [27]:**

{% highlight python %}
#  logic variables
for name, surname in run(lambda x, y: charactero(16, x, y)):
    print name, surname, "has id 16"
{% endhighlight %}

    brandon stark has id 16


*   What are id and name of the characters whose surname is 'baratheon'?  Find  $$ X $$  and  $$ Y $$  s.t. `charactero(X, Y, 'baratheon')`.

**In [28]:**

{% highlight python %}
for _id, name in run(lambda x, y: charactero(x, y, 'baratheon')):
    print _id, name, "is a 'baratheon'"
{% endhighlight %}

    5 joffrey is a 'baratheon'
    6 stannis is a 'baratheon'
    9 tommen is a 'baratheon'
    11 myrcella is a 'baratheon'
    17 renly is a 'baratheon'
    18 robert is a 'baratheon'


In GoT there are several houses. The predicate `id_houseo(house_name, characterid)`, is relation among character ids and houses. A character can belong to more houses.

**In [29]:**

{% highlight python %}
got_houses = {'stark': [0, 1, 2, 3, 4, 10,  15, 16],
              'tully': [0],
              'lannister': [5, 7, 8, 9, 11, 12, 13],
              'baratheon': [5, 7, 9, 11, 6, 18]}

def id_houseo(house_name, characterid):
    # house_name is k and characterid is v for all k,v
    return conde(*[[equiv(house_name, k), equiv(characterid, v)]
                   for (k, vs) in got_houses.iteritems() for v in vs])
{% endhighlight %}

*   The ids of the characters who belong to the house baratheon

**In [30]:**

{% highlight python %}
print 'house of baratheon: ',
for _id, in run(lambda i: id_houseo('baratheon', i)):
    print _id, 
{% endhighlight %}

    house of baratheon:  5 7 9 11 6 18


*    $$ X $$  such that house name is 'baratheon' and character id is 20

**In [31]:**

{% highlight python %}
for x in run(lambda x: id_houseo('baratheon', 20)):
    print x
{% endhighlight %}

No value of  $$ X $$  can satisfy the predicate


*    $$ X $$  such that house name is 'baratheon' and character id is 5

**In [32]:**

{% highlight python %}
for x, in run(lambda x: id_houseo('baratheon', 5)):
    print x
{% endhighlight %}

    free_value(id=0)


Every value of  $$ X $$  satisfies the predicate.


I want to hide the character id, and write the predicate `houseo`, which creates the relation among house names, and name and surname of characters.

To do so, I need to find the character ids in relation with a house, and then find name and surname of that character id.

**In [33]:**

{% highlight python %}
def houseo(house_name, name, surname):
    # I need a new logic variable, characterid. Thus, I create a function and use fresh to get it
    def _f(characterid):
        # if, for any characterid, its house name is house_name, then name and surname are the one of charactero
        return conde([id_houseo(house_name, characterid), charactero(characterid, name, surname)])
    return fresh(_f)
{% endhighlight %}

*   Find all house names and surnames of the character called 'joffrey'

**In [34]:**

{% highlight python %}
for house, surname in run(lambda house, surname: houseo(house, 'joffrey', surname)):
    print 'joffrey', surname, 'belongs to house', house
{% endhighlight %}

    joffrey baratheon belongs to house baratheon
    joffrey baratheon belongs to house lannister


*   Find all house names and first name of characters whose surname is the name of the house they belong to

**In [35]:**

{% highlight python %}
for house, name in run(lambda house, name: houseo(house, name, house)):
    print name, house, '=> house ', house

# no jon snow
{% endhighlight %}

    joffrey baratheon => house  baratheon
    tommen baratheon => house  baratheon
    myrcella baratheon => house  baratheon
    stannis baratheon => house  baratheon
    robert baratheon => house  baratheon
    cersei lannister => house  lannister
    tyrion lannister => house  lannister
    tywin lannister => house  lannister
    jaime lannister => house  lannister
    eddard stark => house  stark
    sansa stark => house  stark
    benjen stark => house  stark
    robb stark => house  stark
    arya stark => house  stark
    brandon stark => house  stark
    catelyn tully => house  tully


# Example: Data structures

In this example I add [`cons`es][cons] to  $$ \mu $$ Kanren. Then we will play with the very interesting `membero` and `appendo` relations.

## `cons` predicates

**In [36]:**

{% highlight python %}
cons = collections.namedtuple("cons", ["car", "cdr"])
nil = ()

def emptyo(d):
    """
    d is empty if it is nil
    """
    return equiv(d, nil)

def conso(a, b, acons):
    """
    a and b forms a cons, when acons is equivalent to cons(a, b)
    """
    return equiv(cons(a, b), acons)

def firsto(acons, elt):
    """
    if elt if the first element of acons then
    conso(elt, tail, acons) holds for all tails
    """
    return fresh(lambda tail: conso(elt, tail, acons))

def tailo(acons, tail):
    """
    if tail is the tail of acons then conso(first, tail, acons)
    holds for all firsts
    """
    return fresh(lambda first: conso(first, tail, acons))
{% endhighlight %}

*   the  $$ C $$  such that  $$ C $$  is cons of 1 and 2

**In [37]:**

{% highlight python %}
for C, in run(lambda C: conso(1, 2, C)):
    print 'cons 1 and 2 to get', C
{% endhighlight %}

    cons 1 and 2 to get [1, 2]


*   the cons such that 1 is its first element

**In [38]:**

{% highlight python %}
for C, in run(lambda x: firsto(x, 1)):
    print 1, 'is the first element of', C
{% endhighlight %}

    1 is the first element of [1, free_value(id=1)]


We can write basic operations, but to make it fully usable, we need to improve the syntax.

What I want to do is to treat python lists as if they were `cons`es, and traslate back to list when showing the result.
The former goal can be achieved by modifying `unify`. Remember? `unify` is where items are defined. The latter by `reify`-ing
correctly the `cons`es

## `unify` python lists to conses

I modify (as little as possibile) `unify`, and as a consequence I need to evaluate again `equiv`.

**In [39]:**

{% highlight python %}
def consify(lst):
    newcons = nil
    for i in reversed(lst):
        newcons = cons(i, newcons)
    return newcons

def unify(x, y, substitutions):
    if isinstance(x, list):
        x = consify(x)
    if isinstance(y, list):
        y = consify(y)
    return _concrete_unify(x, y, substitutions)

def equiv(x, y):
    def _goal((substitutions, fresh_var_counter)):
        unifying = unify(x, y, substitutions)
        if unifying is not None:
            yield (unifying, fresh_var_counter)
    return _goal

assert unify([1, 2, 3], cons(1, cons(2, cons(3, nil))), SubstitutionMap()) == SubstitutionMap()
{% endhighlight %}

## `reify` conses to python lists

I need to evaluate again `run` also.

**In [40]:**

{% highlight python %}
def reify_cons(substitutions, acons):
    lst = []
    while isinstance(acons, cons):
        car, cdr = acons
        lst.append(reify(substitutions, car))
        acons = substitutions.walk(cdr)
    if acons:
        lst.append(reify(substitutions, acons))
    return lst

def reify(substitutions, o):
    o = substitutions.walk(o)

    if is_logic_var(o):
        return free_value(o.index)
    elif isinstance(o, cons):
        return reify_cons(substitutions, o)
    elif is_sequence(o):
        return reify_sequence(substitutions, o)
    else:
        return o
    
def run(goal_builder, stop=None):
    args = inspect.getargspec(goal_builder).args
    var_indexes = range(len(args))
    substitutions = fresh(goal_builder)(empty_state)
    for s, _ in it.islice(substitutions, stop):
        # it reifies the variables in the order of corresponding
        # arguments of goal
        yield tuple(reify(s, logic_variable(idx)) for idx in var_indexes)
{% endhighlight %}

**In [41]:**

{% highlight python %}
# test the new reify
empty_sm = SubstitutionMap()

#assert range(10) == 
print reify(empty_sm, consify(range(10))), "test consify"
{% endhighlight %}

    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9] test consify


## Playing with python lists

In this section I will run all examples on the python list `ten`

**In [42]:**

{% highlight python %}
ten = range(10)
{% endhighlight %}

*   Find  $$ f $$ ,  $$ t $$ , s.t.  $$ f $$  is the first of `ten` and  $$ t $$  is the tail of `ten`

**In [43]:**

{% highlight python %}
for f, t in run(lambda f, t: ALL(
        firsto(ten, f),
        tailo(ten, t))):
    print f, 'is the first of', ten
    print t, 'is the tail of', ten
{% endhighlight %}

    0 is the first of [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    [1, 2, 3, 4, 5, 6, 7, 8, 9] is the tail of [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]


*   Find all  $$ x $$  and  $$ acons $$ , s.t.  $$ acons $$  is the `cons(x, x)`

**In [44]:**

{% highlight python %}
for x, L in run(lambda x, L: ALL(
        firsto(L, x),
        tailo(L, x))):
    print x, 'is first and tail of', L
{% endhighlight %}

    free_value(id=2) is first and tail of [free_value(id=2), free_value(id=2)]


## `membero`

`membero` is the relational version of the python `in` operator. If `membero(collection, elt)` succeeds, then `elt` is in `collection`.

**In [45]:**

{% highlight python %}
def membero(collection, elt):
    def _f(tail): #  for every tail
        return conde(
            [firsto(collection, elt)], # if elt is first of collection it's fine
            [tailo(collection, tail), membero(tail, elt)])  # if tail is tail of collection, then elt must be member of tail
    return fresh(_f)
{% endhighlight %}

*   find all  $$ x $$  s.t. 30 is member of `ten`. Such an  $$ x $$  does not exist.

**In [46]:**

{% highlight python %}
for x in run(lambda x: membero(ten, 30)):
    print x
{% endhighlight %}

*   find all  $$ x $$  s.t. 2 is member of `ten`. All  $$ x $$ s are fine here.

**In [47]:**

{% highlight python %}
for x in run(lambda x: membero(ten, 2)):
    print x
{% endhighlight %}

    (free_value(id=0),)


*   find all  $$ x $$  s.t.  $$ x $$  is member of `ten`. We enumerate the values in `ten`.

**In [48]:**

{% highlight python %}
for elt,  in run(lambda elt: membero(ten, elt)):
    print elt, "is in ", ten
{% endhighlight %}

    0 is in  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    1 is in  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    2 is in  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    3 is in  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    4 is in  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    5 is in  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    6 is in  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    7 is in  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    8 is in  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    9 is in  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]


*   We can inspect the structure itself of the list. Find  $$ x $$ s s.t. [4, 65, 9] is member of [1, 2,  $$ x $$ ].

**In [49]:**

{% highlight python %}
for (solution,) in run(lambda x: membero([1, 2, x], [4, 65, 9]), stop=5):
    print [4, 65, 9], 'is member of', [1, 2, solution]
{% endhighlight %}

    [4, 65, 9] is member of [1, 2, [4, 65, 9]]


*   we can look for more variables

**In [50]:**

{% highlight python %}
for x, y in run(lambda x, y: membero([1, 2, x], [1, y, 3])):
    print '[1, y, 3] member of [1, 2, x] iff x=%s and y=%s' % (x, y)
{% endhighlight %}

    [1, y, 3] member of [1, 2, x] iff x=[1, free_value(id=1), 3] and y=free_value(id=1)


*   and let it generate the whole list

**In [51]:**

{% highlight python %}
for (L, ) in run(lambda L: membero(L, 1), stop=3):
    print 1, 'member of', L
{% endhighlight %}

    1 member of [1, free_value(id=2)]
    1 member of [free_value(id=2), 1, free_value(id=4)]
    1 member of [free_value(id=2), free_value(id=4), 1, free_value(id=6)]


## `appendo`

`appendo(begin, end, collection)` holds when `collection` is the concatenation of the list `begin`
and the list `end`.

Note that `conso` is a relation among a scalar and two lists, while `appendo` among three lists.

**In [52]:**

{% highlight python %}
def appendo(begin, end, collection):
    """
    begin + end = collection
    """
    def _f(x, y, z):
        return conde(
            # if begin is empty, then collection is end
            [emptyo(begin), equiv(end, collection)],
            # otherwise
            [conso(x, y, begin),      # begin is [x] + y
             conso(x, z, collection),  # collection is [x] + z
             appendo(y, end, z)])     # z is end appended to y
    return fresh(_f)
{% endhighlight %}

It can be used to concatenate lists

**In [53]:**

{% highlight python %}
for (L, ) in run(lambda L: appendo([1, 2], [3,4], L)):
    print '[1, 2] + [3, 4] =', L
{% endhighlight %}

    [1, 2] + [3, 4] = [1, 2, 3, 4]


but also to go backwards.

**In [54]:**

{% highlight python %}
for (head, L) in run(lambda head, L: appendo(head, [3,4], L), stop=5):
    print '{head} + [3, 4] = {L}'.format(head=head, L=L)
{% endhighlight %}

    [] + [3, 4] = [3, 4]
    [free_value(id=2)] + [3, 4] = [free_value(id=2), 3, 4]
    [free_value(id=2), free_value(id=5)] + [3, 4] = [free_value(id=2), free_value(id=5), 3, 4]
    [free_value(id=2), free_value(id=5), free_value(id=8)] + [3, 4] = [free_value(id=2), free_value(id=5), free_value(id=8), 3, 4]
    [free_value(id=2), free_value(id=5), free_value(id=8), free_value(id=11)] + [3, 4] = [free_value(id=2), free_value(id=5), free_value(id=8), free_value(id=11), 3, 4]


**In [55]:**

{% highlight python %}
for (head, tail) in run(lambda head, tail: appendo(head, tail, ten)):
    print '{head} + {tail} = {ten}'.format(head=head, tail=tail, ten=ten)
{% endhighlight %}

    [] + [0, 1, 2, 3, 4, 5, 6, 7, 8, 9] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    [0] + [1, 2, 3, 4, 5, 6, 7, 8, 9] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    [0, 1] + [2, 3, 4, 5, 6, 7, 8, 9] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    [0, 1, 2] + [3, 4, 5, 6, 7, 8, 9] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    [0, 1, 2, 3] + [4, 5, 6, 7, 8, 9] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    [0, 1, 2, 3, 4] + [5, 6, 7, 8, 9] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    [0, 1, 2, 3, 4, 5] + [6, 7, 8, 9] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    [0, 1, 2, 3, 4, 5, 6] + [7, 8, 9] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    [0, 1, 2, 3, 4, 5, 6, 7] + [8, 9] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    [0, 1, 2, 3, 4, 5, 6, 7, 8] + [9] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9] + [] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]


Using `appendo` we can easily build arithmetics!

[micro]: https://github.com/jasonhemann/microKanren
[minikanren]: http://minikanren.org/
[lp-overrated]: http://programming-puzzler.blogspot.fr/2013/03/logic-programming-is-overrated.html
[prez-byrd]: https://www.youtube.com/watch?v=eQL48qYDwp4
[reasoned-schemer]: https://mitpress.mit.edu/index.php?q=books/reasoned-schemer
[racket]: http://racket-lang.org/
[core.logic]: https://github.com/clojure/core.logic
[micro-paper]: http://webyrd.net/scheme-2013/papers/HemannMuKanren2013.pdf
