---
title: "What this blog is for"
slug: my-first-post
description: null
date: 2019-05-20T17:15:33-04:00
type: posts
categories:
- Personal
tags:
- Recurse Center
- Learning
---

Hello from my second day at [the recurse center](https://recurse.com). I'll be
at RC for the next 12 weeks working on programming-focused projects, [pair
programming](https://www.recurse.com/manual#sec-pairing), job-hunting, preparing
for interviews, and exploring and enjoying Brooklyn and the rest of NYC. I'll be
using this blog to keep notes on the projects I'll be working on and to share
more generally with the rest of the world. Hopefully it will also keep me honest
and on-task.

Day one mostly consisted of orientation and getting-to-know-you time. In the
afternoon I spent time getting my new XPS 13 developer edition laptop set up and
customized they way I like for development. This is my first time using desktop
linux in about a decade and it's taking some getting used to, but thankfully the
XPS 13 seems to not have any annoying hardware issues or buggy drivers. I also
got the bare bones template for this blog set up using
[Hugo](https://gohugo.io/) and GitHub pages.

Today I'll be starting on the main project I'm planning to work on: a deep dive
into the [mercurial version control system](https://www.mercurial-scm.org/).

## Why Mercurial?

I used to use mercurial a bunch in the context of development for
[yt](https://yt-project.org) and [enzo](https://enzo-project.org)
development. In the past few years both projects switched from mercurial to
git. This reflects less on the quality of Mercurial as a software project and
more on the status of the ecosystem around Mercurial. Git has GitHub, GitLab,
and a plethora of other free tools and services that never took shape around
Mercurial. While Atlassian's Bitbucket does support Mercurial, it's clear that
development time is not currently focused on improving things for Mercurial
users. While recently the [sourcehut](https://sourcehut.org/) project added
mercurial support, the ecosystem around hg is still very barebones. This works
for large companies that use hg like Facebook because they can create a custom
environment and code-hosting solution that works for their systems, but open
source projects that make use of mercurial will have trouble attracting
contributors due to unfamiliarity with the tooling around the version control
system.

In my opinion, this state of things is a shame since Mercurial has a lot of
advantages over Git.

### A comprehensible user interface

By far my favorite feature of mercurial is its consistent and simple user
experience. One clear example is how `git checkout` does completely different
things in different contexts. If I want to switch to an existing branch, that's
`git checkout <branch>`. If I want to create a new branch, that's `git checkout
-b`. If I want to change the content of a file in my working directly to reflect
the content in the current branch, that's `git checkout <path>`. These are all
completely separate operations. With mercurial, these operations would be
`hg update <branch>`,`hg branch <branch>`, and `hg revert <path>`. Another good
exmaple of this is `git rebase`, which encompasses many different kinds of
history-editing operations that are handled in mercurial by several different
commands:

 * `rebase`
   handles the case of moving commits from one branch to another
 * `histedit`
   Interactively rewriting history (e.g. `git rebase -i` except with an awesome
   curses interface).
 * `fold`
   Collapse several commits into a single commit.
 * `split`
   Split a single commit into many commits.
 * `uncommit`
   Undoes a commit leaving the changes behind in the working directory.

Note that some of these commands are provided by the [evolve
extension](https://www.mercurial-scm.org/wiki/EvolveExtension) which enables
support for advanced history-editing workflows.

### A large, thoughtfully structured python codebase.
