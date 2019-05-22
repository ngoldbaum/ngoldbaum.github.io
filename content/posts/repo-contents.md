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
almost 200,000 *obsolecence markers*, this is a data format used by the evolve
extension, which we will return to in a future post. The next couple of messages
let us know which changes were added (since we're cloning, we've added all of
the changes in the repository, this message is more useful if we are only
pulling a subset of the changes), and lets us know that this repository has
something called a bookmark that is named `@` defined. We will talk more about
bookmarks later, but if you are familiar with git, `@` is a bit like the
`'master'` branch in that new clones will have a checkout of `@` in the working
directory of the repository. Finally, it creates the working directory, which
contains almost 2000 files. Note that this count is substantially less than the
3000 files that have ever been defined in the repository, some files that were
present in the past have since been removed.

OK, now that we've cloned the repostiory, let's take a look at what's inside:

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
$ hg init
$ echo "some data" > a
$ hg add a
$ hg commit -m "adding file a"
$ echo "some more data >> a
$ hg commit -m "adding some more stuff to file a"
```

And let's take a look at the contents of this new more trivial repository:

```

```