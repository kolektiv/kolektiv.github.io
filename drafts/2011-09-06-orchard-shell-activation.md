---
title: Orchard Shell Activation
layout: post
categories:
- orchard
---

This is part 3 of a series on [Orchard Internals][]. See [the introductory post][Orchard Internals] for previous posts (and a full post listing).

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

Routes in Orchard are registered by implementing `IRouteProvider` in modules. Looking at that interface, unsurprisingly it implements `IDependency`. This means that `DefaultOrchardShell` can take a constructor dependency on `IEnumerable<IRouteProvider>` and get back all of the routes which need to be registered for all modules, thanks to the IoC implementation scanning all of the loaded modules.

It simply calls `GetRoutes` on each of them and then sends the resulting collection of `RouteDescriptor` instances off to the route publisher to be published (registered) for our shell.

There's a bit more though. Routes in an ASP.NET MVC 3 application are global &mdash; there's only one actual site here, despite the multi-tenancy possibilities, so how are we making sure that only the routes for our shell are used to get to our tenant?

This is where Orchard can go back and take advantage of some of the extensibility options built in to the ASP.NET MVC 3 framework. When you register a route in ASP.NET MVC 3, the method signature on the `RouteCollection` actually takes a `RouteBase`, which can be extended. So the route publisher actually orders the `RouteDescriptor` instances it has (by the Priority property which you can set when registering routes &mdash; as the order of routes is important in ASP.NET MVC 3) and then turns each route descriptor in to an instance of `ShellRoute`, which extends `RouteBase`.

It's worth seeing what `ShellRoute` is and how it works as the principle could be applied elsewhere. It can be found in **Mvc/Routes/ShellRoute.cs** in the **Orchard.Framework** project.

The crucial points in here are that it overrides `GetRouteData` and `GetVirtualPath` to ensure that the route only responds and matches if the request is for the right shell. It does this with the following code at the start of each of those methods:

{% highlight csharp %}

// locate appropriate shell settings for request
var settings = _runningShellTable.Match(httpContext);

// only proceed if there was a match, and it was for this client
if (settings == null || settings.Name != _shellSettings.Name)
    return null;

{% endhighlight %}

It's checking to see whether there's a match between the settings of the current request, which it can get from the running shells table which we saw previously, and the shell settings for this particular shell (`_shellSettings`, which it got as a constructor dependency). If there's no match, it returns null which tells ASP.NET MVC 3 that this route doesn't match this request. There's a bit more cleverness in here around making sure that paths can be found based on the location of the module itself, which you can have a glance at in here if you're interested!

For now though, we'll content ourselves with knowing how the routes are registered on a per shell basis and then how they behave during a request to the application, based on which shell the request was for.

## Model Binders and Shells

If we go back to our `DefaultOrchardHost` the next line after publishing our routes was registering model binders. This is another ASP.NET MVC concept and the way we're doing this looks very similar to the way Orchard handled the routes.

In fact it's pretty much exactly the same, meaning we don't need to dive in to this too far, except to note one thing &mdash; it isn't really implemented! Our `ModelBinderPublisher` actually looks like this:

{% highlight csharp %}

public class ModelBinderPublisher : IModelBinderPublisher {
    private readonly ModelBinderDictionary _binders;

    public ModelBinderPublisher(ModelBinderDictionary binders) {
        _binders = binders;
    }

    public void Publish(IEnumerable<ModelBinderDescriptor> binders) {
        // MultiTenancy: should hook default model binder instead and rely on shell-specific binders (instead adding to type dictionary)
        foreach (var descriptor in binders) {
            _binders[descriptor.Type] = descriptor.ModelBinder;
        }
    }
}

{% endhighlight %}

As you can see, it doesn't really deal with multi-tenancy for model binders yet &mdash; Orchard is only at version 1.2xxx after all! In practice it probably isn't a problem, but you'll know where to look if it is, or when it gets implemented.

## Orchard Events

The last part of the `Activate` method in the shell was this:

{% highlight csharp %}

using (var events = _eventsFactory()) {
    events.Value.Activated();
}

{% endhighlight %}

It's using `_eventsFactory`, which is a function it acquired as a constructor dependency, to get an instance of `IOrchardShellEvents`, and calling the `Activated` method on it.

This can be used to signal changes in application status to allow things like background tasks (which will be covered in another article) to take actions at key points in the application lifecycle.

## Activation Complete!

That's essentially it for the shell activation. As always I'd encourage you to go and dive deeper in to the source around the parts I've called out here &mdash; there's a lot of interesting code although I've tried to highlight the most relevant parts. 

We've seen how the shell registers routes and binders when it starts up in such a way as to have them only apply to the correct shell. We also saw how the dependency system influences the design of the framework here &mdash; recall how the IoC implementation made it easy for Orchard to get all of the route providers that had been implemented across all relevant modules.

Hopefully you'll come back for the next article, which is going to start to look at the request lifecycle &mdash; now that we've got our application up and running. It'll be linked from the [overview][Orchard Internals] once it's ready.

[Orchard Host]: /orchard/2011/09/01/orchard-host.html
[Orchard Startup]: /orchard/2011/08/30/orchard-startup-process.html
[Orchard Internals]: /orchard/2011/08/26/orchard-internals-series.html
