---
layout: post
title: ":label in IO.inspect"
date: 2020-12-04 12:00:00 -0400
comments: true
categories:
- elixir
---

`IO.inspect` is great for println debugging, and generally "does the right thing" with just about any input you throw at
it.  Up until recently, I would use it in a way that looked something like this:

```elixir
IO.puts("debugging pw validation (length)")
validated =
  credential
  |> cast(attrs, [:password])
  |> validate_required([:password])
  |> validate_length(:password, min: 15, max: 1000)
  |> IO.inspect()
  |> validate_complexity(:password)
  |> hash_password()
```

... which is mostly fine, but the extra `IO.puts` is still kind of bothersome -- especially if I have multiple points in
the pipeline that I'm inspecting.  But then I learned about the `:label` option:

```elixir
validated =
  credential
  |> cast(attrs, [:password])
  |> validate_required([:password])
  |> validate_length(:password, min: 15, max: 1000)
  |> IO.inspect(:label, "debugging pw validation (length)")
  |> validate_complexity(:password)
  |> hash_password()
```

`:label` saves an extra line and makes things way nicer for printing multiple steps in a pipeline.  Neat.  This kind of
thing seems petty, but it really does add up over time and make elixir a shockingly pleasant language to use.
