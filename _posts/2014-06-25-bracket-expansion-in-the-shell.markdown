---
layout: post
title: "Bracket Expansion in the Shell"
date: 2014-06-25 18:58
comments: true
categories:
- programming
---

Just a quick post to show of a neat little trick for those who are more command-line-driven: bracket expansion.

Basically, bracket expansion means that `some-string-called-{x,y}-here` desugars in the shell to
`some-string-called-x-here some-string-called-y-here`.  This is especially useful if, say, you're in a Java-like
directory structure and you accidentally placed your source class in your test folder, and you need to move it back:

```bash
# desugars into mv src/test/java/com/foobar/app/Class.java src/main/java/com/foobar/app/Class.java
mv src/{test,main}/java/com/foobar/app/Class.java
```

Or if you've already committed to source control, this also works quite nicely with `git mv`.  Another nice example from
recent memory is, say, if you were cleaning up some directories nested by date and wanted to only wipe a few months:

```bash
rm /posts/2014/{01,03,06,07}/*.html
```

Anyway, just a quick little post for a neat little trick.
