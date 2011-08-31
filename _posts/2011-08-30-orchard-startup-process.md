---
layout: post
title: Orchard Startup
categories:
- orchard
---

This is part 1 of a series on [Orchard Internals][]. Other posts will be linked from [Orchard Internals][] as they are published.

## Overview

Orchard does a lot when it starts up. Although it's an ASP.NET MVC 3 web application at heart, it replaces/augments/ignores various things as it needs to. This article will follow through the application startup process and see what gets run at this stage, right up until Orchard is ready to serve the first actual request.

## The Web Application

Everything starts off in the web application itself. If you've got the source code handy, this is the **Orchard.Web** project. Looking in here we see... not a lot. This is because Orchard only really uses this as a boot up point for the application proper. There's no content in here or traditional models, controllers and views &mdash; only the code needed to call out to Orchard and get things running.

If we look in the **global.asax.cs** file in the root of the Orchard.Web project, this is where we'll find our Orchard code. We don't even have some of the code which would normally be present in a default ASP.NET MVC 3 web app &mdash; there's no registration of Areas or Filters in here as Orchard deals with these itself. It doesn't map any Routes either, only ignoring the Route for resource.axd, as Orchard deals with Routes too.

The first place to look is in the application start method, which looks like this:

{% highlight csharp %}

protected void Application_Start() {
    RegisterRoutes(RouteTable.Routes);
    _starter = new Starter<IOrchardHost>(HostInitialization, HostBeginRequest, HostEndRequest);
    _starter.OnApplicationStart(this);
}

{% endhighlight %}

The call to `RegisterRoutes` is normal, but then we're straight in to Orchard code. We create a new `Starter<IOrchardHost>` and assign it to `_starter`, which is a static member of the application so it's always available.

This gets passed a few arguments on construction, each of which is a method to call at different application stages &mdash; on start, beginning a request, and ending a request. The one which really matters to us here is the method which is going to get called on application start &mdash; and which is actually invoked as part of calling `OnApplicationStart` on our instance of `Starter<IOrchardHost>`.

## Orchard.WarmupStarter

We've seen that we're creating a new `Starter<IOrchardHost>` here, so let's have a look at that. This class can be found in the **Orchard.WarmupStarter** project, which is the only Orchard project which is explicitly referenced in the web application - it's referenced in the **web.config** file. This is because **Orchard.WarmupStarter** also provides an HttpModule (which is a bit out of scope in this article, but will likely be mentioned elsewhere at soime point).

`Starter<T>` (**Starter.cs**) is where we want to look. We've seen that we're passing it three methods which it can call (with suitable signatures) and when we construct it it simply stores those as member variables. When we then call `OnApplicationStart` it calls `LaunchStartupThread`, passing the HttpApplication instance that we've just given it. `LaunchStartupThread` looks like this:

{% highlight csharp %}

public void LaunchStartupThread(HttpApplication application) {
    // Make sure incoming requests are queued
    WarmupHttpModule.SignalWarmupStart();

    ThreadPool.QueueUserWorkItem(
        state => {
            try {
                var result = _initialization(application);
                _initializationResult = result;
            }
            catch (Exception e) {
                lock (_synLock) {
                _error = e;
                _previousError = null;
            }
        }
        finally {
            // Execute pending requests as the initialization is over
            WarmupHttpModule.SignalWarmupDone();
        }
    });
}

{% endhighlight %}

Most of this is just concerned with making sure that this is run asynchronously (and working with the previously mentioned module) and that any errors which result are caught and recorded/dealt with properly, but the key part is that we're initializing `_initializationResult` to the result of the method we passed in on construction. This is our generic type, which we've seen is of type `IOrchardHost`. 

This is important - this is the host that's going to be the root of our Orchard application &mdash; anything that happens in Orchard is going to happen as a result of calls in to this root object and the objects it will create and maintain. It's stored in the `Starter<T>` which in turn is a static member of our web application, so the lifetime for this root object is the same as the application itself.

So &mdash; what's the root object? We need to pop back to our **global.asax.cs** and see what the method which constructs our root object gives us.

## OrchardStarter

The method that gets called to create our host object is `HostInitialization` in **global.asax.cs**. That looks like this:

{% highlight csharp %}

private static IOrchardHost HostInitialization(HttpApplication application) {
    var host = OrchardStarter.CreateHost(MvcSingletons);

    host.Initialize();

    // initialize shells to speed up the first dynamic query
    host.BeginRequest();
    host.EndRequest();

    return host;
}

{% endhighlight %}

We're getting our `IOrchardHost` instance by calling `OrchardStarter.CreateHost`. This is where the meat of our object instantiation is going to happen. It's also the first place that we see another key element of Orchard, the use of *IoC*.

<aside>

If you're not familiar with IoC that's a pretty big topic in its own right, and I'm not about to go in to a lengthy discussion of that &mdash; these articles are plenty long enough on their own. Martin Fowler has written a solid and detailed explanation of the principles of [Dependency Injection][] which covers IoC, and the [Autofac Wiki][] has plenty of information about IoC and Autofac (the IoC container which Orchard uses under the hood). It'll probably help to understand IoC when working with Orchard.

</aside>

[Orchard Internals]: /orchard/2011/08/26/orchard-internals-series.html
[Dependency Injection]: http://martinfowler.com/articles/injection.html
[Autofac Wiki]: http://code.google.com/p/autofac/wiki/GettingStarted
