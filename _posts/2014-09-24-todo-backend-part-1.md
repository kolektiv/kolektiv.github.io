---
layout: post
title: Todo-Backend with Dyfrig &mdash; Part 1 &mdash; Getting Started
categories: fsharp dyfrig
---

This is the __first part__ of a multi-part introduction to building APIs with Dyfrig. Other parts are here:

{% include series/todo-backend.md %}

## Introduction

There's been some talk lately about what "the" story for doing Web Programming in F# is. The [guide][guide] over on the F# Foundation web site lists quite a few options, but picking one isn't always easy without being able to see some examples and compare styles, approaches and strengths/weaknesses.

The [Todo-Backend project][todo-backend] is a good approach to giving people comparable examples -- create an implementation which meets the spec and you can compare it with other ways of meeting the spec using different frameworks, languages, etc.

In this short series I'm going to build up (and extend) the Todo-Backend example using components from the [Dyfrig project][dyfrig-root], a set of libraries for building web things in an idiomatically functional way (or at least, **one** idiomatically functional way!). 

To make it easy to follow along and see the different stages of building up a project using Dyfrig components, as well as code inline I'll provide references to specific commits in the [Github repository][github-root] which is going to contain our example.

We'll use most of the components which currently exist in Dyfrig -- they're all designed to slot together, and you may find you use some or all of them, depending on your use case. Dyfrig works on top of the [OWIN][owin] web astraction which is widely supported across the .NET landscape, so we'll use the [Katana][katana] web server to self host our little project as a simple console application.

Our start point will be this simple server ([commit: xxx][github-1]):

{% highlight fsharp linenos=table %}
// in Api.fs

type TodoBackend () =
    member __.Configuration () =
        OwinAppFunc.fromOwinMonad (owin { return () })

// in Program.fs

[<EntryPoint>]
let main _ = 
    let _ = WebApp.Start<TodoBackend> ("http://localhost:8080")
    let _ = System.Console.ReadLine ()
    0
{% endhighlight %}

If you've used Katana before this should be familiar -- except __line 5__, which is our first glimpse of some Dyfrig specific code.

## Dyfrig.Core

__Line 5__ shows a very small example of the facilities `Dyfrig.Core` provides. First, a set of functions for transforming other types of functions in to OWIN compatible `AppFunc`s. Second, the `owin` computation expression (or monad) -- an AsyncState monad, where the state is an `OwinEnv` (a type alias defined in `Dyfrig.Core` for `IDictionary<string, obj>`, the basic OWIN data structure). This particular `owin` monad currently does nothing -- we've simply added it as a placeholder. We'll add a more interesting one soon.

## The owin monad

The `owin` monad may not seem all that useful at this point -- and indeed it's not something you'd usually expect to use in raw form. It's intended as a base on which to build more useful/specific abstractions. By defining `owin` monad functions which read and mutate the underlying state, we can build abstractions which fit a more functional style of programming. We have to use a mutation based form of programming, as OWIN requires it, but we can at least make it feel like a more natural functional approach to state/IO!

Other libraries within the Dyfrig set build richer abstractions on top of the `owin` monad, giving strongly typed access to HTTP request/reponse data, composition, routing, etc. all with strongly typed interfaces -- but built on a foundation which is a simple mutable dictionary.

## Next...

In the next part, we'll look at [Exposing our API]({% post_url 2014-09-24-todo-backend-part-2 %}) so that we can start to actually run the Todo-Backend tests.

<!-- General -->

[guide]: http://fsharp.org/guides/web/
[todo-backend]: http://todo-backend.thepete.net/
[owin]: http://owin.org/
[katana]: https://katanaproject.codeplex.com/

<!-- Dyfrig -->

[dyfrig-root]: https://github.com/fsprojects/dyfrig

<!-- Example -->

[github-root]: http://fill.in.later
[github-1]: http://fill.in.later

