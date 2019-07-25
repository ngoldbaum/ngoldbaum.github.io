---
title: "Loading MNIST Data in Rust"
slug: loading-mnist-data-in-rust
summary: >
  Loading the MNIST machine learning benchmark data in Rust
date: 2019-07-25T16:41:04-04:00
type: posts
categories:
- Technical
tags:
- Rust
- Machine Learning
- Programming
---

I've been spending a lot of time here at [the Recurse
Center](https://recurse.com) working on problems in the Rust programming
language. In a previous life I had spent a lot of time doing data intensive
numerical work in Python but so far I haven't tried to write similar code in
Rust. This week I've been going through [Michael
Nielson's](http://michaelnielsen.org/) very nice and easy-to-understand online
interactive textbook [Neural Networks and Deep
Learning](http://neuralnetworksanddeeplearning.com/) which comes out of the box
with a Python 2.7 implementations of several neural networks. The Python code
makes heavy use of NumPy and most mathematical operations make use of
vectorized computation.

The most advanced rust equivalent of NumPy is the
[ndarray](https://docs.rs/ndarray/0.12.1/ndarray/) crate. It even has a
[documentation
page](https://docs.rs/ndarray/0.12.1/ndarray/doc/ndarray_for_numpy_users/index.html)
specifically for users of NumPy. My hope was that porting the code should be
straightforward. It would also give me an opportunity to write some rust code
that handles the inherently mutable internal state of the neural network. I've
been told that this sort of thing can be tricky in Rust so let's find out
exactly how tricky it is.

The book is based around the problem of classifying handwritten digits. This
problem is a standard benchmark for machine learning algorithms and a lot of
work has gone into generating a standardized dataset people can use to train and
verify their neural networks. The python code I'm porting loads the data using
the [pickle protocol](https://docs.python.org/3/library/pickle.html) on pickle
files stored in the code repository. Loading pickle files in rust is not
something I want to dive into too deeply so instead I decided to use the
original MNIST datasets available [from the MNIST page on Yann LeCun's
website](http://yann.lecun.com/exdb/mnist/). These files are stored as `idx`
files --- a simple binary format that is fully described at the bottom of the
MNIST page. As previous readers of my blog know I have a little bit of
experience parsing binary formats with rust so this was relatively
straightforward.

The `idx` format stores binary array data. There is a magic number that encodes
the type of the data --- although all the files we are going to be working with
store data as unsigned 8 bit integers --- and the number of dimensions, followed
by the size of each dimension and then the data encoded in C order.

To represent the data as they exist on-disk I defined a struct named `MnistData`
that wraps a vector containing the dimensions of the data and then a `Vec<u8>`
that contains a flattened representation of the data:

```rust
#[derive(Debug)]
struct MnistData {
    sizes: Vec<i32>,
    data: Vec<u8>,
}
```

To actually read the data I created an initializer for the `MnistData`
struct that looks like this:

```rust
impl MnistData {
    fn new(f: &File) -> Result<MnistData, std::io::Error> {
        let mut gz = flate2::GzDecoder::new(f);
        let mut contents: Vec<u8> = Vec::new();
        gz.read_to_end(&mut contents)?;
        let mut r = Cursor::new(&contents);

        let magic_number = r.read_i32::<BigEndian>()?;

        let mut sizes: Vec<i32> = Vec::new();
        let mut data: Vec<u8> = Vec::new();

        match magic_number {
            2049 => {
                sizes.push(r.read_i32::<BigEndian>()?);
            }
            2051 => {
                sizes.push(r.read_i32::<BigEndian>()?);
                sizes.push(r.read_i32::<BigEndian>()?);
                sizes.push(r.read_i32::<BigEndian>()?);
            }
            _ => panic!(),
        }

        r.read_to_end(&mut data)?;

        Ok(MnistData { sizes, data })
    }
}
```

This makes use of the [`byteorder`](https://crates.io/crates/byteorder) crate,
which provides useful methods on rust's I/O types to interpret bytes as various
kinds of integers in big-endian and little-endian format. It also makes use of
the [flate2](https://crates.io/crates/flate2) crate to decompress the gzip
files.

To actually work with the data we will be converting the images into column
vectors - e.g. formally a 2D array with a shape of `(npixels, 1)`. We'll make
use of the [`ndarray`](https://docs.rs/ndarray/0.12.1/ndarray/) crate, which
provides a type that implements vectorized array computation and matrix
operations. We can also configure `ndarray` to use `OpenBLAS`, a C and FORTRAN
linear algebra library that provides highly optimized impelentations for various
matrix operations on a large variety of CPUs. If you want to use `ndarray` with
OpenBLAS, you need to explicitly turn that on in the `Cargo.toml` file:

```toml
[dependencies]
ndarray = { version = "0.12", features = ["blas"] }
blas-src = { version = "0.2.0", default-features = false, features = ["openblas"] }
openblas-src = { version = "0.6.0", default-features = false, features = ["cblas", "system"] }
```

If this works on your operating system and you have the relevant libraries
installed via e.g. your operating system's package manager this will
dramatically accelerate linear algebra operations.

Now to convert the data as loaded in directly from the `idx` file we need to do
a bit of data munging. I decided to create another struct, `MnistImage` that has
a copy of the image vector and the classification for the image:

```rust
#[derive(Debug)]
pub struct MnistImage {
    pub image: Array2<f64>,
    pub classification: u8,
}
```

With these definitions, processing the full dataset looks like this:

```rust
pub fn load_data(dataset_name: &str) -> Result<Vec<MnistImage>, std::io::Error> {
    let filename = format!("{}-labels-idx1-ubyte.gz", dataset_name);
    let label_data = &MnistData::new(&(File::open(filename))?)?;
    let filename = format!("{}-images-idx3-ubyte.gz", dataset_name);
    let images_data = &MnistData::new(&(File::open(filename))?)?;
    let mut images: Vec<Array2<f64>> = Vec::new();
    let image_shape = (images_data.sizes[1] * images_data.sizes[2]) as usize;

    for i in 0..images_data.sizes[0] as usize {
        let start = i * image_shape;
        let image_data = images_data.data[start..start + image_shape].to_vec();
        let image_data: Vec<f64> = image_data.into_iter().map(|x| x as f64 / 255.).collect();
        images.push(Array2::from_shape_vec((image_shape, 1), image_data).unwrap());
    }

    let classifications: Vec<u8> = label_data.data.clone();

    let mut ret: Vec<MnistImage> = Vec::new();

    for (image, classification) in images.into_iter().zip(classifications.into_iter()) {
        ret.push(MnistImage {
            image,
            classification,
        })
    }

    Ok(ret)
}
```

This function takes the name of the dataset --- for example for the `t10k`
training data this function should pass in the string `t10k` and returns a `Vec`
of `MnistImage` instances, one for each image in the dataset. To do this it
loads the classification and image files as an `MnistData` struct. Then we
figure out the dimensions of the image and thus ultimately what the shape of the
column vector we would like to store should be. Then we create a `Vec` from the
portion of the MNIST data corresponding to the image we want to extract, convert
the scale of the image from bytescale to floats ranging from 0 to 1 (we do this
because that's how the data are stored for the Python code), and then finally
create an image array to store the data. Once we've created all the image arrays
we iterate over the images and classifications, creating an `MnistImage`
instance to wrap each image array and classification, which we return.

All of this code lives in its own module so we only need to expose the
`MnistImage` struct and the `load_data` function to the rest of our code. I like
how easy it is to enforce separation of concerns in Rust, much easier than in
Python where separations of concern is more of a social convention.

In the next post I'll go into the process of porting the neural network code to
rust --- stay tuned!
