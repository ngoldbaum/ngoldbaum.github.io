---
title: "Python vs Rust for Neural Networks"
slug: python-vs-rust-nn
summary: >
  Comparing a simple neural network in Rust and Python
date: 2019-07-26T14:17:19-04:00
type: posts
categories:
- Technical
tags:
- Rust
- Python
- Programming
---

In [a previous
post](https://ngoldbaum.github.io/posts/loading-mnist-data-in-rust/) I
introduced the MNIST dataset and the problem of classifying handwritten
digits. In this post I'll be using the code I wrote in that post to port a
simple neural network implementation to rust. My goal is to explore performance
and ergonomics for data science workflows in rust.

## The Python Implementation

[Chapter 1](http://neuralnetworksanddeeplearning.com/chap1.html) of the book
describes a very simple single-layer Neural Network that can classify
handwritten digits from the [MNIST dataset](http://yann.lecun.com/exdb/mnist/)
using a learning algorithm based on [stochastic gradient
descent](https://en.wikipedia.org/wiki/Stochastic_gradient_descent). This sounds
complicated --- and it kind of is, this stuff was state-of-the-art in the mid
1980s --- but really it all comes down to about [150 lines of heavily commented
Python
code](https://github.com/mnielsen/neural-networks-and-deep-learning/blob/master/src/network.py).

I'm going to assume that you already know the content of that chapter so stop
here and go read that if you want to brush up on neural network basics. Or don't
and just pay attention to the code, it's not super important to understand the
details of exactly why the code works the way it does to see the differences
between the Python approach and the Rust approach.

The fundamental data container in this code is a `Network` class that represents
a neural network with a user-controllable number of layers and number of neurons
per layer. The data for the `Network` class are represented internally as lists
of 2D NumPy arrays. Each layer of the network is represented as a 2D array of
weights and 1D array of biases, contained in attributes of the `Network` class
named `biases` and `weights`. These are both lists of 2D arrays. The biases
are column vectors but are still stored as 2D arrays by making use of a [dummy
dimension](https://stackoverflow.com/questions/17428621/python-differentiating-between-row-and-column-vectors).
The initializer for the `Network` class looks like this:

```python
class Network(object):

    def __init__(self, sizes):
        """The list ``sizes`` contains the number of neurons in the
        respective layers of the network.  For example, if the list
        was [2, 3, 1] then it would be a three-layer network, with the
        first layer containing 2 neurons, the second layer 3 neurons,
        and the third layer 1 neuron.  The biases and weights for the
        network are initialized randomly, using a Gaussian
        distribution with mean 0, and variance 1.  Note that the first
        layer is assumed to be an input layer, and by convention we
        won't set any biases for those neurons, since biases are only
        ever used in computing the outputs from later layers."""
        self.num_layers = len(sizes)
        self.sizes = sizes
        self.biases = [np.random.randn(y, 1) for y in sizes[1:]]
        self.weights = [np.random.randn(y, x)
                        for x, y in zip(sizes[:-1], sizes[1:])]

```

In this simple implementation the weights and biases are initialized by drawing
from the standard normal distribution --- a normal distribution with a mean of
zero, standard deviation of 1. We can also see how the biases are explicitly
initialized as column vectors.

The `Network` class exposes two methods that users would call directly. First,
the `evaluate` method, which asks the network to try to identify the digits in a
set of test images and then scores the result based on the *a priori* known
correct answer. Second, the `SGD` method runs a stochastic gradient descent
learning procedure by iterating over a set of images, breaking up the full set
of images into small mini-batches, updating the network's state based on each
mini-batch of images and a user-specifiable learning rate, `eta`, and then
re-running the training procedure for a new randomly selected set of
mini-batches for a user-specifiable number of *epochs*. The core of the
algorithm, where each mini-batch and the state of the neural network gets
updated, looks like this:

```python
def update_mini_batch(self, mini_batch, eta):
    """Update the network's weights and biases by applying
    gradient descent using backpropagation to a single mini batch.
    The ``mini_batch`` is a list of tuples ``(x, y)``, and ``eta``
    is the learning rate."""
    nabla_b = [np.zeros(b.shape) for b in self.biases]
    nabla_w = [np.zeros(w.shape) for w in self.weights]
    for x, y in mini_batch:
        delta_nabla_b, delta_nabla_w = self.backprop(x, y)
        nabla_b = [nb+dnb for nb, dnb in zip(nabla_b, delta_nabla_b)]
        nabla_w = [nw+dnw for nw, dnw in zip(nabla_w, delta_nabla_w)]
    self.weights = [w-(eta/len(mini_batch))*nw
                    for w, nw in zip(self.weights, nabla_w)]
    self.biases = [b-(eta/len(mini_batch))*nb
                   for b, nb in zip(self.biases, nabla_b)]
```

For each training image in the mini-batch, we accumulate estimates for the
gradient of the cost function via backpropagation (implemented in the `backprop`
function). Once we exhaust the mini-batch, we adjust the weights and biases
according to the estimated gradients. The update includes `len(mini_batch)` in
the denominator because we want the average gradient over all the estimates in
the mini-batch. We can also control how fast the weights and biases get updated
by adjusting the learning rate, `eta`, which globally modulates how big the
updates from each mini-batch can be.

The `backprop` function calculates the cost gradient for the neural network by
starting with the expected output of the network given the input image and then
working backward through the network to propagate the error in the network
through the layers. This requires a substantial amount of data munging, and its
where I spent most of my time porting the code to rust but I think it's a little
too long to dive into in depth here, take a look at [chapter
2](http://neuralnetworksanddeeplearning.com/chap2.html) of the book if you want
more detail.

## The Rust Implementation

The first step here was to figure out how to load the data. That ended up being
fiddly enough that I decided to break that off into its [own
post](https://ngoldbaum.github.io/posts/loading-mnist-data-in-rust/). With that
sorted I then had to figure out how to represent the Python `Network` class in
rust. I ended up deciding to use a struct:

```rust
use ndarray::Array2;

#[derive(Debug)]
struct Network {
    num_layers: usize,
    sizes: Vec<usize>,
    biases: Vec<Array2<f64>>,
    weights: Vec<Array2<f64>>,
}
```

The struct gets initialized with the number of neurons in each layer in much the
same way as the Python implementation:

```rust
use rand::distributions::StandardNormal;
use ndarray::{Array, Array2};
use ndarray_rand::RandomExt;

impl Network {
       fn new(sizes: &[usize]) -> Network {
        let num_layers = sizes.len();
        let mut biases: Vec<Array2<f64>> = Vec::new();
        let mut weights: Vec<Array2<f64>> = Vec::new();
        for i in 1..num_layers {
            biases.push(Array::random((sizes[i], 1), StandardNormal));
            weights.push(Array::random((sizes[i], sizes[i - 1]), StandardNormal));
        }
        Network {
            num_layers: num_layers,
            sizes: sizes.to_owned(),
            biases: biases,
            weights: weights,
        }
    } 
}
```

One difference is that in Python we used
[`numpy.random.randn`](https://docs.scipy.org/doc/numpy/reference/generated/numpy.random.randn.html)
to initialize the biases and weights while in rust we use the
`ndarray::Array::random` function which accepts a
`rand::distribution::Distribution`as a parameter, allowing the choice of an
arbitrary distribution.  In this case we used the
`rand::distributions::StandardNormal` distribution.  It's worth noting that this
uses an interface defined in three different crates, two of which --- `ndarray`
itself and `ndarray-rand` --- are maintained by the `ndarray` authors, and
another --- `rand` --- maintained by a different set of developers.

### The merits of monolithic packages

In principle it's nice that random number generation is not isolated inside the
`ndarray` codebase and if new random number distributions or capabilities are
added to `rand`, `ndarray` and all other crates in the rust ecosystem that need
random numbers can benefit equally. On the other hand it does add some cognitive
overhead to need to refer between the documentation for the various crates
instead of having a single centralized place to look. In my particular case I
also got a little unlucky and happened to do this project right after `rand`
made a release that changed its public API. This led to an incompatibility
between `ndarray-rand`, which depended on version 0.6 of `rand`, and my project which
declared a dependency on version 0.7.

I'd heard that `cargo` and rust's build system handle this sort of problem
really well but at least in this case I was presented with a confusing error
message about how the random number distribution I was passing in didn't satisfy
the `Distribution` trait. While this is true --- it satisfied the `Distribution`
trait from `rand 0.7` but not the one from `rand 0.6` that `ndarray-rand`
expected --- it is extremely confusing because the version numbers of the
various crates don't show up in the error message. I ended up reporting this as
[an issue](https://github.com/rust-ndarray/ndarray/issues/658). I discovered
there that these confusing error messages from crates with incompatible APIs is
[a long-standing issue](https://github.com/rust-lang/rust/issues/22750) for the
rust language. Hopefully in the future rust can grow more helpful error
messages.

In the end this separation of concerns caused a lot of friction for me as a new
user. In Python I could have simply done `import numpy` and be done. I do think
that NumPy probably went a bit too far in the direction of being completely
monolithic --- it was originally written at a time when packaging and
distributing Python code with C extensions was much harder than it is today ---
I do think that going too far in the other extreme can make a language or
ecosystem of tools harder to learn.

### Types and ownership

The next bit I'll show in detail is the rust version of `update_mini_batch`:

```rust
impl Network {
    fn update_mini_batch(
        &mut self,
        training_data: &[MnistImage],
        mini_batch_indices: &[usize],
        eta: f64,
    ) {
        let mut nabla_b: Vec<Array2<f64>> = zero_vec_like(&self.biases);
        let mut nabla_w: Vec<Array2<f64>> = zero_vec_like(&self.weights);
        for i in mini_batch_indices {
            let (delta_nabla_b, delta_nabla_w) = self.backprop(&training_data[*i]);
            for j in 0..nabla_b.len() {
                nabla_b[j] += &delta_nabla_b[j];
            }
            for j in 0..nabla_w.len() {
                nabla_w[j] += &delta_nabla_w[j];
            }
        }
        let nbatch = mini_batch_indices.len() as f64;
        for i in 0..self.weights.len() {
            self.weights[i] -= &nabla_w[i].mapv(|x| x * eta / nbatch);
        }
        for i in 0..self.biases.len() {
            self.biases[i] -= &nabla_b[i].mapv(|x| x * eta / nbatch);
        }
    }
}
```

The function makes use of two short helper functions I defined that makes this a little
less verbose:

```rust
fn to_tuple(inp: &[usize]) -> (usize, usize) {
    match inp {
        [a, b] => (*a, *b),
        _ => panic!(),
    }
}

fn zero_vec_like(inp: &[Array2<f64>]) -> Vec<Array2<f64>> {
    inp.iter()
        .map(|x| Array2::zeros(to_tuple(x.shape())))
        .collect()
}
```

Comparing with the Python implementation the interface for calling
`update_mini_batch` is a little different. Rather than passing in a list of
objects directly, instead of I pass in a reference to the full set of training
data and a slice of indices to consider within that full set. This ended up
being a little easier to reason about without triggering the borrow checker.

Creating `nabla_b` and `nabla_w` in `zero_vec_like` is very similar to the list
comprehension we used in Python. There is one wrinkle that caused me some
frustration which is that if I try to create a zero-filled array with
`Array2::zeros` and pass it a slice or `Vec` for the shape, I get back an `ArrayD`
instance. To get an `Array2` --- that is explicitly a 2D array and not a generic
D-dimensional array --- I need to pass a tuple to `Array::zeros`. However, since
`ndarray::shape` returns a slice, I need to convert the slice to a tuple
manually using the `to_tuple` function. This sort of thing can be glossed over
in Python but in rust the difference between a tuple and slice can be very
important, as in this API.

The code to estimate the updates for the weights and biases via backpropagation
has a very similar structure to the python implementation. We train each example
image in the mini-batch and obtain estimates for the gradient of the quadratic cost
as a function of the biases and weights:

```rust
let (delta_nabla_b, delta_nabla_w) = self.backprop(&training_data[*i]);
```

and then accumulate these estimates:

```rust
for (nb, dnb) in nabla_b.iter_mut().zip(delta_nabla_b.iter()) {
    *nb += dnb;
}
for (nw, dnw) in nabla_w.iter_mut().zip(delta_nabla_w.iter()) {
    *nw += dnw;
}
```

Once we've finished processing the mini-batch, we update the weights and biases,
modulated by the learning rate:

```rust
let nbatch = mini_batch_indices.len() as f64;
for (w, nw) in self.weights.iter_mut().zip(nabla_w.iter()) {
    *w -= &(nw.mapv(|x| x * eta / nbatch));
}
for (b, nb) in self.biases.iter_mut().zip(nabla_b.iter()) {
    *b -= &(nb.mapv(|x| x * eta / nbatch));
}
```

This example illustrates how the ergonomics of working with array data is very
different in Rust compared with Python. First, rather than multiplying the array
by the float `eta / nbatch`, we instead use `Array::mapv` and define a closure
in-line to map in a vectorized manner over the full array. This sort of thing
would not be very fast in Python because function calls are very slow. In rust it
doesn't make much difference. We also need to wrap the return value of `mapv` in
parens and borrow it with `&` when we subtract, lest we consume the array data
while we iterate over it. Needing to think carefully about whether functions
consume data or take references makes it much more conceptually demanding to
write code like this in Rust than in Python. On the other hand I do have much
higher confidence that my code is correct when it compiles. I'm not sure whether
the fact that this code was so demanding for me to write is due to Rust really
being harder to write or the disparity between my experience in Rust and Python.

## Rewrite it in rust and everything will be better

At this point I was left with something that was faster than the unoptimized
Python version I had started with. However, instead of a 10x or better speedup
that one might expect moving from a dynamic, interpreted language like Python to
a compiled performance-oriented language like rust, I only observed about a 2x
improvement. To understand why I decided to measure the performance of the rust
code. Luckily there is a very nice project that makes it easy to generate flame
graphs for rust projects:
[flamegraph](https://github.com/ferrous-systems/flamegraph). This adds a
`flamegraph` subcommand to `cargo`, so one needs only to do `cargo flamegraph`
in a crate, it will run the code, and then write a flamegraph `svg` file one can
inspect with a web browser.

{{< svg "flamegraph_rust.svg" >}}

If you've never looked at a flamegraph before the idea is that the proportion of
a program's runtime that occurs in a routine is proportional to the width of the
bar for that routine. The main function is at the bottom of the graph and
functions called by main are stacked on top. This gives you a simple view into
what functions take up the most time in a program - anything that is very "wide"
in the graph is where most of the time is spent and any stack of functions that
is very tall and wide is spending a lot of time in code very deep in a call
stack. Looking at the flamegraph above we can see that about half of the time is
spent in functions with names like `dgemm_kernel_HASWELL` --- these are
functions in the OpenBLAS linear alebra library. The rest of the time is spent
doing addition between arrays in `update_mini_batch and allocating arrays ---
all other parts of my program make a negligible contribution to the runtime.

If we made an analogous flamegraph for the python code, we would see a
similar pattern --- most time is spent doing linear algebra (in the places
where `np.dot` is called inside the backpropagation routine). So since most of
the time in either Rust or Python is spent inside a numerical linear algebra
library, we can never hope for a 10x speedup.

In fact it's worse than that. One of the exercises in the book is to rewrite the
Python code to use vectorized matrix multiplication. In this approach the
backpropagation for all of the samples in each mini-batch happens in a single
set of vectorized matrix multiplication operations. This requires the ability to
matrix multiplication between 3D and 2D arrays. Since each matrix multiplication
operation happens using a larger amount of data than the non-vectorized case,
OpenBLAS is able to more efficiently occupy CPU caches and registers, ultimately
better using the available CPU resources on my laptop. The rewritten Python
version ends up faster than the Rust version, again by a factor of two or so.

In principle it's possible to apply the same optimization to the Rust code,
however the `ndarray` crate does [not yet
support](https://github.com/rust-ndarray/ndarray/issues/16) matrix
multiplication for dimensionalities higher than 2. It might also be possible to use
thread parallelization on the mini-batch updates using a library like
[rayon](https://docs.rs/rayon/1.1.0/rayon/). I tried this on my laptop and did
not see any speedups but might have on a beefier machine with more CPU
threads. I could also have tried using a different low-level linear algebra
implementation, for example there are rust bindings [for
tensorflow](https://github.com/tensorflow/rust) and
[torch](https://github.com/LaurentMazare/tch-rs), however at that point I feel
like I might as well be using the Python bindings for those libraries.

## Is rust suitable for data science workflows?

Right now I have to say that the answer is "not yet". I'll definitely reach for
rust in the future when I need to write optimized low-level code with minimal
dependencies. However using it as a full replacement for python or C++ will
require a more stabilized and well-developed ecosystem of packages.



