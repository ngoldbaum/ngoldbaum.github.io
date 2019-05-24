---
title: "Revlogs"
slug: revlog
summary: >
  Revlogs are Mercurial's primary data storage format. In this post we will
  explore the structure of revlogs and write a rust parser for the revlog data
  format.
date: 2019-05-23T09:31:46-04:00
type: posts
draft: true
categories:
- Technical
tags:
- Mercurial
- Rust
---

Mercurial makes use of the [revlog data
format](https://www.mercurial-scm.org/wiki/Revlog) for storing versioned data of
all kinds on-disk. The design constraints that led to the choice of this data
format are described [in a paper by Matt
Mackall](https://www.mercurial-scm.org/wiki/Presentations?action=AttachFile&do=view&target=ols-mercurial-paper.pdf),
the original author of Mercurial. There is also internal technical
documentation for the revlog data format included [in Mercurial's online
help](https://www.mercurial-scm.org/repo/hg/file/default/mercurial/help/internals/revlogs.txt),
accessible via `hg help internals.revlogs`

## What is a revlog exactly?

A revlog - short for a revision log - is an append-only data structure for
storing discrete data entries that relate to other entries via a directed
acyclic graph (a
[DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph)). For Mercurial's
usage, the DAG in question is the graph of changes to the repository. Each entry
in a revlog consists of metadata and revision data for the entry. The metadata
contains a hash of the content of the revision, the size of the content and
metadata, and references to the *parent* entries for the revlog. Each entry in a
revlog can only have two parents. Mercurial does
*not* allow [octopus
merges](http://www.freblogg.com/2016/12/git-octopus-merge.html), where revisions
can have an arbitrary number of parents.

Let's take a look at a revlog file from the test repository we created in the
last blog post. For now we're going to be looking at the *changelog* file. This
revlog contains information with metadata for each commit in the repository.

```
$ cd test-repository/.hg/store
$ xxd 00changelog.i
00000000: 0001 0001 0000 0000 0000 006f 0000 0077  ...........o...w
00000010: 0000 0000 0000 0000 ffff ffff ffff ffff  ................
00000020: 6f33 46b9 4a1f bee7 0a81 0370 8fd6 d485  o3F.J......p....
00000030: edc8 8602 0000 0000 0000 0000 0000 0000  ................
00000040: 789c 25c8 410e 0221 0c00 c03b afe0 05a6  x.%.A..!...;....
00000050: 85b6 6062 8c37 6f7e c194 2dbb 9200 7b59  ..`b.7o~..-...{Y
00000060: ff6f e21e 6732 5052 b41a 2d20 1346 4939  .o..g2PR..- .FI9
00000070: 8914 415e 0024 64ae d722 d956 f7d2 e3a3  ..A^.$d..".V....
00000080: d33f f76e 45bf c3df e63f 3044 8a8f 6d68  .?.nE....?0D..mh
00000090: eb97 651f 7787 cc99 23a6 401e 8900 9cbe  ..e.w...#.@.....
000000a0: d7d6 ab73 6ad6 e6e6 4ffe 00b4 ba22 1400  ...sj...O...."..
000000b0: 0000 0000 6f00 0000 0000 7800 0000 8400  ....o.....x.....
000000c0: 0000 0100 0000 0100 0000 00ff ffff ff0e  ................
000000d0: 80b4 9a8e dc08 c2d9 ffcd cd7f d71b 55de  ..............U.
000000e0: 9a7f 7f00 0000 0000 0000 0000 0000 0078  ...............x
000000f0: 9c25 c94b 0a02 310c 00d0 7d4f 9113 483f  .%.K..1...}O..H?
00000100: 894d 41c4 9d3b af20 eda4 1d0b fdc0 50c1  .MA..;. ......P.
00000110: e30b ba7d 4fce 1c74 287a 4317 10a9 38b4  ...}O..t(zC...8.
00000120: deb1 f7cc 2892 434e 44c9 ea8c ea11 d72b  ....(.CND......+
00000130: 0eb8 cf26 29be 3b5c c60f 8c75 e86e 7b8f  ...&).;\...u.n{.
00000140: b59d b6d9 afca 1031 39e3 89c1 206a ade2  .......19... j..
00000150: b3d4 9695 8a22 75ec d0e7 9161 e5cf 8235  ....."u....a...5
00000160: e17f 5fb0 0c27 1c                        .._..'.
```

Hmm, no luck about any of the data in here being human-readable - it appears to
be more or less random binary data that we're going to need to do a bit more
work to decode.

According to the Mercurial documentation (in particular `hg help
internals.revlogs`, see
[here](https://www.mercurial-scm.org/repo/hg/help/internals.revlogs)), the first
four bytes of the revlog encodes the version of the revlog data format used by
the file as well as various feature flags. In this case the bytes are `0001`,
which corresponds to a file that uses v1 of the revlog format and sets no
special feature flags. This version of the revlog format is called RevlogNG - NG
is short for next generation, which made a lot of sense when it replaced
Mercurial's original revlog format in 2006.

Following the header is the first entry. Each entry consists of metadata and
revision data:

* The number of bytes from the beginning of the file (6 bytes)
* Bit flags for special behavior. For now we will ignore these. (2 bytes)
* 

Since the offset to revision 0 will always be zero, it is elided. The 4-byte
header takes the place of the offset for the first revision.


