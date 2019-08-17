---
layout: post
title: "Adventures with Mail.app Gremlins"
date: 2014-01-26 09:05
comments: true
categories:
---
#### Background

Usually I don't care too much if an email or two gets dropped en route.  Between spam filters and sometimes-spotty
(cough, mobile) connections, it's not too much of a stretch to assume that things get lost once in a great while.  But
recently I was trying to set up an interview on the other side of the country, so naturally I paid a little more
attention than usual.  Everything was going quite well, correspondence was zipping back and forth, and I'd even bought a
plane ticket for the trip.  But then a lull came when it was time to receive confirmation for hotel arrangements and a
rough schedule for the interview process.

Unbeknownst to me, the HR rep had already tried to send the information twice at this point, but for some
reason the emails weren't landing in my inbox.  I sent one last-ditch follow-up when my flight was boarding, and got a
reply with the info (thankfully) right before the plane took off.  Along with the reply, the HR rep mentioned that the
previous emails she'd sent had gone to `waffles@mochify.com`. _(aside: While `waffles@mochify.com` isn't the first email
I'd give out professionally, I'm thankful it wasn't something like `sexbadger69@gmail.com` ...  Actually, now I wonder
if that address is open ...)_

#### The technical details

This was an old address that I'd added to my various devices previously, but then removed for inactivity; I certainly
didn't recall sending any recent email from it.  But when I logged in to check the inbox, lo and behold the missing
emails were staring me in the face, along with a few others that had been "dropped" not too long ago.  Something smelled
fishy.  I checked over my Sent box for my regular email to no avail; all the correspondence was there, with the correct
`From`s and `To`s.  I ended up having to dig into the plain-text of the mime header to spot the issue:

```
From: Alex Kuang <[...]>
Content-Type: multipart/alternative;
    boundary="Apple-Mail=_7DA4001C-0AA1-48BD-80F5-00ACDBCCAE9C"
    Message-Id: <ADFD0F4C-6FE0-41F5-AA68-EF8E9845B360@gmail.com>
    Mime-Version: 1.0 (Mac OS X Mail 6.6 \(1510\))
    X-Smtp-Server: smtp.gmail.com:waffles@mochify.com
[...]
```

_X-Smtp? What?_  After a bit of googling I discovered that Mail.app on the Mac keeps a list of outgoing smtp servers
associated with your mail accounts, which you can see in Preferences -> Accounts -> "Outgoing Mail Server" -> Edit SMTP
Server List.  The problem is, the entry with the association persists __even after an account is removed from the
list__: when I checked my smtp list, it included an entry for mochify as well as a few other one-off addresses that I'd
added and removed in similar fashion.  Most of my email (including `mochify.com`) is handled by google apps, which means
that the smtp server the entries pointed to were all `smtp.gmail.com`, and the only difference was the
username/authentication associated.

So what ended up happening here was that I'd sent the email from my regular account through Mail.app so it still carried
the correct `From`/etc.  However, for reasons unknown, the outgoing smtp entry for that account did not work at
that moment.  Since `mochify`'s smtp entry pointed at the same `smtp.gmail.com` server, I'm willing to bet that Mail.app
decided it was a perfectly good fallback, added the `X-Smtp-Server` MIME header, and sent the email causing this weird
reply-to behavior.

There is a checkbox in account preferences that will lock you into using one-and-only-one smtp server and prevent this
from happening, but honestly after this ordeal I will probably just be even more biased towards composing my email using
the web gmail ui.  I'm just glad that everything worked out in the end, and anyway this is a good reminder that I should
be more diligent in setting up auto-forwarding even for email addresses I don't plan on using.
