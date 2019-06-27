---
title: "Project Euler"
slug: project-euler
summary: >
  An overview of my solutions to the first 20 project euler problems.
description: null
date: 2019-06-06T10:23:45-04:00
type: posts
categories:
- Technical
tags:
- Rust
- Programming
---

# The first 20 Project Euler problems

During my first week here at RC I decided to attend a session led by another
recurser on [Project Euler](https://projecteuler.net/), an online coding
challenge based on a series of discrete math problems. The problems are written
to lend themselves to a computational solution. There are hundreds of problems
and the problems get increasingly hard past the first 100. While the early
problems are definitely easier than the later ones, they still take some doing
to finish and are a great way to quickly work on a bunch of problems to learn a
new programming language.

I'm getting more and more used to thinking about Rust code in an idiomatic way,
but I don't think I'm comfortable enough to call myself a
[rustacean](https://rustacean.net/) yet. To further my goal of oxidizing my
brain with rust knowledge, I decided to start working through project euler
problems sequentially. I've recently finished the first 20 problems and I though
I'd share the highlights of what I learned about rust along the way.

If you'd like to peruse my solutions, they're all [on
GitHub](https://github.com/ngoldbaum/project_euler).

## Functional processing pipelines

I'm used to Python, where computation tends to happen in an imperative
way. Although Python does have [functional programming
features](https://docs.python.org/3.7/howto/functional.html), real-world code
I've seen tends to avoid it or use them sparingly instead of composing an entire
computation or block of code as a functional data processing pipeline. Guido van
Rossum is also [of the
opinion](https://developers.slashdot.org/story/13/08/25/2115204/interviews-guido-van-rossum-answers-your-questions)
that some of these features should be removed from the language and [tried to
remove](https://www.artima.com/weblogs/viewpost.jsp?thread=98196) `lambda`,
`map`, `filter`, from Python 3 and succeeded in removing `reduce` as a builtin
and moving it to `functools`.

In Rust it's much more natural to process a collection of things by creating an
iterator over the collection and then applying transformations to the elements
of the iterator as needed and, if necessary, doing a reduction step to
process the transformed elements of the collection. One advantage of this
approach is that if we're handed an abstract iterator that lazily generates
data, this approach will avoid needing to hold all the elements of the iterator
in memory.

[Problem 6](https://projecteuler.net/problem=6) asks us to compute the
difference between the sum of the squares and the square of the sum of the
natural numbers up to 100. This lends itself to a nice straightforward
functional solution:

```rust
fn main() {
    let maxnum = 100;
    let sumsquares: u64 = (1..maxnum+1).map(|x| x*x).sum();
    let sum: u64 = (1..maxnum+1).sum();
    let squaresum = sum*sum;
    println!("{}", squaresum - sumsquares);
}
```

To me the rust `map` syntax looked a little strange at first but I'm starting to
get used to it. The `|foo|` syntax defines a *closure*. [Closures in
rust](https://doc.rust-lang.org/rust-by-example/fn/closures.html) are a bit like
python's `lambda` statement, it lets us name a temporary variable that is valid
inside the scope of the `map` function call. In this case it represents a single
value in the range iterator defined by `1..maxnum+1`. After mapping the numbers
to their square, we then sum the values. Since range iterators lazily generate
data, our solution never needs to hold all of the values of the iterator in
memory - cool!

## Generating prime numbers

Several of the Project Euler problems require generating large numbers of prime
numbers quickly. The brute force approach to this is called [trial
division](https://en.wikipedia.org/wiki/Trial_division). For a natural number
`$n$`, we check whether all natural numbers `$1 < m < n$` have the property `$n
\mod m \ne 0$`. There are some straightforward performance optizations one can
apply to speed this up. If `$n$` is divisible by 2 then it can't be prime, so we
reject it. If it's not divisible by 2 then it can't be divisible by any larger
even number either, so we can skip all even numbers - a 2x performance
boost. We can get an even bigger performance improvement for large `$n$` by
realizing that one need not check any integer bigger than `$\sqrt{n}$`, since if
`$n$` is divisible by a number bigger than `$\sqrt{n}$` it must also be
divisible by a number smaller than `$\sqrt{n}$`, which we would have already
checked. We need to go all the way up to `$\sqrt{n}$` in case `$n$` is a perfect
square.

Another approach is to use the [Sieve of
Eratosthenes](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes), an ancient
algorithm for generating all primes up to a limit. Start with a list of all the
natural numbers from two to $n$ and then for each number, find all the multiples
of $n$ in the list and mark them as not prime. Once all the numbers in the list
up to `$n/2$` have been tested, all the numbers that have not been discarded
must be prime. For Problem #7 I ended up coding this up like so:

```rust
static MAX: usize = 1_000_000;

fn sieve(n: usize) -> usize {
    let data: Vec<usize> = (0..MAX).collect();
    let mut isprime: Vec<bool> = vec![true; data.len()];
    isprime[0] = false;
    for (i, d) in data.iter().enumerate() {
        let offset = *d;
        if !isprime[i] || offset == 1 {
            continue
        }
        for j in ((i+offset)..isprime.len()).step_by(offset) {
            isprime[j as usize] = false
        }
    }
    let primes = isprime.iter().enumerate().fold(vec![], |mut acc, (index, value)| {
        if *value {
            acc.push(index)
        }
        acc
    });
    primes[n]
}
```

This approach uses a `Vec<usize>` called `data` to contain the prime numbers and
a `Vec<bool>` to contain the list of numbers that are prime in `data`. We make
use of `enumerate` in two places to generate an indexed iterator over a list of
numbers, a pattern I like to use from Python where one array indexes another
array. It's nice how the approaches I like to use in Python tend to be adaptable
in Rust despite the languages being very different.

It turns out that Problem 10 also needs a large number of prime numbers, so we
can reuse this function with a small adaptation there.

## The `num` crate and the `BigInt` type

Many of the problems in Project Euler generate integers that cannot be
represented using the integer data types built into rust. In Python, integers
transparently get promoted from machine integers to arbitrary-precision integers
(see [PEP 237](https://www.python.org/dev/peps/pep-0237/) for more details about
python integers) transparently, so we don't need to worry about the
distinction. Rust is a lower-level language and wants you to carefully consider
the memory cost of an operation, so it won't automatically convert integers when
they overflow. However, in debug mode, [integer overflow does trigger a
panic](https://github.com/nox/rust-rfcs/blob/master/text/0560-integer-overflow.md)!
I find this delightful compared with my experiences in other low-level languages
like C or C++:

{{< tweet 1134102247149887488 >}}

In this way we know immediately when the builtin integer types are no longer
sufficient and we need to switch to an arbitrary-precision integer arithmetic
library. 

Currently, the most popular library for this sort of thing on
[crates.io](https://crates.io) is the [`num`](https://crates.io/crates/num)
crate, in particular the `num::bigint::BigInt` type. Rather than coercing
integers automatically to be instances of `BigInt` by overriding the arithmetic
operators as a Python library probably would do, the approach here is to force
users to specify which numbers exactly are `BigInt` instances by explicitly
converting.

I used `num` for 5 of the first 20 problems, in problems 8, 13, 15, 16,
and 20. The usage from Problem 20, which asks to compute the sum of the digits
of `$100!$`, is probably the most readable:

```rust
use num::bigint::{BigInt, ToBigInt};
use num::traits::{One, Zero};

fn main() {
    let mut num = 100.to_bigint().unwrap();
    let mut ret: BigInt = One::one();
    let zero: BigInt = Zero::zero();

    while num != zero {
        ret *= &num;
        num -= 1;
    }

    let digits = format!("{}", ret)
        .chars()
        .map(|c| c.to_digit(10).expect("not a digit!"))
        .collect::<Vec<u32>>();

    println!("{}", digits.iter().sum::<u32>());
}
```

First, we convert 100 to a `BigInt` using the `to_bigint` method that the
`ToBigInt` trait defines on rust's buitin integer types. Adding new features to
standard library types like this would be very strange in Python, but in rust
it's very natural. Since we explicitly bring the `ToBigInt` trait into scope
it's still explicit, however care must be taken to understand the side effects
of bringing traits into scope.

Next we define bigints to represent one and zero. The `num` package helpfully
provides traits to generate `BigInt` instances directly. From there the math
looks more or less identical to how we'd write it for a builtin type, however
with a `BigInt` this will not overflow. One difference is that the
multiplication step needs to borrow the data in the `BigInt` with the `&`
operator:

```rust
while num != zero {
    ret *= &num;
    num -= 1;
}
```

This borrow isn't necessary for machine integers because they implement the Copy
trait and can be cheaply copied. A `BigInt` is a wrapper for the byte buffer
that could be arbitrarily long and stores that data on the heap, so we need to
explicitly borrow or copy whenever we use the value.


Finally we format the bigint as a string to get a base 10 representation,
convert the digits of the string to integers, and sum the digits.

## Problem 11: Largest product of four numbers in a grid

In [this problem](https://projecteuler.net/problem=11) we are given a 20x20 grid
of natural numbers and asked to find the sequence of 4 neighboring numbers (in
any direction, including along diagonals) that has the largest product.

This problem lends itself to using a 2D array data structure to store the grid
of numbers. Because I like `NumPy`, I decided to use the
[ndarray](https://crates.io/crates/ndarray) crate, which provides a ND array
data structure that acts a lot like a NumPy array.

To generate the array, I first read the data into a Vec containing Vecs of `u64`
entries:

```rust
let rows = contents.split("\n").collect::<Vec<&str>>();

let rows = rows
    .iter()
    .map(|r| {
        r.split(" ")
            .filter_map(|d| d.parse::<u64>().ok())
            .collect::<Vec<u64>>()
    })
.collect::<Vec<Vec<u64>>>();
```

A tricky bit here is my usage of
[`ok()`](https://doc.rust-lang.org/std/result/enum.Result.html#method.ok) to
convert a `Result` into an `Option`, which allows me to use `filter_map` and
avoid using `unwrap`. Since I know *a priori* that the input table doesn't have
blanks this is mostly a semantic dance and I could probably get away with using
`unwrap`, but it's nice to know of ways to generate code that provably cannot
panic and crash.

Next I generate 2D 20x20 array and read the data into it:

```rust
let nrows = rows.len();
let ncols = rows[0].len();

let mut array = ndarray::Array::zeros((nrows, ncols));

for i in 0..nrows {
    for j in 0..ncols {
        array[(i, j)] = rows[i][j];
    }
}
```

Now to actually check sequences, I defined a wrapper type for a 4-tuple and gave
it a `prod` method to calculate the product of the numbers in the sequence:

```rust
struct Sequence((u64, u64, u64, u64));

impl Sequence {
    fn prod(&self) -> u64 {
        (self.0).0 * (self.0).1 * (self.0).2 * (self.0).3
    }
}
```

And I wrote a `check` function that checks to make sure if the product of the
sequence is greater than a number:

```rust
fn check(s: Option<Sequence>, largeprod: &mut u64) {
    match s {
        Some(x) => {
            let product = x.prod();
            if product > *largeprod {
                *largeprod = product
            }
        }
        None => {}
    }
}
```

Note how this function accepts an `Option<Sequence>`. This lets us transparently
handle cases where there isn't a valid sequence and ignore them without
carefully designing our code to avoid testing places near the edge of the array
where we cannot generate a sequence 4 numbers long.

Finally we need code that generates sequences that go to the right, up, rising
diagonally, or falling diagonally from a starting location defined by the tuple
of indices `(i, j)`. Here's the code to generate horizontal sequences, the other
types are analogous:

```rust
fn horizontal_sequence(i: usize, j: usize, m: usize, arr: &Array<u64, Ix2>) -> Option<Sequence> {
    match (i, j) {
        (i, j) if i + 3 >= m || j + 3 >= m => None,
        _ => Some(Sequence((
            arr[(i, j)],
            arr[(i + 1, j)],
            arr[(i + 2, j)],
            arr[(i + 3, j)],
        ))),
    }
}
```

If the sequence goes off the edge of the array, we return `None`, otherwise we
return a `Sequence` to test.

Finally, we check all the possible sequences:

```rust
let mut largeprod = 0;

for i in 0..nrows {
    for j in 0..ncols {
        // horizontal sequences
        check(horizontal_sequence(i, j, nrows, &array), &mut largeprod);
        check(vertical_sequence(i, j, nrows, &array), &mut largeprod);
        check(
            rising_diagonal_sequence(i, j, nrows, &array),
            &mut largeprod,
        );
        check(
            falling_diagonal_sequence(i, j, nrows, &array),
            &mut largeprod,
        );
    }
}

println!("{}", largeprod);
```

The slightly unsightly formatting here is generated for me automatically by
`rustfmt`. Sometimes I disagree with it but it's better in my opinion not to
fight and just live with the standard automatic code formatting for the sake of
sanity.

This is an example of [dynamic
programming](https://ngoldbaum.github.io/posts/dynamic-programming/), where we
come up with a clever way of exhaustively but efficiently testing all possible
cases without repeating work.

## Problem 17: Converting Numbers to English

In [this problem](https://projecteuler.net/problem=17) we're asked to generate
English versions of all of the natural numbers up to 1000 and then count how
many non-hyphen and non-whitespace characters are in all of the numbers.

I found it most natural in rust to use a `match` statement:

```rust
fn englishify(i: usize) -> String {
    let ret = match i {
        1 => "one".to_owned(),
        2 => "two".to_owned(),
        3 => "three".to_owned(),
        4 => "four".to_owned(),
        5 => "five".to_owned(),
        6 => "six".to_owned(),
        7 => "seven".to_owned(),
        8 => "eight".to_owned(),
        9 => "nine".to_owned(),
        10 => "ten".to_owned(),
        11 => "eleven".to_owned(),
        12 => "twelve".to_owned(),
        13 => "thirteen".to_owned(),
        14 => "fourteen".to_owned(),
        15 => "fifteen".to_owned(),
        16 => "sixteen".to_owned(),
        17 => "seventeen".to_owned(),
        18 => "eighteen".to_owned(),
        19 => "nineteen".to_owned(),
        20 => "twenty".to_owned(),
        30 => "thirty".to_owned(),
        40 => "forty".to_owned(),
        50 => "fifty".to_owned(),
        60 => "sixty".to_owned(),
        70 => "seventy".to_owned(),
        80 => "eighty".to_owned(),
        90 => "ninety".to_owned(),
        1000 => "one thousand".to_owned(),
        _ => {
            let hundreds = i / 100;
            let tens = (i - hundreds * 100) / 10;
            let ones = i - hundreds * 100 - tens * 10;
            let mut ret: String = String::new();
            if hundreds > 0 {
                ret += &(englishify(hundreds) + " hundred");
            }
            if tens < 2 {
                let remainder = i - hundreds * 100;
                if remainder != 0 {
                    ret += &(" and ".to_owned() + &englishify(i - hundreds * 100));
                }
            } else {
                if hundreds > 0 {
                    ret += " and ";
                }
                ret += &englishify(tens * 10);
                if ones > 0 {
                    ret += &("-".to_owned() + &englishify(ones))
                }
            }
            ret
        }
    };
    ret
}
```

For all of the numbers whose English spellings cannot be generated
algorithmically, we have special cases in the match statement. For all other
cases, we generate the phrase algorithmically by considering the hundreds, tens,
and ones digits of the number and calling `englishify` recursively.

The usage of `to_owned()` in many of the match cases is a little ugly, but this
way it's clear that the return value of `englishify` owns the data for the
string. I think it might also be possible to store references to `'static
&[str]` instances and then build the result by dereferencing the references, but
that seemed more complicated than this approach, even if there's more copying
happening.

With this function, calculating the result is a straightforward functional
processing pipeline on the range of numbers between one and 1000:

```rust
fn main() {
    let numbers = (1..1001)
        .map(|n| englishify(n).replace(" ", "").replace("-", ""))
        .collect::<Vec<String>>();
    dbg!(&numbers);
    let nchars = numbers.iter().fold(0, |acc, x| acc + x.len());
    println!("{}", nchars);
}
```

Here we made use of the `fold` function, which allows us to define custom
accumulation logic, although we could have done the same thing with a `map(|x|
x.len()).sum()`.

## More Project Euler?

From here on I think I'm going to be a bit more sparing in my problem selection
and not try to exhaustively do them all. Hopefully I'll have some more blog
posts about Project Euler problems as I run into particularly interesting ones
or ones that teach me something new about rust.
