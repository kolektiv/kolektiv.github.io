---
layout: post
title: Todo-Backend with Dyfrig &mdash; Part 3 &mdash; Defining and Routing to Resources
categories: fsharp dyfrig
---

This is the __third part__ of a multi-part introduction to building APIs with Dyfrig. Other parts are here:

{% include series/todo-backend.md %}

## Defining Resources

In the previous part we'd got to the stage of an API which was capable of responding to requests -- but it didn't really have much to respond with. The Todo Backend expects that we'll expose two resources over HTTP -- a list of Todo items at '/' (relative to some root prefix if needed) and an individual Todo item at a URL we supply when a Todo is created. So if we wanted to come up with a rough specification for that we'd say that we need two resources at paths like this:

{% highlight text linenos=table %}
/       - Todo collection resource
/:id    - Todo item resource
{% endhighlight %}

Happily, `Dyfrig.Machine` is designed to let us work just like this -- by modelling those resources as distinct items, and then using some other facility to route requests to them as required. (You may not be surprised to hear by now that there's a Dyfrig library just for that which we'll see soon).

## Dyfrig.Machine

`Dyfrig.Machine` is based on (and is a "spritual" port of) [Webmachine][webmachine] (an Erlang library for resource based HTTP programming) and also takes some ideas from [Liberator][liberator] (a Clojure library which is itself inspired by Webmachine).

The idea behind all three of these implementations is for the library to provide default logic for how to handle an HTTP request, in accordance with HTTP RFC [2616][rfc2616] (though this is now superseded by RFCs [7230][rfc7230], [7231][rfc7231], [7232][rfc7232], [7233][rfc7233], [7234][rfc7234] and [7235][rfc7235] which break the original RFC down and clarify some ambiguities). The user can then decide -- using some simple programmatic configuration mechanism -- which parts of the logic to allow and which parts to override/replace with logic/behaviour specific to the resource being implemented.

For example, a resource may override the function which by default will respond to a request which has resulted in a **"200 OK"** response, to make sure that some data is sent back, or it may override the configuration which says which HTTP Methods this resource allows.

The library will then take in to account this configuration and runtime and respond accordingly -- for example, by automatically sending the right kind of response if a request is made with an HTTP Method which isn't allowed. This saves the programmer from having to write a lot of boilerplate (or from having to think carefully about the minutiae of HTTP specifications) to correctly respond to HTTP requests.

## Defining our Resources with Dyfrig.Machine

In `Dyfrig.Machine`, the definition and configuration of a resource uses a computation expression based syntax to allow a way of configuring resources which is both (hopefully) readable and strongly typed. To define resources we therefore use the `machine` syntax like so:

{% highlight fsharp linenos=table %}
let todos =
    machine {
        return () } |> reifyMachine

let todo =
    machine {
        return () } |> reifyMachine
{% endhighlight %}

As we did with our simple `owin` monad before when we didn't actually know what was going to go in it, we just return `unit` from both of our `machine` monads for now. Note that we also pipe both of these to `reifyMachine` -- this function simply takes our monadic **definition** of what our resource is and turns it in to a function -- a `Dyfrig.Pipeline` function in fact so that we can easily compose it with other Dyfrig components.

## Routing to our Resources with Dyfrig.Router

We now have two resources, although they don't do much. We need to make it so that requests actually go to the right place. As mentioned early, we can use `Dyfrig.Router` to do this. Like `Dyfrig.Machine`, `Dyfrig.Router` uses a simple monadic syntax to register a route. Registering our two resources (as they're simply Pipeline functions, which is what `Dyfrig.Router` expects) looks like this:

{% highlight fsharp linenos=table %}
let api =
    routes {
        route Any "/" todos
        route Any "/:id" todo } |> reifyRoutes
{% endhighlight %}

We've replaced our old empty `api` function with this new one which uses `Dyfrig.Router`. As with `Dyfrig.Machine`, we pipe this to `reifyRoutes` to turn the **definition** of our routing in to a concrete Pipeline function. Each line in our `routes` monad here is simply saying "Register a route with Any HTTP method and this path, and use the following Pipeline function to respond to it (returning the result of the Pipeline function as the result of the routing function)". If the Router can't match the request to a registered route, it'll return `Next` and so allow something later in a pipeline to handle the request.

## Next

Now that we have two resources and routing to ensure that requests are routed to the correct resource, it's time to start filling in the logic of what they should do. In the next part we'll start [Implementing Resources]({% post_url 2014-09-24-todo-backend-part-4 %}).

<!-- Libraries -->

[webmachine]: https://github.com/basho/webmachine/wiki
[liberator]: http://clojure-liberator.github.io/liberator/

<!-- Specs -->

[rfc2616]: http://www.ietf.org/rfc/rfc2616.txt
[rfc7230]: http://tools.ietf.org/html/rfc7230
[rfc7231]: http://tools.ietf.org/html/rfc7231
[rfc7232]: http://tools.ietf.org/html/rfc7232
[rfc7233]: http://tools.ietf.org/html/rfc7233
[rfc7234]: http://tools.ietf.org/html/rfc7234
[rfc7235]: http://tools.ietf.org/html/rfc7235
