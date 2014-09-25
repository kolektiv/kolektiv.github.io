---
layout: post
title: Todo-Backend with Dyfrig &mdash; Part 4 &mdash; Implementing Resources
categories: fsharp dyfrig
---

This is the __fourth part__ of a multi-part introduction to building APIs with Dyfrig. Other parts are here:

{% include series/todo-backend.md %}

## Data and Logic

Although this isn't a walkthrough on designing logic driven systems, we are going to need an actual backend to handle the logic of storing Todo items, etc. We won't go in to the details of that here, but we've added one to the project in the `Storage.fs` file. Notice that this logic layer knows nothing about the web -- one of the nice things about building things in this way is that it's easy to maintain a separation between what's required to fulfil some logical requirement and what's needed to implement some specific interface -- in this case an HTTP interface.

We've provided enough functionality on our Todo storage system that we should be able to implement the requirements of the API, but we might need to tweak or add some functionality later to meet some details, or to extend the capabilities of the API past the basic Todo-Backend specification.

## The Todo Collection Resource

Let's start by seeing how many tests our Todo collection resource passes. It at least passed the first test before, so hopefully it still should... Oh! When we run it now, we find that it doesn't. Actually this shouldn't be too unexpected -- before the only handler was simply doing nothing, but now we're actually handling the requests using a Machine resource -- so we need to make sure it knows to allow the correct methods! At the moment, it's not allowing OPTIONS requests, and so they are (correctly) returning **"405 Method Not Allowed"**. Let's change that at least as we know we'll need to allow OPTIONS, and GET. (By default, the allowed methods are only GET and HEAD, so we'll need to override that default configuration. Now our resource looks like this:

{% highlight fsharp linenos=table %}
let todos =
    machine {
        allowedMethods [ GET; OPTIONS ] } |> reifyMachine
{% endhighlight %}

If we run the tests again, the first one passes! (It's a slightly worrying fact that it does as we're not really returning anything valid, but that's the way the tests are written...)
