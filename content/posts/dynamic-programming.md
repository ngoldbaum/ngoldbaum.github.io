---
title: "Dynamic Programming"
slug: dynamic-programming
description: null
date: 2019-05-24T15:24:59-04:00
type: posts
categories:
- Technical
tags:
- Interview Prep
- Algorithms
- Programming
- Python
---

Today I'm going to be pausing the Mercurial content in favor of material I
learned today in the algoganza study group here at the Recurse Center. This
study group is working together to learn content that commonly comes up in job
interviews and to prepare for the dreaded whiteboard technical
interview. Today's session focused on a new concept for me, dynamic programming,
an approach one can use to solve problems that are conceptually amenable to a
recursive approach, but where the naive recursive approach might be very slow.

## A Problem Amenable to Recursion

To explore these ideas let's think about the Fibonacci sequence. We can
calculate Fibonacci number `$n$` with the formula `$F_n = F_{n-1} +
F_{n-2}$`. For `$n=0$` and `$n=1$` we impose `$F_0 = 0$` and `$F_1 = 1$`. The
first few numbers in this sequence are `$0, 1, 1, 2, 3, 5, 8, 13, \ldots$`. This
formula is amenable to a recursive implementation because we can calculate new
numbers in the sequence using only information we collected about previous
numbers in the sequence.

We can write a recursive implementation of the `$F_n$` function in Python like
this:

```python
def fib(n):
    if n == 0:
        return 0
    if n == 1:
        return 1
    return fib(n-1) + fib(n-2)
```

This works but is *extremely* slow:

```
In [2]: %time fib(5)
CPU times: user 20 µs, sys: 1 µs, total: 21 µs
Wall time: 29.1 µs
Out[2]: 5

In [3]: %time fib(10)
CPU times: user 153 µs, sys: 8 µs, total: 161 µs
Wall time: 98.9 µs
Out[3]: 55

In [4]: %time fib(15)
CPU times: user 0 ns, sys: 2.02 ms, total: 2.02 ms
Wall time: 1.33 ms
Out[4]: 610

In [5]: %time fib(20)
CPU times: user 2.81 ms, sys: 179 µs, total: 2.99 ms
Wall time: 2.48 ms
Out[5]: 6765

In [6]: %time fib(25)
CPU times: user 51.1 ms, sys: 0 ns, total: 51.1 ms
Wall time: 49.6 ms
Out[6]: 75025

In [7]: %time fib(30)
CPU times: user 204 ms, sys: 0 ns, total: 204 ms
Wall time: 202 ms
Out[7]: 832040

In [8]: %time fib(35)
CPU times: user 2.03 s, sys: 691 µs, total: 2.03 s
Wall time: 2.03 s
Out[8]: 9227465

In [9]: %time fib(40)
CPU times: user 22.9 s, sys: 3.65 ms, total: 22.9 s
Wall time: 22.9 s
Out[9]: 102334155
```

The problem is that we are calling the `fib` function far more times than we
actually need to. Rather than calculating, say, `fib(15)` only once, we instead
calculate it over and over again for all numbers greater than 15.

## Memoization

One way to improve this approach is to use *memoization*. That is, we cache the
output of our `fib` function the first time we call the function for a given
`n`.  If we call the function again for the same `n`, we use the cached output
we saved and avoid recursively recomputing all Fibonacci numbers smaller than
`n`. The easiest way to implement memoization in Python is to use a decorator:

```python
def memoize(f):
    cache = {}
    def wrapped(n):
        if n not in cache:
            cache[n] = f(n)
        return cache[n]
    return wrapped

@memoize
def fib(n):
    if n == 0:
        return 0
    if n == 1:
        return 1
    return fib(n-1) + fib(n-2)
```

Here we've created a decorator `memoize` that stores a dictionary that caches
the results of the function, using the inputs to the function as the cache
keys. We then apply the `memoize` decorator to the `fib` function, whcih is
unchanged from above. This implementation is *substantially* faster, we're now
able to calculate `$F_{100}$` in less than a millisecond:

```
In [14]: %time fib(100)
CPU times: user 299 µs, sys: 0 ns, total: 299 µs
Wall time: 316 µs
Out[14]: 354224848179261915075
```

Memoization can be very useful if we don't mind paying the memory cost of
storing all inputs and outputs to all functions. All we need to do is create a
cache and save results to the cache. The rest of the algorithm is completely
unchanged and we still retain all the intuition we developed while thinking
about the recursive approach.

## Dynamic Fibonacci

There is a more optimal way to do this problem, using a dynamic programming
approach. To see why this might be the case, consider how the recursive and
memoized approaches we examined already are *top-down* approaches. We coded
things in a very general way such that our implementation doesn't know ahead of
time how many times the `fib` function will get called or when it will
eventually get called with arguments that trigger the terminating conditions for
the recursion (e.g. `n = 1` and `n = 0`). However, we know ahead of time that to
calculate the 40th Fibonacci number, we are definitely going to need the 0th
through 39th number. A more clever *bottom-up* algorithm takes advantage of this
knowledge. A dynamic Fibonacci solver looks like this:

```python
def fib(n):
    if n == 0:
        return 0
    if n == 1:
        return 1
    nm1 = 0
    nm2 = 1
    for i in range(2, n+1):
        fibi = nm1 + nm2
        nm1 = nm2
        nm2 = fibi
    return fibi
```

This version uses constant memory and runs in `$O(n)$` time. It also doesn't use
the call stack to store temporary variables and is not susceptible to raising
errors due to running out stack frames (in Python) or overflowing the stack and
triggering undefined behavior (in an unsafe compiled language like C).

## Making change

Let's close this out with a discussion of one more problem that is amenable to
this sort of approach. Here we'd like to know how many different ways we can
make change for a dollar using US currency (that is, using pennies, nickles,
dimes, quarters, half-dollars, and dollar coins). It may not be obvious that
this problem is amenable to a recursive approach. One way to see this is to
realize that one subset of solutions is the set of ways to make change for
99 cents along with one more penny. Another set is the set of ways to make
change for 95 cents along with another nickel, 90 cents with a dime, and so
on. All of these solutions depend on the solution of a smaller version of the
problem - a classic signal that recursion might be useful. What about the other
solutions? Well, we know that, for example, there is only one way to take a way
to make change for 99 cents and make it a way to make change for a dollar: add
another cent. So that means we've accounted for all of the ways to make a dollar
with the set of coins that includes pennies. So the other set of solutions is
the way to make change for a dollar using all coins *but* pennies. Again, we are
dealing with a smaller problem, another recursive path.

Let's take a look at Python code for this recursive algorithm:

```python
def make_change(amount, denominations):
    if len(denominations) == 0:
        return 0
    if amount == 0:
        return 1
    if amount < 0:
        return 0
    num_with_amount = make_change(amount - denominations[0], denominations)
    num_without_denom = make_change(amount, denominations[1:])
    return num_with_amount + num_without_denom
```

The terminating cases might look a little weird. The first, checking the number
of denominations to consider, handles the case where we're asked to make change
with no coins at all. There's no way to make change in this case, so the number
of ways to make change from this branch of the recursive call graph is zero. The
second case, when we're asked to make change for 0 cents, corresponds to the
case where we've made exact change already, so this branch contributes exactly
one way to make change. Finally, if we're asked to make change for a negative
amount of currency that means that again this is an invalid way to make change
for a dollar (the value of the coins goes over a dollar).

As with the Fibonacci numbers, this is a top-down approach. What would the
bottom-up dynamic approach look like? One way to do it is to make use of a table
that caches the results for simple cases and then builds up more complicated
cases as we go:

```python
import itertools

def make_change_dynamic(amount, denominations):
    # Generate all combinations of the given denominations
    # e.g. for a penny and a nickel, this would be just a penny, just a 
    # nickel, and a penny and a nickel
    combinations = []
    for l in range(1, len(denominations) + 1):
        for x in itertools.combinations(denominations, l):
            combinations.append(x)

    # table to cache results
    TABLE = {}

    # terminating case for making change for 0 cents
    for c in combinations:
        TABLE[0, c] = 1

    for i in range(1, amount + 1):
        # terminating case for making change with no money
        TABLE[i, ()] = 0
        for c in combinations:
            if i - c[0] < 0:
                # terminating case for making change for negative cents
                TABLE[i, c] = 0
            elif i - c[0] == 0:
                # exact change
                TABLE[i, c] = 1
            else:
                TABLE[i, c] = TABLE[i - c[0], c] + TABLE[i, c[1:]]

    return TABLE[amount, denominations]
```

I suspect that there's probably a fancier version that can compute the
result with constant memory like the Fibonacci case we discussed above.

## Futher Reading

Another blog post diving into some more in-depth theory:
https://blog.racket-lang.org/2012/08/dynamic-programming-versus-memoization.html

A set of dynamic programming practice problems:
https://atcoder.jp/contests/dp/tasks
