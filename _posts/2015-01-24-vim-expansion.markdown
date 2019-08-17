---
layout: post
title: "Vim % Expansion"
date: 2015-01-24 11:52:53 -0500
comments: true
categories:
- vim
- totw
---

### % (Current File Name)

Another vim tip this week!  This time, it’s about ‘%’, which expands to ‘current file name’.  This is especially useful
in projects with java/scala style directory setups, where your source is approximately 1.5 million folders away from the
project root, but you kind of want to hang around project root for things like ant/sbt/etc to work.  `%` makes this
easier to work with files in the deeply nested folders while doing this.

Taking a contrived example, instead of doing something like this to `git log` the file you are currently editing:

```
:!git log src/main/scala/com/bizo/(…)/Foo.scala
```

You can just do:

```
:!git log %
```

This is extremely convenient and works everywhere in command line mode (basically, whenever ‘:’ is used), but is also
useful to have if you’re ever writing vim script.  See `:h expand` for the function to use in vim script, and some other
special keywords.

But wait!  There’s more!

<!-- more -->

Vim also supports file modifiers.  For example, `:h` gives you the ‘head’ of the file name, i.e. the directory of the
file.  Taking another (contrived) example, you can git add the entire folder containing the file you are editing by
doing something like:

```
:!git add %:h
```

See `:h file-modifiers` for more details (and more modifiers).

### Another Convenient Expansion

I use `%:h` so often (for example, when I realize I’ve opened a file before creating the directory containing it, or am
editing a file in a directory that doesn’t exist) that I’ve made a shortcut for it in my vimrc:

```
cnoremap %% <C-R>=expand('%:h').'/'<CR>
```

Roughly speaking, it remaps the key chord `%%` in command line mode to paste from a special register that evals the vim
script inside it, which calls the expand() function.

Long story short, what this allows me to do is do something like:

```
:!mkdir -p %%
```

And the `%%` will expand in-place into whatever `%:h` resolved to.  Not only is this a win because it’s slightly less to
type than %:h, but the expansion also allows you to quickly modify your command on the fly and go up/down a directory if
needed.

And of course, here’s the requisite asciinema with a quick demo of this in action:

<script type="text/javascript" src="https://asciinema.org/a/14592.js" id="asciicast-14592" async></script>

Hope that’s useful / mildly interesting!
