---
layout: post
title: Orchard Internals
categories:
- orchard
---

Lately I've been doing quite a bit of work with the [Orchard CMS][]. It's a powerful system and although it's quite new, becoming a 1.0 at the beginning of this year, there's a lot of code to understand if you want to start extending it and working with it in significant ways as a developer. 

While [the documentation][Orchard Documentation] is pretty good, I find that it helps me to really understand the internals of a system I'm using, especially when &mdash; like Orchard &mdash; it does quite a lot based on convention and "magic". So I'll be writing a series here which is a crawl around some of the Orchard internals examining how it works and which bits do what. It helps me to understand the system better, and hopefully it'll have some value for other developers that want to better understand Orchard and push the limits of what can be done with it.

I should note now that I'm not on the Orchard team &mdash; I didn't write any of this code, so if I misunderstand it, apologies in advance. I'll happily make corrections where errors or inclarity are pointed out.

Posts will be listed below as they're added...

1. [The Orchard Startup Process][Orchard Startup]
2. [The Orchard Host][Orchard Host]
3. Orchard Shell Activation
4. Request Lifecycle in Orchard
5. Orchard Extensions Mechanics
6. next...

Note that if something says it's "in progress", that pretty much means I'm writing it in public and I haven't finished yet. Sometimes it takes me a few days to write things, other times I rattle through in an hour, but I quite like just getting something up even if it's not finished. If something stops weirdly half way through, come back and check whether I'm writing as you're reading &mdash; I'll finish off soon, just bear with me...

If you've got any suggestions for areas you'd like me to cover, corrections, thoughts, etc. just drop me an email (see the Contact page) or say hello on Twitter, etc.

[Orchard CMS]: http://www.orchardproject.net
[Orchard Documentation]: http://www.orchardproject.net/docs/
[Orchard Startup]: /orchard/2011/08/30/orchard-startup-process.html
[Orchard Host]: /orchard/2011/09/01/orchard-host.html
