---
layout: post
title: "asciinema"
date: 2014-07-16 15:40
comments: true
categories:
- programming
---

I was tooling around the other day and discovered a service called [asciinema](https://asciinema.org) which provides
terminalcasts--essentially, "screencasts" with terminal I/O.  This is awesome for a few reasons.

From a content consumer's standpoint, it's often hard to follow a blog post that's trying to outline a command line tip,
or a vim tip, for the simple fact that the static nature of writing alone isn't optimal for showing the flow between
"steps" (commands, keystrokes, what have you) and "output" (what you're supposed to see after executing commands).
Having a dynamic format for demos helps with this greatly.  Sure, there are screencasts, but that involves dealing with
videos and their associated heavy Flash Player bullshit.  asciinema is rendered with just bits of html and js.

From a wannabe content producer's standpoint...  Writing is _hard_.  Writing while crafting appropriate examples is
harder.  Doing all that while struggling with capturing the nature of the examples in plain text?  No thanks.
Again--Yes, there is the option of screencasts, but those are painful to set up.  Screencasts means worrying about
things like capturing software, background audio, What Tab Do I Have Open In My Browser, and Will Video Compression
Screw My Text Legibility.  As a consummately lazy person who only wants to do short self-contained clips for now...
That's way too big of a barrier.  asciinema is easy--Just install, then `asciinema rec` from the terminal and `<CTRL-D>`
to exit and upload.

And embedding takes 2 seconds--Check it!

<script type="text/javascript" src="https://asciinema.org/a/10785.js" id="asciicast-10785" async></script>
