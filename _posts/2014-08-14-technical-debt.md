---
layout: post
title: Technical Debt &mdash; A Note
---

[Ward Cunningham][ward], in coining the analogy [Technical Debt][technical-debt] intended to convey a meaning &mdash; debt incurred for the purposes of a specific gain in some form, incurred consciously and knowingly. That meaning has become dangerously distorted:

## Technical Debt

A technical debt discussion between two surprisingly stilted characters with suspicous names might be something like this:

> -- <cite>Decision Maker</cite>
>
> We've got a choice here. We can build the full feature now, but miss this customer and their lucrative contract, or we can build a good cut-down version of the feature now -- it'll do what the customer needs.

> -- <cite>Implementer</cite>
>
> If we build a cut-down version, we'll hit the deadline, but then it's going to take us more time in the future whenever we would have needed the full feature. We'll have to go back and build that at some point, and it'll be more work in the long run -- we'll have to throw away at least part of the cut-down version.

> -- <cite>Decision Maker</cite>
>
> Ok, we'll do that. We'll build the cut-down version for now and take the technical debt. It's worth it, this customer is worth too much to ignore -- we know it isn't the final thing and we know it'll cost us to live with for a while, but we'll take that choice.

## Technical Bankruptcy

A discussion ostensibly about technical debt, but really about a dangerous attitude to codebase quality might instead sound something like this. 

> -- <cite>Decision Maker</cite>
>
> We need to get this out quickly. How long's it going to take?

> -- <cite>Implementer</cite>
>
> A couple of weeks if we do it properly. If I just hack it for now and copy over some of the old code I can probably mangle the old system in to doing roughly that until we get a chance to do it properly -- it'll work and we can fix it later. Technical debt right?

> -- <cite>Decision Maker</cite>
>
> Ummmm. [I Do Not Think It Means What You Think It Means][pb]

## Debt?

The first of these discussions is akin to borrowing money from a bank at a nice fixed interest rate, knowing you can pay it back. The second is borrowing money from a huge scary guy called Vinnie who won't tell you what the interest rate is and who routinely carries a baseball bat with a nail through it. This isn't debt in any sensible way at all -- it's a disaster waiting to happen.

[ward]: http://c2.com/~ward/
[technical-debt]: http://en.wikipedia.org/wiki/Technical_debt
[pb]: http://knowyourmeme.com/memes/you-keep-using-that-word-i-do-not-think-it-means-what-you-think-it-means
