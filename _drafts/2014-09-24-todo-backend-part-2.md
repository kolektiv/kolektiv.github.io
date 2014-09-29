---
layout: post
title: Todo-Backend with Dyfrig &mdash; Part 2 &mdash; Exposing our API
categories: fsharp dyfrig
---

This is the __second part__ of a multi-part introduction to building APIs with Dyfrig. Other parts are here:

{% include series/todo-backend.md %}

## Quick Tests

The [Todo-Backend project][tb] gives us a few useful things, not least of which is a [quick test suite][tb-tests] that we can run online. It seems like a sensible first thing to do might be to run that, pointing at what we have so far. Maybe it'll pass already! (No of course not, but let's humour any TDD folk who happen to be reading...)

As expected, we get a bunch of errors. The first one seems like something we can fix fairly easily perhaps - it's about CORS... This isn't the right place to go in to a discussion of the details of CORS, but there is a good set of links on the [Todo-Backend project website][tb-cors]. For now, we can probably get away with simply adding the **"Access-Control-Allow-Origin"** and **"Access-Control-Allow-Headers"** header to anything we send back, with a value of "*" (i.e. we don't care who you are) and "Content-Type" respectively. This probably won't be enough later on when we come to deal with HTTP methods outside of the simple GET and POST, but this will get us somewhere.

Not only that, but it's a useful place to introduce another library in the Dyfrig family which can come in handy for this kind of job -- `Dyfrig.Http`.

## Dyfrig.Http

`Dyfrig.Http` is a set of abstractions which can be used on top of the `owin` monad. As mentioned previously, the `owin` monad isn't all that useful alone, but when you begin to build abstractions on top of it code can become much neater. It provides a few things -- strongly typed access to various common properties of the OWIN data object (`OwinEnv`) (using a [lens based approach][aether]) including many common Request/Response headers, and functions for common HTTP functionality (such as content negotiation, etc.).

We'll write something to add the required headers to any requests we handle. Here it is:

{% highlight fsharp linenos=table %}
let cors =
    owin {
        do! setPLM (Response.header "Access-Control-Allow-Origin") [| "*" |]
        do! setPLM (Response.header "Access-Control-Allow-Headers") [| "Content-Type" |] }
{% endhighlight %}

And that's it! Well, not quite as we're not actually using it yet. Let's break that function down first.

`setPLM` is an `owin` monad function. It takes a `PLens` (as defined in [Aether][aether]) and a value and applies it to the `OwinEnv` contained within the `owin` monad. `Response.header` is a function which takes a header name and returns a lens (all of the provided lenses are organised in a module structure under `Request` and `Response` modules). It's a general purpose lens, so doesn't attempt to map the value to a strongly typed value, but simply exposes the default OWIN approach to headers -- an array of strings.

When `cors` is called it will set the response headers we want! But... We're not calling it anywhere yet.

(**Aside:** `setPLM` is part of a small set of functions, equivalent to the `setL`, `setPL`, `getL`, etc. functions found in the [Aether][aether] library, but which work within an `owin` monad. They're provided as part of `Dyfrig.Http`.)

## Dyfrig.Pipeline

We already have a function in our `TodoBackend` class which is our `owin` monad. Although it doesn't do anything yet, we know we need to do more than just replace it with a function which sets a header! We need to run the cors function, and then our api function (empty as it is right now). We can do this using `Dyfrig.Pipeline`.

A Dyfrig pipeline is simply a chain of `owin` monad functions which return a `PipelineChoice` value. This is a simple union type, with values `Next` and `Halt`. Functions in the pipeline will be executed until one of the functions returns `Halt` at which point the rest of the pipeline will be skipped. A simple custom operator (`>?=` -- which can be thought of as "maybe bind") is used to form a pipeline from these functions.

Here's our `cors` function and our main `api` function (which we've pulled out of the `TodoBackend` class) as a pipeline:

{% highlight fsharp linenos=table %}
let cors =
    owin {
        do! setPLM (Response.header "Access-Control-Allow-Origin") [| "*" |]
        do! setPLM (Response.header "Access-Control-Allow-Headers") [| "Content-Type" |] }
    *> next

let api =
       owin { return () }
    *> next

let app =
        cors
    >?= api
{% endhighlight %}

Note that both of the functions now return `Next`, as they've been composed with the `next` function (a utility function which always returns Next in an `owin` monad) using the `*>` composition operator (which applies the two functions and returns the value of the second, discarding the value of the first). They are then of the correct type to be composed in a Dyfrig pipeline. Of course, if we hadn't wanted to continue processing the pipeline, we could have composed them with `halt`, rather than `next`.

Given that operator we can actually simplify a little further, removing the computation expression syntax entirely (`owin {`, `do! ...`, etc.) and make that `cors` function the following:

{% highlight fsharp linenos=table %}
let cors =
       setPLM (Response.header "Access-Control-Allow-Origin") [| "*" |]
    *> setPLM (Response.header "Access-Control-Allow-Headers") [| "Content-Type" |]
    *> next
{% endhighlight %}

Whether you prefer that is of course a matter of programming taste! I personally find it a little neater and more concise to be able to remove computation expression syntax in short functions -- and I don't really like long functions anyway as I only have a small brain.

At this point, we replace our original empty `owin` monad function in `TodoBackend` with `app`, and we should (and do) now have an API which passes the first CORS test! See the full code as it stands at [commit xxx][commit-2].

## Next

Now we have an "API" (a very empty one) which responds to requests from JavaScript, we'll start trying to implement the functionality we need to pass a few more tests. To do that, we'll look at how we can approach the requirements from the perspective of HTTP resources. `Dyfrig.Machine` will help us here -- it's designed to make exposing resources in a standards based way simple and predictable, with the minimum of complexity for your situation. 

Now we need to look at [Defining and Routing to Resources]({% post_url 2014-09-24-todo-backend-part-3 %})...

<!-- Todo Backend -->

[tb]: http://todo-backend.thepete.net
[tb-tests]: http://todo-backend.thepete.net/specs/index.html
[tb-cors]: http://todo-backend.thepete.net/implementing.html

<!-- General -->

[aether]: https://github.com/xyncro/aether
