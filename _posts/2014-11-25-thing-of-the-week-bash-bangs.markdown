---
layout: post
title: "Bash Bangs"
date: 2014-11-25 17:40
comments: true
categories:
- totw
---

This week I wanted to do a quick tip on some neat functionality in bash (and other bash-like shells), the bang commands.

Let's say you have some long important command that you want to run.  You run it, only to discover that you need sudo
privileges.  For situations like this, `!!` (entire last command) can be a great time-saver.

```sh
$ echo 1 2 3 4 5
1 2 3 4 5 # let's pretend echo throws an error too and wants sudo for some reason

$ sudo !!
sudo echo 1 2 3 4 5
Password:
1 2 3 4 5
```

Now, let's say you don't want to blindly re-run the last command.  `:p` can be used to print it without overwriting the
"last command" history.

```sh
$ echo 1 2 3 4 5
1 2 3 4 5

$ !!:p
echo 1 2 3 4 5

$ sudo !!
sudo echo 1 2 3 4 5
Password:
1 2 3 4 5
```

Bang commands also extend to the individual parts of the last command you ran.  The most basic form of this is `echo
!:[n]`, where [n] is the nth word in the command, indexed from 0.  There are also shortcuts: `!^` gives the first arg
(like `!:1`) and `!$` gives the last arg.

```sh
$ echo a b c d e
a b c d e

$ echo !:1
echo a
a

$ echo a b c d e
a b c d e

$ !:5:p # :p works with any bangs!
e

$ echo !:5
echo e
e

$ echo a b c d e
a b c d e

$ echo !$
echo e
e
```

Personally, I use `!$` the most, since very often I'll only want the last arg (e.g., `ls [some tab-completed dir]` ->
`rm -r !$`).  Plus, it's the easiest sequence to hit, finger-wise.

One final tip: awesome shells like, say, `zsh`, will actually tab-complete bangs and get rid of the need for `:p`.
e.g. `sudo !!<TAB>` gets replaced with `sudo echo 1 2 3 4 5` in place.
