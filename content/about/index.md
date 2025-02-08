---
title: "About"
---

{{< image
src="/ngoldbaum2019.jpg"
alt="Nathan Goldbaum" >}}

I am a Senior Software Engineer at [Quansight Labs](https://labs.quansight.org),
where I am currently working on [community support for free-threaded
Python](https://py-free-threading.github.io). During the course of this project,
I led the effort to support [free-threaded
Python](https://docs.python.org/3/whatsnew/3.13.html#whatsnew313-free-threaded-cpython)
in [NumPy](https://numpy.org/) and helped add support for free-threaded Python
to foundational community Python projects accross the ecosystem. I also am the
author or editor of most of the documentation on
[py-free-threading.github.io](https://py-free-threading.github.io), which I hope
will be a lasting community resource.

After shipping [NumPy
2.1](https://numpy.org/devdocs/release/2.1.0-notes.html) with support for the
free-threaded build, I moved onto [PyO3](https://pyo3.rs) and led the effort to
ship support for building thread-safe Rust extensions for use with the
free-threaded build in [PyO3
0.23](https://github.com/PyO3/pyo3/releases/tag/v0.23.0). I have also been
fixing bugs in NumPy reported by the community. As a foundational library it's
important that NumPy be usable on the free-threaded build and I am excited to
already hear about preliminary use-cases that demonstrate the raw compute power
unlocked by the free-threaded build.

I will be giving a talk about my experience over the course of 2024 and 2025
with free-threaded Python at PyCon 2025. Stay tuned for more information about that!

Before working on free-threaded Python, I shipped support for [UTF-8
variable-length string
arrays](https://numpy.org/devdocs/user/basics.strings.html#variable-width-strings)
in [NumPy 2.0](https://numpy.org/devdocs/release/2.0.0-notes.html) and helped
lead the wider community effort to ship the first major release of NumPy in more
than a decade. I championed a UTF-8 string DType as the major new feature of
NumPy 2.0 and authored [NEP
55](https://numpy.org/neps/nep-0055-string_dtype.html) to argue that the
community should accept the new feature. This in-depth design process led to a
number of radical improvements to the initial design. I described my experience
going from very little low-level Python C API experience to becoming a NumPy
maintainer in a [talk at the Scipy 2024
conference](https://www.youtube.com/watch?v=cUhP0OCSWsk). The talk was
transcribed into a blog post on [the Quansight
blog](https://quansight.com/post/my-numpy-year-creating-a-dtype-for-the-next-generation-of-scientific-computing/).

I took time off from work on software From March 2020 to October 2022 to help
support my family and to recover from burnout.

After a stint at the [Recurse Center](https://recurse.com) in 2019, I took at
job at Quansight where I worked on [PyTorch](https://pytorch.org). At the time,
the Quansight PyTorch team helped the Meta PyTorch team to identify and fix bugs
reported by the community and to lead projects implementing features heavily
requested by the community. I fixed a number of bugs in PyTorch and shipped [the
`__torch_function__`
mechanism](https://pytorch.org/docs/stable/notes/extending.html#extending-torch-python-api)
for easily overriding and extending PyTorch in libraries that wrap PyTorch
tensors.

My time at the Recurse Center focused on improving my facility with the Rust
programming language. You can read the blog posts I wrote there on this blog
about my efforts to implement a Rust client for the
[Mercurial](https://www.mercurial-scm.org) distributed version control system.

My academic career ended as a a research scientist at the University of
Illinois' [National Center for Supercomputing
Applications](http://www.ncsa.illinois.edu/) in the [Data Exploration
Lab](https://data-exp-lab.github.io/).

At that time, I was a maintainer of The [`yt`](https://yt-project.org) Project,
a Python toolkit for working with 3D simulation data. As a member of the `yt`
steering committee, I helped bring `yt` under the umbrella of
[NumFOCUS](https://numfocus.org/project/yt). I also helped write, publish, and
maintain [`unyt`](https://github.com/yt-project/unyt/), a library for handling
data with physical units in Python. I published [a
paper](https://joss.theoj.org/papers/dbc27acb614dd33eb02b029ef20e7fe7) in the
Journal of Open Source Software describing the library, its origins, and
comparing performance and software engineering decisions with competing Python
libraries.

In the past I worked on [improving scaling and performance](https://ytep.readthedocs.io/en/master/YTEPs/YTEP-0032.html) when working with
gigabyte and terabyte-scale particle datasets in
`yt` and shared my
experience with the community in [a SciPy conference
talk](https://www.youtube.com/watch?v=pkZgQIGac6I) in 2017. I am also the
original author of the `PlotWindow` plotting interface that makes opinionated
styling decisions to make it possible to quickly generate publication-quality
visualizations of simulation data using an API based on what the data physically
represent rather than how the data are laid out on-disk. I described the
philosophy behind `PlotWindow` as a domain-specific visualization tool in a
[2016 talk at the PlotCon
conference](https://www.youtube.com/watch?v=Fd4TDoyQffw).

During my PhD I
[ran](https://ui.adsabs.harvard.edu/abs/2015ApJ...814..131G/abstract)
[simulations](https://ui.adsabs.harvard.edu/abs/2016ApJ...827...28G/abstract) of
Milky-Way like disk galaxies to understand how global properties of disk
galaxies influence the formation of stellar nurseries and how the energy
deposited by newborn massive stars feeds back to global disk scales, creating a
steady-state system much like our galaxy has experienced for billions of
years. The data and [analysis
software](https://bitbucket.org/ngoldbaum/galaxy_analysis/src/default/) and raw
simulation data used in my thesis are [publicly
available](https://girder.hub.yt/#collection/573647d3dd9119000164acf0). I
described the process of publicly releasing a large tranche of data in [a
talk](https://www.youtube.com/watch?v=zb0HBu3IhbU) at the Python in Astronomy
conference. I [also talked](https://youtu.be/nzr2vMQqiug?t=358) about exporting
simulation data into a Minecraft server.