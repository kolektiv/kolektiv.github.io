---
layout: post
title: Aether &mdash; Update
categories: fsharp aether
---

Just before the weekend, I pushed out 4.0 of [Aether][aether] (yes, that does seem a high version number, but that's what happens with SemVer!) It's a good update I hope, as it makes one aspect much better -- isomorphisms.

Previously, Isomorphisms were simply treated as another kind of lens, and composed as such. However, this didn't really work for cases where you had chains of isomorphisms composed with a partial lens. Even if all of the isomorphisms would succeed, you'd never be able to set a value if the existing partial lens returned `None`. This is because composition worked "left to right", so the chain of composition would halt when a partial lens returned `None` (setting a lens involves getting the value "left" of the lens to set to, as a rough intuition).

The new model works better by making isomorphisms a first class concept -- indeed they have their own type signature (and it's very obvious):

{% highlight fsharp linenos=table %}
type Iso<'a,'b> = ('a -> 'b) * ('b -> 'a)
{% endhighlight %}

Very simply -- the pair of functions which map `a` to `b` and back. Partial isomorphisms are represented the same way, with the first function returning `'b option` rather than `'b`. Isomorphisms now also have their own composition operators for composing an isomorphism with a lens which solves the partial chain issue and they follow the exact same pattern as operators for composing lenses with lenses:

{% highlight fsharp linenos=table %}
<--> // Compose a total lens with a total isomorphism, giving a total lens
<-?> // Compose a total lens with a partial isomorphism, giving a partial lens
<?-> // Compose a partial lens with a total isomorphism, giving a partial lens
<??> // Compose a partial lens with a partial isomorphism, giving a partial lens
{% endhighlight %}

Using these operators and signatures for defining and composing isomorphisms with lenses gives a much more effective and succinct way of dealing with isomorphisms in partial cases. The latest version of [Aether][aether] is available through [Nuget][aether-nuget] as always...

[aether]: https://github.com/xyncro/aether
[aether-nuget]: https://www.nuget.org/packages/Aether
