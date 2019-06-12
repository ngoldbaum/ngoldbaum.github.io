---
title: "The Rust Module System and Useful Crates for Command Line Applications"
slug: helpful-rust-cli-crates
summary : >
  Exploration of rust crates that are useful for writing command-line applications.
date: 2019-06-12T10:06:24-04:00
type: posts
categories:
- Technical
tags:
- Rust
- Mercurial
- Programming
---

Today I'll be continuing my
[series](https://ngoldbaum.github.io/posts/my-first-post/)
[of](https://ngoldbaum.github.io/posts/repo-contents/)
[posts](https://ngoldbaum.github.io/posts/revlog/) on the rust implementation of
the Mercurial version control system I've been working on. In this post I'll be
focusing on what I learned this week about the rust module system as well as a
few helpful crates I discovered to aid in command-line argument parsing and
error handling.

# What's in a Name?

Since my last post I've landed on a name for my project that's a bit nicer than
`hg-rust`. From now on this project will be known as `rug`. I've renamed the
repository on sr.ht and the code now lives at
https://hg.sr.ht/~ngoldbaum/rug. There should be redirects in place so the URLs
in my old posts will continue to work. I'd also like to come up with a
logo. Perhaps a rug with a crab on it that's playing with a droplet of mercury?
Probably not healthy for poor [Ferris](https://rustacean.net/)...

# The Rust Module System

As of my last post, [all of the
code](https://hg.sr.ht/~ngoldbaum/rug/browse/ae0220d9fb018d4890f3b7a8ab7585d49f74899c/src/main.rs)
lived in a single `main.rs` file that had grown to more than 200 lines of
code. Long modules like this can make it difficult to understand exactly how
everything interrelates. [Following the rust
book](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html)
I decided to break out the code in my project into submodules organized
according to the logical structure of the existing code.

First, I [moved the
code](https://hg.sr.ht/~ngoldbaum/rug/rev/0585ddebf295fc6ce623086dc4904671b579aaec)
that defines the various custom structs I wrote last week out of `main.rs` and
into a new `revlogs` module. At this point my `main.rs` file was much, much
simpler:

```rust
use std::env;
use std::fs::File;

mod revlogs;

fn main() -> std::io::Result<()> {
    let args: Vec<String> = env::args().collect();
    let fname = &args[1];

    let mut f = File::open(fname)?;

    let revlog = revlogs::Revlog::new(&mut f)?;

    println!("{}", revlog);

    Ok(())
}
```

Before this change all of the code that defined the `Revlog` struct lived above
the definition of the `main` function. Now that code has been replaced with a
single line: `mod revlogs`. This line tells the rust compiler that there is
either a file named `revlogs.rs` or a file named `revlogs/mod.rs`. The latter
allows splitting out a module even further into submodules. The other
modification to the `main` function is the way I'm creating the `Revlog`
instance. Rather than being able to use the `Revlog` name directly, I need to
refer to it as `revlogs::Revlog`.  I could have also said `use revlogs::Revlog`
above main to bring the `Revlog` struct into scope, but I prefer to avoid doing
that too much to make it clear where things are defined as I'm glancing at the
code.

I also needed to make the `Revlog` struct public, along with the `new` method I
implemented on it to create new Revlog instances from a file stream, so the
struct definition now looks like:

```rust
#[derive(Debug)]
pub struct Revlog {
    inline: bool,
    generaldelta: bool,
    entries: Vec<RevlogEntry>,
}
```

And the signature for the `new` method now begins with `pub fn new` instead of just `fn
new`. I haven't thought in detail about what *should* be public versus what the
compiler insists has to be public due to how I'm using these modules. I think
for a command-line application it doesn't matter so much what my public API is
because no one will be consuming it, but for a library it's probably
important. I will come back to these considerations later and see if I can
understand how to manage separation of concerns in rust in more detail.


Next I [further split
out](https://hg.sr.ht/~ngoldbaum/rug/rev/18cff5dfd47d45ed6167c4dd5a51f8e1b2d0e82f)
out the code for the `Revlog` struct into submodules for the `Entry`, `Content`,
and `Header` structs and [then
moved](https://hg.sr.ht/~ngoldbaum/rug/rev/6cbbe3b1683453eba4382b8ab04f5c043cd10e5e)
the `content` and `entry` modules to be submodules of the `entry` submodule. Now
everything is nice and modular, each module is relatively short, and the code is
structured according to the logical structure of the data structure the code
describes. Nice!

## Error Handling with the Snafu Crate

Error handling in rust is still something that confuses me. It's very different
from how error handling works in Python with exceptions. In rust functions that
might raise errors return an enum called `Result` that wraps either a valid
return value or an error. One problem I have with this is that the errors in the
rust standard library do not contain context (e.g. a backtrace) unless you
explicitly add a context to the error. Any context associated with the error
needs to be present at the location the error gets created, calling sites higher
up the call stack that might have more information that would be usable to
create a more helpful error message must consume the error and transform it into
a new error with the appropriate context, all completely manually. Finally,
rust's static type system means that errors of one type are not necessarily
convertible to errors of another type, so one must either explicitly convert
errors from one type to another or manually define the conversion methods to and
from a custom error type to other error types. This leads to a proliferation of
boilerplate code for each error type.

The rust error handling story is still somewhat in flux. For example, [RFC
2504](https://github.com/rust-lang/rfcs/blob/master/text/2504-fix-error.md)
describes an ongoing effort to reword the `Error` type in the standard
library). In online discussions people might suggest using the
[`error-chain`](https://github.com/rust-lang-nursery/error-chain) crate, the
[`failure`](https://github.com/rust-lang-nursery/failure) crate, or suggest just
using the standard library `Error` type and having lots of boilerplate in code
to handle conversions. As of early Summer 2019, the consensus seems to have
moved to the [`snafu`](https://github.com/shepmaster/snafu) crate. From my
perspective, one of the main advantages of `snafu` over `failure` is that
`snafu` has much better documentation that contains clear usage examples. That's
the main reason I chose to use it. A [recent reddit
thread](https://old.reddit.com/r/rust/comments/bubtu8/which_error_crate_are_going_to_use_in_2019/)
summarizes the state of things in 2019. I'm hoping that in the next year or two
this situation will grow more clear.

The philosophy behind the Snafu crate is to transform instances of errors
generated by standard library code or code outside of a developers control into
application-specific errors that are variants of a generic enum that represents
generic errors an application can produce. One defines an enum, in my case I
called it `RugError`, with variants that correspond to various kinds of errors:

```rust
#[derive(Debug, Snafu)]
enum RugError {
    #[snafu(display("rug must be run from inside a valid directory"))]
    NotAValidDirectory {
        backtrace: Backtrace,
        source: std::io::Error,
    },

    #[snafu(display("rug must be run from inside a repository"))]
    NotARepository { backtrace: Backtrace },

    #[snafu(display("The changelog file is not present in repository {}: {}",
                    path.display(), source))]
    NoChangelog {
        path: PathBuf,
        source: std::io::Error,
        backtrace: Backtrace,
    },

    #[snafu(display("The revlog file {} cannot be parsed: {}", path.display(), source))]
    CannotParseRevlog {
        path: PathBuf,
        source: std::io::Error,
        backtrace: Backtrace,
    },
}
```

I've told the compiler that my `RugError` enum derives from the `Snafu`
attribute. Each variant in the `RugError` enum is given a `snafu` attribute, which allows
me to customize the error message based on context-specific data. Together these
attributes generate all of the error-conversion boilerplate that I would
otherwise need to write myself to allow instances of my error type to be created
from standard library errors. Each error type can optionally define a `source`
and `backtrace` field. If `source` is defined, it maps to an error type. That
means that the corresponding variant must be created only from errors of the
corresponding type. If one tries to create an error from an incompatible error
type that will lead to a type mismatch and failed compilation. If `source` is
not provided, that means one is creating an error from the `None` variant of
some `Option`. If the `backtrace` field is defined, the error type generated by
snafu will contain a traceback such that when the error is printed out in a
`Debug` or `Display` representation, the backtrace will also be printed. This is
extremely helpful if it isn't clear where exactly an error of some type might be
generated in the code or if it isn't clear how a piece of code is ultimately
getting called by the application. Finally there can also be optional fields
that contain metadata one can use to construct a nice error message. For example
the `CannotParseRevlog` variant in my `RugError` enum contains a `path` field
that represents the path to the changelog file that cannot be parsed. The error
message generated by `CannotParseRevlog` uses both the `path` and the `source`
field to generate the error message.

To make use of these errors, the `snafu` crate provides the `ResultExt` and
`OptionExt` trait to extend the standard library `Result` and `Option` enums
with new methods that can transform errors at call sites. I made use of the
`context` method in a few places. For example, here is the function that
determines whether the current working directory is a mercurial repository:

```rust
fn hg_dir() -> Result<PathBuf, RugError> {
    let current_dir = env::current_dir().context(NotAValidDirectory)?;
    let mut anc = current_dir.ancestors();

    loop {
        let p = match anc.next() {
            Some(d) => d,
            None => break None,
        };
        let possible_hg_dir = p.join(".hg");
        if possible_hg_dir.is_dir() {
            break Some(possible_hg_dir);
        }
    }
    .context(NotARepository)
}
```

This function takes no arguments and returns a `Result` that can represent
either one of the custom errors I defined - a variant of the `RugError` enum, or
the path of the `.hg` directory in the root of the repository, represented by a
rust `PathBuf` object. The `loop` block returns an anonymous `Option` (e.g. it's
not bound to a variable name), that I call `context` on. I pass `context` the
`NotARepository` variant. The `context` function converts the `None` variant of
the `Option` into the `NotARepository` error. If the error ever bubbled back to
`main` it would get printed along with a backtrace because `NotARepository` has
a `backtrace` field. All of this happens automatically - this is the magic of
the `snafu` crate!

Side note - this uses a newish feature of rust - the `break` statement can
return values from inside a `loop` block. This feature was very handy here.
Without it I would have needed to create a function that did the loop and
explicitly returned an `Option`.

I can also call `context` on a `Result`. For example, here's the line where I
try to open the changelog file. If it isn't present, I create a custom error
that includes the path to the file that is supposed to exist:

```rust
let mut f = File::open(&fname).context(NoChangelog { path: &fname })?;
```

One downside of the `snafu` approach to error handling is that I need to be
careful to ensure standard library errors get converted into `RugError`
variants. In practice this means replacing usages of `?` with
`context(SomeError)?`, This can definitely be more verbose, however it also
forces me to think about the meaning of my code and what exactly each error
state really means. I'm hopeful that this will make debugging easier and lead to
fewer cases where I'm looking at opaque, poorly-described errors.

## Command Line Argument Parsing with `clap` and `structopt`

Of course it's possible to parse command line arguments fully manually by
consuming the iterator over arguments returned by the `std::env::args`
function. This works but requires a lot of wheel-reinventing to get common
behaviors like subcommands, positional arguments, optional arguments, and help
output to work properly. It makes sense to delegate that work to an external
library.

My first attempt at this used the `clap` library. In my usage of `clap` I
generated the command line arguments for the `rug log` subcommand like this:

```rust
use clap::{App, AppSettings, SubCommand};

fn main() -> Result<(), RugError> {
    let matches = App::new("rug")
        .version("0.1")
        .author("Nathan Goldbaum")
        .about("A rust implementation of some hg functionality")
        .setting(AppSettings::ArgRequiredElseHelp)
        .subcommand(SubCommand::with_name("log"))
        .get_matches();

    match matches.subcommand_name() {
        Some("log") => {
            hg_log()
        }
        _ => panic!("should be unreachable!"),
    }

    Ok(())
}

```

The name of the `App` corresponds to the name of the CLI binary. The `version`,
`author`, and `about` fields populate information in the help text for the
binary reported by `rug --help`. The `setting` usage tells `clap` to print the
help text if someone calls `rug` with no arguments. Finally the `subcommand`
creates a `log` subcommand that for now takes no arguments.

Finally to initiate the control flow for the program, I match over the name of
the subcommand that a user supplied and then do the work of running `rug log` if
someone passes in `log`. Note that the default branch is marked as unreachable,
that's because any other subcommand name will be caught and result in an error
message reported to the user at the command line. Here's a small command-line
session to see all of that in action:

```bash
$ rug
rug 0.1
Nathan Goldbaum
A rust implementation of some hg functionality

USAGE:
    rug <SUBCOMMAND>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

SUBCOMMANDS:
    help    Prints this message or the help of the given subcommand(s)
    log

$ rug notacommand
error: Found argument 'notacommand' which wasn't expected, or isn't valid in this context

USAGE:
    rug <SUBCOMMAND>

For more information try --help
```

I also get colored output in the error case to visually highlight the important
parts of the error message to the user - the colored output doesn't show up in
this post so don't worry that you can't see it here. I get all of this fancy
functionality more or less "for free" just by setting up `clap`. I like it!

One thing I don't like is that I'm matching over strings. In general `clap` will
return strings to me that represent the values of command line
options. That will work but will be brittle. I also won't be able to use the
ability of rust to check that I'm using all of the variants of an enum in a
match statement at compile time - so I might forget to implement a feature and
the compiler won't alert me about it.

This problem is solved by `structopt`, another crate that wraps `clap` and
allows one to define the command-line arguments and subcommands in terms of and
enums or structs. Here is the equivalent `structopt` code to my usage of `clap`
above:


```rust
#[derive(StructOpt)]
#[structopt(
    name = "rug",
    about = "A rust implementation of some hg functionality",
    author = "Nathan Goldbaum",
    version = "0.1",
    raw(setting = "structopt::clap::AppSettings::ArgRequiredElseHelp")
)]
enum Rug {
    #[structopt(name = "log")]
    Log {},
}

fn main() {
    match Rug::from_args() {
        Rug::Log {} => match hg_log() {
            Ok(_) => {}
            Err(e) => println!("{}", e),
        },
    }
}
```

We define an `enum` whose variants represent all of the different
subcommands. Each subcommand can then in turn define arguments that it
accepts. In `main` I instantiate an instance of the enum from the command-line
arguments and match over the result. Since the result will be variants of the
enum, I know that I've handled all possible subcommands, otherwise I would
generate a compiler error.

At this point I'm pretty happy with the state of things. The only thing that
bothers me about structopt (and generically with code that uses rust's
[attribute](https://doc.rust-lang.org/reference/attributes.html) system) is that
I'm programming inside of the attribute block, which feels a bit like writing
code inside of a string: outside of normal control flow. My editor doesn't
highlight this code like normal code. The whole thing feels very magical. That
said, I'm OK with the magic if it allows me to avoid a ton of boilerplate and
make my code more maintainable.
