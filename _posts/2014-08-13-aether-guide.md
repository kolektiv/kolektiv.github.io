---
layout: post
title: Aether &mdash; Brief Guide
categories: fsharp aether
---

__NOTE:__ This article has been superceded by later changes to the Aether library in late 2015. See the [Aether site][aether-site] for up to date information.

__NOTE:__ This article was updated in June 2015 to indicate the use of the final syntax in Aether. Additionally, it is worth noting the changes around isomorphisms mentioned in the later [update][update].

A couple of days ago I released [Aether][aether] &mdash; introduced [here][aether-intro] &mdash; a little library for using Lenses in F#. [FSharpx][fsharpx-lens] already includes a lens implementation (and it's well written about by Mauricio Scheffer [here][bugsquash]), so "why bother?" is a reasonable question. Firstly, for libraries which wish to provide lenses over their own types, Aether does not require you to take a dependency on Aether &mdash; you can implement the types required natively. Secondly, Aether makes a distinction explicit between total and partial lenses.

## First &mdash; Lenses

Lenses are a useful tool for working with functional data structures. They allow us to get, set, and modify properties of immutable entities (in reality, return a new entity with new properties) deep within some nested structure of immutable types. For example, take the following very simple example of two record types, one nested within the other.

```ocaml
type Inner = { Value: string }
type Outer = { Inner: Inner }
```

To get the value of `Value`, if we have an instance of `Outer` is simple, but to set it we need to modify two records, like so:

```ocaml
let set v outer = { outer with Inner = { outer.Inner with Value = v } }
```

That's a bit awkward already, and we're only two levels of nesting in! If we get to a situation where we've got several complex nested types, perhaps where some fields are Maps or Lists of other complex types, or Options of types, the logic just to set or modify a value deep in the structure becomes unwieldy and error-prone.

Lenses can solve this problem neatly. A lens in essence is simply a pair of functions, a getter and a setter. The first, given some entity, retrieves a value from that object and the second, given some entity and a value, returns a new entity with the value set. A lens as defined in this way is essentially `('a -> 'b) * ('b -> 'a -> 'a)`. A lens for the `Value` field of the `Inner` type from earlier would look like this:

```ocaml
let valueLens = (fun x -> x.Value), (fun v x -> { x with Value = v })
```

It's a `Lens<Inner,string>` &mdash; a lens from `Inner` to `string`. In isolation this might not look very useful but as they're just functions, we can compose lenses together. If we have a lens from `Outer` to `Inner` and a lens from `Inner` to `string` we can compose them &mdash; and end up with a lens of `Outer` to `string`.

Now we can simply use a lens function (`Lens.set` in this case) to set `Value` like so (assuming we have a lens `Outer` to `Inner` named `innerLens`):

```ocaml
(* (>-->) is an operator which composes two lenses *)
let composedLens = innerLens >--> valueLens

(* or with partial application, let setValue = Lens.set composedLens *)
let setValue v outer = Lens.set composedLens v outer
```

If we provide lenses for commonly used aspects of our types, they can be composed in all kinds of new ways and we can save ourselves a lot of awkward/tedious/error-prone "longhand" manipulation of immutable values.

## Partial Lenses

The lenses we've seen so far have always been "reliable". We know that the getter function for `Inner` is going to return an instance of `Inner` because the type system won't let us - it's got to be there for us to have constructed an instance of `Outer`! But what about cases where a getter function might not be able to return something? Maybe the value is an `option` type. Maybe we're using a lens which refers to an item within a map at a certain key &mdash; and that key doesn't exist.

In this case, we need a partial lens. The lenses so far have been total, with the signature `('a -> 'b) * ('b -> 'a -> 'a)` but for a partial lens, we need to change the first function there to be `('a -> 'b option)`. These partial lenses can still be composed &mdash; and composed with total lenses. To do this, we introduce three more composition operators in addition to `>-->` we saw earlier.

```ocaml
(* composition operators *)

(>-->) // Total -> Total -> Total
(>?->) // Partial -> Total -> Partial
(>-?>) // Total -> Partial -> Partial
(>??>) // Partial -> Partial -> Partial
```

All except the `(>-->)` operator return a new partial lens. Let's change the earlier example slightly and see what this looks like:

```ocaml
type Inner = { Value: string }
type Outer = { Inner: Inner option }
```

`Inner` is now optional in our `Outer` type, so our new lenses look like this:

```ocaml
let innerLens = (fun x -> x.Inner), (fun i x -> { x with Inner = Some i })
let valueLens = (fun x -> x.Value), (fun v x -> { x with Value = v })
```

Our first lens now returns an `option` type for the getter &mdash; but note that according to our signature the setter still takes a non-`option` type, so we set the value to `Some i`. Now our composed lens looks like this:

```ocaml
(* we use the partial -> total composition operator now *)
let composedLens = innerLens >?-> valueLens

(* returns a string option *)
let getValue outer = Lens.getPartial composedLens outer

(* still takes a string, but it'll only be set if Inner is not None *)
let setValue v outer = Lens.setPartial composedLens v outer
```

We can see that we're using a couple of new functions here as well &mdash; `Lens.getPartial` and `Lens.setPartial`. They're simply the equivalents of `Lens.set` and `Lens.get` for partial lenses, and in general this is a convention that the Aether library sticks to. You will be unsurprised to learn that these functions are joined by `Lens.map` and `Lens.mapPartial` too, behaving much as you would expect.

So, that's a very brief intro to Aether, the next thing to do is to have a glance at the code if you're interested in using it. It's small and not very complicated and the useful functions are all quite apparent (hopefully). Head over to [here][aether] and take a look, and feel free to get in touch/raise issues/send pull requests if you feel like it!

[aether]: https://github.com/xyncro/aether
[aether-intro]: http://kolektiv.github.io/fsharp/aether/2014/08/10/aether/
[aether-site]: https://xyncro.tech/aether
[bugsquash]: http://bugsquash.blogspot.co.uk/2011/11/lenses-in-f.html
[fsharpx-lens]: https://github.com/fsprojects/fsharpx/blob/master/src/FSharpx.Core/Lens.fs
[update]: http://kolektiv.github.io/fsharp/aether/2014/09/29/aether-update/
