---
title: Orchard Shell Activation
layout: post
categories:
- orchard
---

This is part 2 of a series on [Orchard Internals][]. See [the introductory post][Orchard Internals] for previous posts (and a full post listing).

## Overview

In [part 2][Orchard Host] we looked at the initialization of the Orchard host, and how that created new shell contexts and shells to host Orchard sites. We left off at the point where a host was pretty much complete and Orchard was ready to begin serving requests, but we skipped over the activation of our actual shells. We're going to look in there in this article, and see what happens on that call to `Activate` on our shell once it's been created.

## DefaultOrchardShell

We can see from our dependency registrations that our instance of `IOrchardShell` is going to be an instance of `DefaultOrchardShell`, which is found in **Environment/DefaultOrchardShell.cs**. We'll take a look at that now. First of all, let's look at the constructor:

{% highlight csharp %}

public DefaultOrchardShell(
    Func<Owned<IOrchardShellEvents>> eventsFactory,
    IEnumerable<IRouteProvider> routeProviders,
    IRoutePublisher routePublisher,
    IEnumerable<IModelBinderProvider> modelBinderProviders,
    IModelBinderPublisher modelBinderPublisher) {
    _eventsFactory = eventsFactory;
    _routeProviders = routeProviders;
    _routePublisher = routePublisher;
    _modelBinderProviders = modelBinderProviders;
    _modelBinderPublisher = modelBinderPublisher;

    Logger = NullLogger.Instance;
}

{% endhighlight %}

We wouldn't normally be too interested in this &mdash; it's standard constructor injection in IoC. But it's worth remembering that this instance of `DefaultOrchardShell` was resolved in the shell scope, so those dependencies are shell scoped dependencies. It's also worth remembering that the dependencies at the shell level aren't necessarily ones that have been registered in the traditional way, as seen in `OrchardStarter`. They've probably been registered in the Orchard way, by finding dependencies that implement `IDependency` and the related family of interfaces. This means that we can't look up dependencies quite as easily, as they're now being registered dynamically.

For example, let's look at the instance of `IRoutePublisher` which is a constructor dependency. That interface looks like this:

{% highlight csharp %}

namespace Orchard.Mvc.Routes {
    public interface IRoutePublisher : IDependency {
        void Publish(IEnumerable<RouteDescriptor> routes);
    }
}

{% endhighlight %}

We can see that it implements IDependency, so that any class which implements that interface will be automatically registered as a dependency. In this case that class is `RoutePublisher` in the **Orchard.Framework** project, found in **Mvc/Routes/RoutePublisher.cs**. It was dynamically registered as a dependency when we were creating our shell lifetime scope, and when we resolved our shell instance, it was resolved from that scope. This is a good thing to remember and be aware of.

## Activation

Once we've got our `DefaultOrchardShell` instance, the host is going to call `Activate` when we're initializing our host, as we've seen in previous posts. Happily, the `Activate` method looks pretty straightforward, like this:

{% highlight csharp %}

public void Activate() {
    _routePublisher.Publish(_routeProviders.SelectMany(provider => provider.GetRoutes()));
    _modelBinderPublisher.Publish(_modelBinderProviders.SelectMany(provider => provider.GetModelBinders()));

    using (var events = _eventsFactory()) {
        events.Value.Activated();
    }
}

{% endhighlight %}

Not too much going on &mdash; publishing the routes and model binders which are part of our site. (We also do some work with events, (`IOrchardShellEvents`) which we'll look at later on. Before then, let's look at how routes work within a shell.

## Routes and Shells

As we can see, routes are published on a per shell basis, so routes can be different between tenants. We also know from the [first article][Orchard Startup] that routes are not registered as part of our main web application as they often are in ASP.NET MVC web applications.

[Orchard Host]: /orchard/2011/09/01/orchard-host.html
[Orchard Startup]: /orchard/2011/08/30/orchard-startup-process.html
[Orchard Internals]: /orchard/2011/08/26/orchard-internals-series.html
