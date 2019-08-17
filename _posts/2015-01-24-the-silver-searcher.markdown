---
layout: post
title: "The Silver Searcher"
date: 2015-01-24 11:53:21 -0500
comments: true
categories:
- totw
---

### grep

This week I’d like to talk about grep.  Grep is a great general-purpose tool and works very well for filtering text in
the middle of a long command chain, but I’ve found it a bit clunky as a codebase search tool.

For example, let’s say you’re sitting in some project and trying to grep for all the places where a function is being
called.  The naïve first attempt would be:

```
grep myFunc
```

Except that just hangs, since grep defaults to reading from stdin.  A next attempt might be:

```
grep myFunc .
```

Except then grep would complain that . is a directory, which leads to:

```
grep –r myFunc .
```

Which finally works, but still leaves a bit to be desired.  The biggest annoyance is that grep will get caught up in
files that you don’t necessarily care about, e.g. Tags files, third-party dependency files, binary files… “Binary file
./lib/default/xxx.jar matches” anyone?

### ag

Introducing [ag, the silver searcher](http://geoff.greer.fm/ag/)!  Ag:

- Fulfills the above “find this in cwd” use case via a simple, short `ag myFunc`
- Is easy to install and super fast
- Respects project ignore files: for example, it will ignore the patterns found in your .gitignore
- In the case of files that you want in the repo but still don’t want to search, it also supports the use of a .agignore file
- Integrates well into other tools: AFAIK there are ag plugins for vim, emacs, and text mate.

Fun fact: I was doing the whole `grep –r` thing for an embarrassingly long time before I bothered to search for a better
workflow.  My initial search turned up ack, which then led to ag.  As far as I can tell, feature-wise they’re
comparable; I eventually settled on ag just ‘cause the command requires less typing.
