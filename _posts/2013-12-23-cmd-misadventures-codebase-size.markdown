---
layout: post
title: "CMD Misadventures - Codebase Size"
date: 2013-12-23 06:50
comments: true
categories:
- cmd
- programming
---

After watching the [wat](https://www.destroyallsoftware.com/talks/wat) talk and trolling my friends with the
[aneditor](https://www.destroyallsoftware.com/talks/a-whole-new-world) talk for about the 200th time, I decided to
finally purchase one season of the Destroy All Software screencasts, despite the (IMHO) steep price tag and my financial
destitution.  (So far?  Totally worth it.  But a full review of the screencasts is neither here nor there.)

I've always been a big fan of the unix power tools--`find`, `grep`, `xargs`, and so forth--but the DAS talks introduced
an idea that had never occurred to me for some insane reason: combine them with git to extract some interesting
information about your codebase.  And so, I decided to go diving into my biggest scala project for insights about its
code size.

One of the most common problems that code size can indicate is the presence of "god classes" or libraries, which know
and do way too much and thus are correspondingly bigger than the rest of the code by orders of magnitude.  This command
was relatively simple and does not involve git, so here it is in its entirety:

```bash
alexkuang@Orion [00:00:00] [~foobar/src/main/scala] [master]
-> % find . -type f -name "*.scala" | while read file; do wc -l $file; done | sort -n
       9 ./com/foobar/models/Permission.scala
      11 ./com/foobar/util/LocParams.scala
      14 ./com/foobar/util/OrgSettings.scala
      16 ./com/foobar/security/package.scala
      17 ./com/foobar/scripts/ReloadStageDB.scala
      18 ./com/foobar/scripts/oneoff/InitSchema.scala
      # ...
     231 ./com/foobar/js/Calendar.scala
     247 ./com/foobar/persistence/Access.scala
     287 ./com/foobar/snippet/BookingCalendar.scala
     307 ./com/foobar/lib/Registration.scala
     319 ./com/foobar/lib/Scheduler.scala
```

The output was slightly interesting, but nothing groundbreaking.  300 lines is not ideal to me, but manageable.  Broken
down quickly, `find #...` finds all files inside the current directory ending in '.scala', reads each file in, and
passes it off to wc -l, which does a linecount on the file, whitespace and all.  `sort` does what its name implies, with
`-n` making it sort `1 2 3 11` instead of `1 11 2 3`.  The information was slightly cool, but as a hack it's not very
interesting, so let's throw some git in there to try to get a sense of how fast the codebase has grown over time.  After
all, superlinear growth is usually indicative of a ton of repetition and therefore unnecessary code complexity.

<!-- more -->

First, starting with walking the git repo.  `git rev-list <branch>` should do what we want it to, but in the case of
larger repos it the list can get a bit unwieldy/huge.  Enter `awk`, which lets you do a bunch of neat things with your
text but most importantly has an easy variable for line number, of all things _(note to self: learn2awk better?)_, thus:
`awk 'NR % <n> == 0'` to get only every nth revision list.  Combine that with the same reading as above, and do a
similar scala file find with a linecount, and the command is as follows: _(Yes, in this particular project I dev'd right
in master instead of using a nvie-style git-flow.  Bad developer, bad!)_

```bash
git rev-list master | awk 'NR % 20 == 0' | while read revhash; do git checkout -q $revhash | \
&& find . -name '*.scala' | xargs cat | wc -l; done
```

The more finicky among us might comment right about now that the command is already pretty huge and nigh unreadable if
revisited in about two weeks--and he'd be right.  But this is a quick one-off hack for some interesting info (something
that unix tools are absolutely amazing at), and if I cared that much I'd probably write a real script, or at least
re-format it into a proper bash function.

So the above command gives us a bunch of line counts which is useful, but it doesn't really give us a sense of the
progression.  At this point I'd usually either 1) compose some huge complicated thing that kept track of the current
line AND the previous in an attempt to do math, or 2) give up and write a real script for it later, but one of the DAS
videos showed something that was completely new to me: using `jot` to create a chart.  Even if I learned nothing else,
this alone made everything worth it.  Very quickly...

```bash
-> % jot - 1 5
# print range 1 to 5
1
2
3
4
5
-> % jot -b '*' - 1 5
# range 1 to 5, printing '*' instead
*
*
*
*
*
-> % jot -b '*' - 1 5 | xargs
# For all its magic, xargs just chunks up your input to be used as args.
* * * * *
-> % jot -b '*' - 1 5 | xargs | tr -d ' '
# And tr to translate.  Side note: as a recovering Perl user, it slightly annoys me that there's a tr util but not an s
# util.  But I guess that's what sed is for...?
*****
```

And now all that's left is to combine the `jot` magic with the above command by reading a the linecount into a variable
called `lines`, using that in the `jot` call, and printing everything out.  In the interest of full disclosure, here's
the final command along with the output from my project:

```bash
-> % git rev-list master | awk 'NR % 20 == 0' | while read revhash; do git checkout -q $revhash && \
find . -name '*.scala' | xargs cat | wc -l | \
read lines && ((hashes = $lines / 100)) && \
echo "`jot -b '#' - 1 $hashes | xargs | tr -d ' '` $lines"; done
######################################################### 5700
##################################################### 5333
################################################### 5151
############################################### 4796
############################################# 4530
##################################### 3786
################################### 3528
#################################### 3660
#################################### 3615
################################ 3208
############################ 2848
############################ 2832
############################ 2855
########################### 2786
######################## 2418
##################### 2186
################## 1834
################ 1664
############## 1412
############ 1270
########### 1179
######## 892
###### 651
#### 420
# 138
```

The growth at the beginning looked pretty normal, and I must say I'm slightly happy that around the middle it remained
constant, and even took a slight dip afterwards.  After the dip though it seems like the growth started shooting up
again, which is not a good sign.  This is consistent with my personal experience, as I recall starting to really throw
in the super-hacks at around that time, so everything is probably due for another refactor.

In closing, I'd like to remark that while this post was pretty monolithic and it took a lot of text to explain
everything for the first time, in real life this command probably took about 2-3 minutes to write.  And that's what I
find these utils are really really good at--Quick dirty answers to the little "I wonder..." / "What if..." questions
that tend to pop up while coding.
