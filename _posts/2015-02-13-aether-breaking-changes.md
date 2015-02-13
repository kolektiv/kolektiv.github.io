---
layout: post
title: Aether &mdash; Breaking Changes
categories: fsharp aether
---

Very quick note here! I just made some breaking changes to the [Aether][aether] API, changing functions like `getL` and `setPL` to more idiomatically F# formulations like `Lens.get` and `Lens.setPartial`. The new API should be if anything more obvious in terms of usage, and the slight increase in verbosity is probably compensated by the decreased "F# surprise".

I have, of course, bumped the library a major version, so assuming you're using a good dependency management tool which handles semantic versions correctly where specified (e.g. [Paket][paket]) you shouldn't have any immediate problems. However, it will be a mechanical refactor at some point.

My apologies if this causes some disruption to anyone, hopefully it should be quite minor, and makes F# a little more discoverable.

[aether]: https://github.com/xyncro/aether
[paket]: https://fsprojects.github.io/Paket/
