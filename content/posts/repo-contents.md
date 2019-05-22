---
title: "What's in a repository?"
slug: repo-contents
description: null
date: 2019-05-21T16:08:55-04:00
type: posts
draft: false
categories:
- Technical
tags:
- Mercurial
---

When you look inside of a code repository you will likely first see just the
contents of a codebase at a particular revision. However, the repository also
contains a compressed version of the contents of every file at every version of
the codebase. For a mercurial repository, these compressed data are contained in
a hidden `.hg/` directory. In this blog post I'm going to try to figure out what
data are contained in this directory and how it's structured.

To follow along you will need to have mercurial installed. I'm going to use the
latest version, Mercurial 5.0, which you can install in a number of ways,
perhaps easiest using the `pip` associated with a Python 2.7
installation. Mercurial 5.0 has beta support for Python 3.5 or newer, so you can
use that as well if you do not have a Python 2.7 installation set up:

```
$ pip install mercurial --user
```

If you use `pip install --user` like I have here you will also need to ensure
that `$HOME/.local/bin` is in your PATH environment variable.

Let's take a look at the contents of the `.hg` directory for a real-world
repository. For this purpose let's use the repository for mercurial itself -
mercurial development is tracked using mercurial, naturally:

```
$ hg clone https://mercurial-scm.org/repo/hg
real URL is https://www.mercurial-scm.org/repo/hg
destination directory: hg
requesting all changes
adding changesets
adding manifests
adding file changes
added 42325 changesets with 80197 changes to 3352 files (+1 heads)
179808 new obsolescence markers
new changesets 9117c6561b0b:2338bdea4474
updating to bookmark @
1964 files updated, 0 files merged, 0 files removed, 0 files unresolved
```

Depending on how fast your internet connection is, this operation might take a
while to finish. Mercurial is telling us a lot of information here in its debug
output that might be helpful for understanding Mercurial's internals. First, it
tries to figure out if we've given it a URL or some other URI it can resolve. We
gave it an HTTPS URL so it just uses that to communicate with the mercurial
instance running on mercurial-scm.org. Second, it prints "Requesting all
changes" when it begins to pull the changes from the remote repository. This
happens in three steps, first obtaining the *changesets*, then the *manifests*,
and finally the *file changes*. Each of these correspond to different kinds of
*revlog* files on-disk that we will be looking at shortly. Briefly, the *revlog*
is the file format that mercurial uses to store versioned data, let it be
metadata, the manifest of files in a repository at any given time, and the
contents of each file at each revision in history.

After these steps, Mercurial lets us know how much data it has processed. For
this repository there are more than 40,000 commits to more than 3000 files over
the history of the repository. Next it tells us that this repository contains
almost 200,000 *obsolecence markers*, this is a data format used by the [evolve
extension](https://www.mercurial-scm.org/wiki/EvolveExtension), which is one of
Mercurial's coolest features but is also beyond the scope of this post, I will
try to return to it in the future. The next couple of messages let us know which
changes were added (since we're cloning, we've added all of the changes in the
repository, this message is more useful if we are only pulling a subset of the
changes), and lets us know that this repository has something called a bookmark
that is named `@` defined. We will talk more about bookmarks later, but if you
are familiar with git, `@` is a bit like the `'master'` branch in that new
clones will have a checkout of `@` in the working directory of the
repository. Finally, it creates the working directory, which contains almost
2000 files. Note that this count is substantially less than the 3000 files that
have ever been defined in the repository, some files that were present in the
past have since been removed.

OK, now that we've cloned the repository, let's take a look at what's inside:

```
$ cd hg/.hg
$ ls -lh
total 136K
-rw-rw-r-- 1 goldbaum goldbaum   57 May 21 16:26 00changelog.i
-rw-rw-r-- 1 goldbaum goldbaum   43 May 21 16:30 bookmarks
-rw-rw-r-- 1 goldbaum goldbaum    1 May 21 16:30 bookmarks.current
-rw-rw-r-- 1 goldbaum goldbaum    8 May 21 16:30 branch
drwxrwxr-x 2 goldbaum goldbaum 4.0K May 21 16:30 cache
-rw-rw-r-- 1 goldbaum goldbaum  88K May 21 16:38 dirstate
-rw-rw-r-- 1 goldbaum goldbaum  501 May 21 16:30 hgrc
-rw-rw-r-- 1 goldbaum goldbaum   59 May 21 16:26 requires
drwxrwxr-x 3 goldbaum goldbaum 4.0K May 21 16:30 store
-rw-rw-r-- 1 goldbaum goldbaum    0 May 21 16:27 undo.bookmarks
-rw-rw-r-- 1 goldbaum goldbaum    7 May 21 16:27 undo.branch
-rw-rw-r-- 1 goldbaum goldbaum   41 May 21 16:27 undo.desc
-rw-rw-r-- 1 goldbaum goldbaum   40 May 21 16:27 undo.dirstate
drwxrwxr-x 2 goldbaum goldbaum 4.0K May 21 16:35 wcache
```

Hmm, this is a lot of stuff. Let's make this a little simpler by starting with a
new repository with a single file and only a couple of commits:

```
$ cd ../../
$ mkdir test-repository
$ cd test-repository
$ hg init
$ echo "some data" > a_file
$ hg add a_file
$ hg commit -m "adding a_file"
$ echo "some more data >> a_file
$ hg commit -m "adding some more text to a_file"
```

This creates a repository containing a single file with two revisions:

```
$ hg log --graph
@  changeset:   1:0e80b49a8edc
|  tag:         tip
|  user:        Nathan Goldbaum <nathan12343@gmail.com>
|  date:        Wed May 22 09:29:18 2019 -0400
|  summary:     adding more text to a_file
|
o  changeset:   0:6f3346b94a1f
   user:        Nathan Goldbaum <nathan12343@gmail.com>
   date:        Wed May 22 09:28:44 2019 -0400
   summary:     adding a_file
```

Let's take a look at the contents of the `.hg` directory in this new more
trivial repository:

```
☿ ls -lh .hg
total 44K
-rw-rw-r-- 1 goldbaum goldbaum   57 May 22 09:27 00changelog.i
drwxrwxr-x 2 goldbaum goldbaum 4.0K May 22 09:29 cache
-rw-rw-r-- 1 goldbaum goldbaum   63 May 22 09:28 dirstate
-rw-rw-r-- 1 goldbaum goldbaum   26 May 22 09:29 last-message.txt
-rw-rw-r-- 1 goldbaum goldbaum   59 May 22 09:27 requires
drwxrwxr-x 3 goldbaum goldbaum 4.0K May 22 09:29 store
-rw-rw-r-- 2 goldbaum goldbaum   63 May 22 09:28 undo.backup.dirstate
-rw-rw-r-- 1 goldbaum goldbaum    0 May 22 09:29 undo.bookmarks
-rw-rw-r-- 1 goldbaum goldbaum    7 May 22 09:29 undo.branch
-rw-rw-r-- 1 goldbaum goldbaum    9 May 22 09:29 undo.desc
-rw-rw-r-- 2 goldbaum goldbaum   63 May 22 09:28 undo.dirstate
drwxrwxr-x 2 goldbaum goldbaum 4.0K May 22 09:29 wcache
```

Still a decent number of files but definitely less complex. There is a very
[helpful page](https://www.mercurial-scm.org/wiki/FileFormats) on the [mercurial
wiki](https://www.mercurial-scm.org/wiki/) that describes mercurial's custom
file formats, so we can look there to decide which of these files is important.

The first, `00changelog.i` is there to inform older versions of mercurial that
this repository was created with a newer version and is incompatible with the
old version. Mercurial development proceeds with [strict backward
compatibility](https://www.mercurial-scm.org/wiki/CompatibilityRules)
guarantees so repositories created by older versions of mercurial should
continue to work with newer versions forever, however there's guarantee that an
old mercurial client should be able to read a repository created by a new
one. Since mercurial is a distributed system it is important for it to be able
to talk to various versions of itself over the network or when operating on
repositories on disk.

The `cache` and `wcache` directories contain caches of various kinds used by
mercurial and some extensions:

```
$ ls -lh .hg/cache
total 1.4M
-rw-rw-r-- 1 goldbaum goldbaum  148 May 21 16:30 branch2-base
-rw-rw-r-- 1 goldbaum goldbaum  42K May 21 16:30 evoext-obscache-00
-rw-rw-r-- 1 goldbaum goldbaum 992K May 21 16:30 hgtagsfnodes1
-rw-rw-r-- 1 goldbaum goldbaum   14 May 21 16:30 rbc-names-v1
-rw-rw-r-- 1 goldbaum goldbaum 331K May 21 16:30 rbc-revs-v1
```

These arent documented on the wiki (last updated in 2013) and appear to contain
opaque binary data. I'm going to ignore these for now.

The `dirstate` file contains information about the state of the working
directory (e.g. everything in the repository *except* for the `.hg`
directory). Quote the mercurial wiki:

> This file contains information on the current state of the working directory
> in a binary format. It begins with two 20-byte hashes, for first and second
> parent, followed by an entry for each file. Each file entry is of the
> following form:

> <1-byte state><4-byte mode><4-byte size><4-byte mtime><4-byte name
> length><n-byte name>

> If the name contains a null character, it is split into two strings, with the
> second being the copy source for move and copy operations.

In addition there is a [wiki page](https://www.mercurial-scm.org/wiki/DirState)
devoted just to this file that contains more information.

Let's take a look at the contents of the `dirstate` file for our repository:

```
$ xxd .hg/dirstate
00000000: 0e80 b49a 8edc 08c2 d9ff cdcd 7fd7 1b55  ...............U
00000010: de9a 7f7f 0000 0000 0000 0000 0000 0000  ................
00000020: 0000 0000 0000 0000 6e00 0081 b400 0000  ........n.......
00000030: 195c e54e 9600 0000 0661 5f66 696c 65    .\.N.....a_file
```

If you're unfamiliar with hexadecimal output, I'm using the `xxd` tool to
quickly preview the binary content of the `dirstate` file. The first column
tells you how many bytes into the file we are. Each set of 4 hex characters
corresponds to two bytes in the file. If we look above to where we examined the
output of `hg log` for this repository, you can see that the first 20 bytes of
this file is the SHA1 [nodeid](https://www.mercurial-scm.org/wiki/Nodeid)
associated with the most recent change (`hg log` only shows the first 12 bytes
of the nodeid for brevity). The `nodeid` for a changeset is also sometimes
called a *changeset hash*. It is a cryptographically unique identifier for a
commit generated by hashing the commit contents along with some metadata for the
commit. The next 20 bytes is filled with zeros. This is a special nodeid called
the *nullid* that represents a nonexistent commit. These two commits are the
*parents of the working directory*, these are usually referred to as `p1` and
`p2`. In this case *p1* is the most recent commit, and since the last commit was
not a merge, *p2* is set to the nullid. In addition to being `p2` for non-merge
commits, an empty repository with no commits will have both `p1` and `p2` set to
the nullid. An interesting consequence of this choice is that completely
unrelated repositories can be merged with no issues, since ultimately all
repositories histories descend from the "commit" associated with the nullid.

Following the nodeid entries for the parents of the commit is the state entry
for the only file in this repository, `a_file`. This consists of a set of binary
encoded metadata for the file, first a one-byte "state", which for this file is
"n", corresponding to a "normal" state. Other options include "a" for added, "r"
for removed, and "m" for merged. Following this is 4 bytes containing the "mode"
of the file. This corresponds to the bytes `000081b4`. In this case the first
two bytes are null and the UNIX file permissions are encoded in the last two
bytes. In this case it corresponds to the octal permission code `664`

```
$ stat -c "%a %n" a_file
664 a_file
```

How this is calculated based on the contents of the `dirstate` file is a little
confusing to me, I'd like to come back to this later. Internally mercurial is
doing something like this python code:

```
>>> import os
>>> mode = '%3o' % (0x000081b4 & 0o777 & ~os.umask(0))
>>> mode
'664'
```

The first operation makes some sense, masking with `0o777` ignores the first two
and half bytes. The 8 may indicate that the next 12 bits correspond to three
octal characters, and then the next three characters are the file mask. I'm not
sure why we additionally need to mask with `~os.umask(0)`. Digging into the
history of mercurial, it looks like this extra masking step [was
added](https://www.mercurial-scm.org/repo/hg/rev/9ab2b3b730ee) to fix issues on
windows and wasn't in the original implementation, so let's just ignore it for
now.

The next 4 bytes contain the size of the file in bytes (in this case the entry
is `0x19`, or 25 bytes). As an aside, this makes me wonder what happens if you
add a file bigger than `0xFFFFFFFF` bytes! After this come 4 more bytes for the
modification time, in this case stored as the UNIX timestamp `0x5ce54e96`, about
9:30 AM EST on May 22 2019 when this blog post was being written. This will also
be not-great in 2038 when the UNIX epoch overflows a 32 bit integer. Next we
have 4 bytes for the length of the name of the file, in this case '0x6', or
plain old 6 to you and me, the number of characters in the filename. Finally the
filename itself, which is encoded in UTF-8, but in this case we can get away
with just reading off the ASCII in the hex dump.

Ok, that covers the `dirstate` file. There's still a few more files left, so
let's quickly go over those.

* `last-message.txt` 

  This file contains the content of the last commit message, presumably for
  caching purposes or so people can set up prompts that don't need to actually
  start up the mercurial executable.
  
* `requires`

  A record of repository requirements. This tells mercurial clients what
  features must be supported in order to work with the repository. Old clients
  that do not have support for newer features will refuse to load a repository
  that lists requirements from newer mercurial versions.
  
* `undo.*` files

  Files used by the deprecated "hg rollback" command to undo the last
  transaction. I will ignore these since they are only useful for a deprecated
  feature in mercurial.
  
Finally there is one last directory, the `store`:

```
$ ls .hg/store
00changelog.i  data     phaseroots  undo.backupfiles
00manifest.i   fncache  undo        undo.phaseroots
```

The primary purpose of this directory is to store the bulk of the repository
data, in the form of [revlog](https://www.mercurial-scm.org/wiki/Revlog)
files. This is a special data structure that was invented by mercurial's
original developer to store versioned data in a compressed manner. We will come
back to revlogs and the contents of this directory in the next blog post.
