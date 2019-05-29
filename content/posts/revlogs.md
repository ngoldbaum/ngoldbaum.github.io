---
title: "Storing versioned data with revlogs"
slug: revlog
summary: >
  Revlogs are Mercurial's primary data storage format. In this post we will
  explore the structure of revlogs and write a rust parser for the revlog data
  format.
date: 2019-05-23T09:31:46-04:00
type: posts
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
the original author of Mercurial. There is also internal technical documentation
for the revlog data format included [in Mercurial's online
help](https://www.mercurial-scm.org/repo/hg/file/default/mercurial/help/internals/revlogs.txt),
accessible via `hg help internals.revlogs`.

## What is a revlog exactly?

A revlog - short for a revision log - is an append-only data structure for
storing discrete data entries that relate to other entries via a directed
acyclic graph (a
[DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph)). For Mercurial's
usage, the DAG in question is the graph of changes to the repository. Each entry
in a revlog consists of metadata and compressed revision data for the entry. The
metadata contains a cryptographic hash of the content of the revision, the size
of the content and metadata, and references to the *parent* entries for the
revlog. Each entry in a revlog can only have two parents. This is one of the
reasons that Mercurial does *not* allow [octopus
merges](http://www.freblogg.com/2016/12/git-octopus-merge.html), where revisions
can have an arbitrary number of parents. The revision data are stored in a
compressed format, either containing the full content of a file at a given
revision or as a delta relative to the state of the file at a previous
revision. Whether or not a revision contains a delta or the full content of a
file depends on how much data would be required to reconstruct the file
(e.g. the length of the already existing delta chain or the size of the change
in the revision). By storing occasional snapshots, Mercurial can reconstruct
repository content at any revision without going through all of the history of
the project and also without storing an unreasonable amount of data for each
revision. The style of delta chains with occasional snapshots is inspired by
video compression technology, where information about each frame is stored as
deltas to the previous frame, with occasional keyframes containing the entire
content of a frame of video.

## Kinds of revlogs

Mercurial stores all versioned data using three different kinds of revlog files,
*changelogs*, *manifestlogs*, and *filelogs*. Each of these have the same format, a
header followed by zlib-compressed content, but differ in the meaning of the
content. Changelogs store metadata about each commit, reading from this file is
sufficient to get most information displayed by `hg log`. Manifestlogs store the
manifest of the repository - a list of files contained in the repository at a
revision. Filelogs contain the revision information for individual files tracked
in a repository. Each of these revlogs are linked to each other. The changelog
contains a reference to a manifestlog entry, that manifestlog entry will in turn
contain references to filelog entries. By reading data from each of these revlog
files in turn, one can get the state of the files tracked by the repository at
each revision. This is how `hg update` works to update the state of the working
directory to a different revision.

## Content of a revlog file

Let's take a look at a revlog file from the test repository we created in [a
previous blog post](https://ngoldbaum.github.io/posts/repo-contents/). For now
we're going to be looking at the *changelog* file,
`.hg/store/00changelog.i`. This revlog file contains information about each
commit in the repository. Note that most revlogs used by Mercurial store the
content of individual files in the repository, but since revlogs are a generic
store for versioned data they can store versioned metadata like commit
descriptions and authors as well.

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

Hmm, no luck about any of the data in this file being human-readable - it
appears to be more or less random binary data that we're going to need to do a
bit more work to decode. We should perhaps not be surprised about this, since
revlogs store revision data in a compressed format - we'd need to be able to see
the uncompressed revision data to get human-readable text back.

According to the Mercurial documentation (in particular `hg help
internals.revlogs`, see
[here](https://www.mercurial-scm.org/repo/hg/help/internals.revlogs)), the first
four bytes of the revlog encodes the version of the revlog data format used by
the file as well as various feature flags. In this case the bytes are
`00010001`, which corresponds to a file that uses v1 of the revlog format and
sets a feature flag that says revision data are stored *inline* in this
file. This `v1` version of the revlog format is called RevlogNG - NG is short for
next generation, which made a lot of sense when it replaced Mercurial's original
revlog format in 2006. These days practically all repositories use one variant
or another of the `v1` RevlogNG format. Data for revlog entries can either be
stored interleaved with metadata entries or in a separate data file. Interleaved
data are used for relatively small revlogs while separate data files are stored
for large revlogs. This ensures revlog metadata can always be parsed without
also needing to scan through large amounts of revision data.

Following the header is the first entry. Each entry consists of metadata and
revision data:

* The number of bytes from the beginning of the file (6 bytes)
* Bit flags for special behavior. For now we will ignore these. (2 bytes)
* The length of the compressed revision data or delta stored in the entry. (4
  bytes)
* The length of the full revision data in uncompressed format, not the size
  of an uncompressed delta if the entry stores a delta. (4 bytes)
* Base revision to use when restoring the full text from a delta. If the base
  revision is the current revision, the revision stores the full text. (4
  bytes).
* A *linkrev*, the revision number of a revision that this entry is linked to. This
  allows a revision in one revlog to refer to a revision in a different
  revlog. For example, the revlog entry for a file points to the revlog entry in
  the `00changelog.i` changeset index file for the changeset that produced the
  revlog entry. (4 bytes)
* Signed integer revision of the first parent (`p1`), -1 indicates no parent. (4
  bytes)
* Signed integer revision of the second parent (`p2`), -1 indicates no
  parent. (4 bytes)
* Hash of the revision's content. (32 bytes).

Currently Mercurial uses a SHA-1 hash. Since SHA-1 hashes are 20 bytes long, the
remaining 12 bytes are set to zero. In the future repositories will use a
different, more secure hash function that can use up to 32 bytes to store
hashes. Since the offset to revision 0 will always be zero, it is elided. The
4-byte header takes the place of the offset for the first revision. The linkrev
and parent revisions are stored as integer revision numbers. The linkrev is the
revision number in the linked revlog file while the parent revisions are the
revision numbers in the current revlog.

For inline revlogs, the raw revision data follows the index entry, with no
header or padding in between. For non-inline revlogs, the data for the entry are
written at the offset specified in the first six bytes of the index entry. For
non-inline revlogs, the data are stored in a file with a `".d"` filename suffix
while the revlog index entries are stored in a `".i"` file. The revision entires
themselves consist of an optional one-byte header followed by the revision
data. The byte is either `\0`, for entries that are header-only and contain no
revision data, `u`, for uncompressed revision data, and `x`, for zlib compressed
revision data.

## Representing revlogs in rust

I've written a basic parser for this data format in Rust and [put it up on
hg.sr.ht](https://hg.sr.ht/~ngoldbaum/hg-rust/browse/466e0f40365e/revlogs/src/main.rs),
a free (for the time being) service for hosting Mercurial repositories. The data are stored
using two structs, one for the header:

```rust
struct RevlogHeader {
    offset: u64, // really 6 bytes but easier to represent as a u64
    bitflags: [u8; 2],
    compressedlength: u32,
    uncompressedlength: u32,
    baserev: i32,
    linkrev: i32,
    p1rev: i32,
    p2rev: i32,
    hash: [u8; 32],
}
```

And another for the content of the revlog entry itself, which includes a header
as well as uncompressed content of the revlog:

```rust
struct RevlogEntry {
    header: RevlogHeader,
    content: RevlogContent,
}
```

The content field is represented using an enum:

```rust
enum RevlogContent {
    Generic(String),
}
```

For now the enum only has a single variant, and is a trivial wrapper for a
String. The next step is to add special handling for changelogs,
manifestlogs, and filelogs, so this will eventually look like:

```rust
enum RevlogContent {
    Changelog(ChangelogEntry),
    Manigestlog(ManifestlogEntry),
    Filelog(FilelogEntry),
}
```

This will allow me to add special handling and pretty-printers or the different
kinds of revlogs used by Mercurial.

To actually read the changelog entries, we need to read the header, get the size
of the content of the revlog entry from the header, read the content, and
finally decompress the content. Since revlog headers are always 64 bytes, to get
the header we read 64 bytes of the changelog file and parse those destructure
the header content into the logical described above into the fields of the
`RevlogHeader` struct:

```rust
use byteorder::{BigEndian, ReadBytesExt};
use std::io;

impl RevlogHeader {
    fn new(buffer: &[u8; 64]) -> Result<RevlogHeader, io::Error> {
        let mut cursor = Cursor::new(buffer as &[u8]);
        Ok(RevlogHeader {
            offset: cursor.read_u48::<BigEndian>()? as u64,
            bitflags: cursor.read_u16::<BigEndian>()?.to_be_bytes(),
            compressedlength: cursor.read_u32::<BigEndian>()?,
            uncompressedlength: cursor.read_u32::<BigEndian>()?,
            baserev: cursor.read_i32::<BigEndian>()?,
            linkrev: cursor.read_i32::<BigEndian>()?,
            p1rev: cursor.read_i32::<BigEndian>()?,
            p2rev: cursor.read_i32::<BigEndian>()?,
            hash: {
                let mut res = [0u8; 32];
                cursor.read_exact(&mut res)?;
                res
            },
        })
    }
}
```

I made use of the [byteorder](https://docs.rs/byteorder/1.3.1/byteorder/), which
provides a nice interface for converting bytes of a binary stream into
big-endian or little-endian integers. We store the hash as a raw array of bytes,
but we can display it in a nice way like so:

```rust
header.hash.iter().map(|b| format!("{:02x}",b)).collect().join("");
```

This will convert the bytes into hexadecimal digits, collect those digits into a
vector of strings, and then joins the strings into a single hash digest. This
code is in the `fmt::Display` implementation for `RevlogHeader`.

Once we have the header, we read the content, and then decompress the content
using `zlib`. In practice we make use of [the flate2
crate](https://crates.io/crates/flate2), which provides an interface for
decompressing `zlib` streams:

```rust
use flate2::read::ZlibDecoder;

let mut gz = ZlibDecoder::new(&zlib_buffer[..]);
let mut decompressed_data = String::new();
let decompressed_data = gz.read_to_string(&mut decompressed_data)?;
```

With all of that, we're able to read the data in the changelog file for the test
repository we are working with:


```
$ cargo run
header:
offset: 0
bitflags: [0, 0]
compressed length: 111
uncompressed length: 119
base revision: 0
linked revision: 0
p1: -1
p2: -1
hash: 6f3346b94a1fbee70a8103708fd6d485edc88602000000000000000000000000

content: 8047a1de3d215413678766b615c006285e9b68df
Nathan Goldbaum <nathan12343@gmail.com>
1558531724 14400
a_file

adding a_file

header:
offset: 111
bitflags: [0, 0]
compressed length: 120
uncompressed length: 132
base revision: 1
linked revision: 1
p1: 0
p2: -1
hash: 0e80b49a8edc08c2d9ffcdcd7fd71b55de9a7f7f000000000000000000000000

content: d68909f0c439445f34273877884dde9eb55b20e4
Nathan Goldbaum <nathan12343@gmail.com>
1558531758 14400
a_file

adding more text to a_file
```

Here we can see that changelog entries store a hash that encodes a reference to
a manifestlog entry, the commit author and e-mail, a timestamp and timezone
offset, a list of files that are touched by the changeset, and the commit
description.

Next up I will be exploring the content of manifestlog and filelog files, with
the ultimate goal of being able to reconstruct snapshots of a repository at a
given revision, implementing the functionality of `hg update`.
