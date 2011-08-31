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

If we look in the **global.asax.cs** file of the Orchard.Web project, this is where we'll find our Orchard code. We don't even have some of the code which would normally be present in a default ASP.NET MVC 3 web app &mdash; there's no registration of Areas or Filters in here as Orchard deals with these itself. It doesn't map any Routes either, only ignoring the Route for resource.axd, as Orchard deals with Routes too.

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

We've seen that we're creating a new `Starter<IOrchardHost>` here, so let's have a look at that. This class can be found in the **Orchard.WarmupStarter** project, which is the only Orchard project which is explicitly referenced in the web application - it's referenced in the **web.config** file. This is because **Orchard.WarmupStarter** also provides an HttpModule (which is a bit out of scope in this article, but will likely be mentioned elsewhere at some point).

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

<aside><p>
<strong>Note</strong>: If you're not familiar with IoC that's a pretty big topic in its own right, and I'm not about to go in to a lengthy discussion of that &mdash; these articles are plenty long enough on their own. Martin Fowler has written a solid and detailed explanation of the principles of <a href="http://martinfowler.com/articles/injection.html">Dependency Injection</a> which covers IoC, and the <a href="http://code.google.com/p/autofac/wiki/GettingStarted">Autofac Wiki</a> has plenty of information about IoC and Autofac (the IoC container which Orchard uses under the hood). It'll probably help to understand IoC when working with Orchard.
</p></aside>

Let's look at `OrchardStarter` and see what it's doing. It's in the **Orchard.Framework** project in **Environment/OrchardStarter.cs**. We start off by calling `CreateHost`, passing in an Action which takes a `ContainerBuilder`. This is an Autofac concept, but essentially it's the class used to register dependencies when we build an IoC container. At the moment it's being passed a method `MvcSingletons` in our **global.asax.cs** which registers the Routes, Binders and View Engines as they are currently set up for the application, making them available to the IoC container.

`CreateHost` looks like this:

{% highlight csharp %}

public static IOrchardHost CreateHost(Action<ContainerBuilder> registrations) {
    var container = CreateHostContainer(registrations);
    return container.Resolve<IOrchardHost>();
}

{% endhighlight %}

It gets an IoC container by calling `CreateHostContainer` and then asks that container for an instance of `IOrchardHost`. `CreateHostContainer` (which I won't give source for here, but it's a very useful place to look if you're wondering what implentation of an interface is used by default) registers all of the default dependencies that Orchard needs to run &mdash; all of the dependencies needed to create an `IOrchardHost`. So we can see that IoC is key to Orchard &mdash; there's no manual creation of object graphs here, just IoC being used to satisfy all of the dependencies at runtime. 

This is also a really useful place to look if you're wondering about the lifecycle of any of the components of Orchard. It's likely that they're registered here, with the lifecycle being managed by the IoC container (Autofac is very good in this regard). 

## DefaultOrchardHost

At this point, if all has gone well, we have an instance of `IOrchardHost`. Looking at `CreateHostContainer` we can see that this will actually be an instance of `DefaultOrchardHost` &mdash; that's the type which has been registered for this interface. Now we can go back to our web application in **global.asax.cs** and wrap up the tour of the startup process by looking what happens to our host.

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

Looking again at our host initialization code, we can see that we call `Initialize` on our host. That's pretty much it &mdash; the other calls on our host here are just there for performance reasons and aren't really relevant to our startup process.

## Wrapping Up

So we've been through the startup procedure and seen how Orchard creates the runtime environment. We haven't looked at the actual host much yet &mdash; we'll look at that in the next article in this series, but we have seen how a host is constructed so we can probably have a guess that by now our host has all of the dependencies it needs and is ready to start handling Orchard requests.

We've also seen where to find the default registrations and lifecycles for the core Orchard components and how those are made available to the host.

We're now ready to start taking a deep dive in to the [Orchard Host][].

[Orchard Internals]: /orchard/2011/08/26/orchard-internals-series.html
[Orchard Host]: /orchard/2011/09/01/orchard-host.html
