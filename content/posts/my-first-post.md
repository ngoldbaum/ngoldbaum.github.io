---
title: "What this blog is for"
slug: my-first-post
summary: >
  What this blog is for.
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
into the [Mercurial version control system](https://www.mercurial-scm.org/).

## Why Mercurial?

I used to use Mercurial a bunch in the context of development for
[yt](https://yt-project.org) and [enzo](https://enzo-project.org)
development. In the past few years both projects switched from Mercurial to
git. This reflects less on the quality of Mercurial as a software project and
more on the status of the ecosystem around Mercurial. Git has GitHub, GitLab,
and a plethora of other free tools and services that never took shape around
Mercurial. While Atlassian's Bitbucket does support Mercurial, it's clear that
development time is not currently focused on improving things for Mercurial
users. While recently the [sourcehut](https://sourcehut.org/) project added
Mercurial support, the ecosystem around hg is still very barebones. This works
for large companies that use hg like Facebook because they can create a custom
environment and code-hosting solution that works for their systems, but open
source projects that make use of Mercurial will have trouble attracting
contributors due to unfamiliarity with the tooling around the version control
system.

In my opinion, this state of things is a shame since Mercurial has a lot of
advantages over Git. Just to take one example, by far my favorite feature of
Mercurial is its consistent and simple user experience. One clear example is how
`git checkout` does completely different things in different contexts. If I want
to switch to an existing branch, that's `git checkout <branch>`. If I want to
create a new branch, that's `git checkout -b`. If I want to change the content
of a file in my working directly to reflect the content in the current branch,
that's `git checkout <path>`. These are all completely separate operations. With
Mercurial, these operations would be `hg update <branch>`,`hg branch <branch>`,
and `hg revert <path>`. Another good exmaple of this is `git rebase`, which
encompasses many different kinds of history-editing operations that are handled
in Mercurial by several different commands:

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

Mercurial feels like a good fit for a deep dive at the recurse center for social
reasons as well.

### Lots of data structures and algorithms to explore

For each revision in the history of a codebase, Mercurial must be able to
efficient reconstruct the state of the code at that revision, store the changes
to the code made by that revision, and index it in a way that makes it easy to
search for information related to that commit in a number of different ways. To
be able to parse the data output by Mercurial I'll need to understand these
design constraints and reimplement some of the algorithms used by Mercurial on
my own. This will allow me to learn about [Merkle
trees](https://en.wikipedia.org/wiki/Merkle_tree), the [revlog data storage
scheme](https://www.mercurial-scm.org/wiki/Presentations?action=AttachFile&do=view&target=ols-mercurial-paper.pdf)
implemented by Mercurial, and algorithms for merging and diffing text.

### Friendly community

I've [participated in Mercurial
development](https://www.mercurial-scm.org/repo/hg/log?rev=author%28Goldbaum%29)
in the past and came away very happy with the experience. Development happens
via a mailing list or via [phabricator](https://www.phacility.com/phabricator/)
as well as on mailing lists and a relatively active [IRC channel on
freenode](https://www.mercurial-scm.org/wiki/IRC). In addition one of the
maintainers has agreed to answer questions from me in exchange for me sending in
patches to clarify and expand on the documentation for Mercurial's internals as
needed. Basically, I feel comfortable in my ability to access experts to get my
questions answered in a reasonable time frame as they come up.

### A large, thoughtfully structured python codebase.

Mercurial is one of the largest and oldest continuously developed open source
Python codebases in existence. This means that the quality of the code tends to
be high, enforced by rigorous line-by-line code review of every commit. That's
not to say there aren't dusty corners with high WTF factors, but hopefully the
code I'll need to read to understand how Mercurial works won't be too incredibly
nasty for me to ever have a hope of decoding it.

### An excuse to write a bunch of rust code

Finally, I'm hoping to use this project as an excuse to learn more about the
rust programming language. My goal is to write as much code as possible in
rust. I have no expectation that the code I'll be writing will be useful for
anyone, however there is a long-term plan to [rewrite portions of Mercurial in
rust](https://www.mercurial-scm.org/wiki/OxidationPlan), so it's possible that
some of the code I'll be writing will be useful for the project as whole. I'm
excited to learn about rust because the idea of never needing to worry about
memory corruption or data races while writing low-level code makes me feel
tingly inside.

## What's next and goals

I'll first start with a dive into the `.hg` directory of a real-world
repository. I'll investigate the file structure and try to understand the way
Mercurial stores its data on-disk. Next, I'll be writing parsers for some of the
data formats, and eventually hope to get to the point where I have a rust client
that can parse data from the repository. If I get to the point where I can do
`rust-hg log` on a real-world repository, I will be very happy.
