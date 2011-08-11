---
layout: post
title: Examining the Orchard start up process
categories:
- Orchard Internals
---

## Introduction

This is the first in a series of posts diving in to the internals of the Orchard CMS. The existing documentation does include an overview of the architecture ([How Orchard Works][]) - this series is for people that want to go a bit more deeply in to the actual code and understand how some of the features of Orchard are built.

These will probably be quite long posts - there's a lot of code to look at, and the treatment will be quite exhaustive.

I should emphasise that I'm *not* part of the Orchard team and *didn't write* any of the code I'll be looking at. Given that, if I make some mistakes in interpretation I'll be happy to apologise and correct them (if someone with knowledge closer to the metal wants to get in touch and correct me).

## Context

Orchard does quite a lot when it starts up. Setting up the initial application, the boot up process, is the foundation for all of the functionality that's then used by actual requests to an Orchard site. Let's take a look and see what gets run.

The first place to look is the Web Application itself. If you're looking at the Orchard source, this is the `Orchard.Web` project. The first thing to notice is that there's not much in this project - it's a very barebones ASP.NET MVC 3 application. The application itself is not a source of content. This means that the only points where Orchard can introduce itself are in the `MvcApplication` class, inheriting `HttpApplication` (`global.asax.cs`) and via configuration in `web.config`.

We'll look at the `MvcApplication` class first of all.

## MvcApplication

### Differences between Orchard and default ASP.NET MVC 3

Open up `global.asax.cs`. There's a bit more going on in here than with a default ASP.NET MVC 3 web app. There's also a bit *less*. This is because Orchard provides alternative ways of approaching some things which ASP.NET MVC 3 would usually do out of the box. So there's no registration of Areas (`AreaRegistration.RegisterAllAreas()`) or of Global Filters in the Orchard implementation. Orchard does different things in these areas, so they're not needed at this point.

It does still register routes, but the web application only registers a single route to ignore (`{resource}.axd/{*pathInfo}`). All other routes which the application needs will be provided by Orchard modules later on in the boot up process.

### Orchard Starter

The majority of the code in the `MvcApplication` class is concerned with the Orchard start and request lifecycle. The important piece of this is a static instance of `Starter<T>`, which is provided by a key Orchard module: `Orchard.WarmupStarter`. This is instantiated in Application Start, and is passed a few functions to call in the constructor. These are then called at Application Start, Begin Request and End Request. They're also in the MvcApplication, so we can see what's happening here.

The first to be called is `HostInitialization`, which is called by our instance of `Starter<T>` on Application Start. `T` in this case is `IOrchardHost` - the function called must return an instance of this type.

`HostInitialization` does exactly what it says - it uses `OrchardStarter` (found in the `Orchard.Environment` namespace) to create a new instance of a host, implementing `IOrchardHost`. It then calls `Initialize()` on the created host and returns it (it also calls other methods just to speed up later calls to them).

The functions (actually Actions) called on Begin and End Request take an instance of HttpApplication and an instance of our `IOrchardHost` host. They call `BeginRequest()` and `EndRequest()` on this instance respectively.

So we can see that the host is clearly a major entry point in to Orchard - pretty much everything that happens is going to have some connection to this host.

### Dependencies

One thing to note about the creation of our host instance - the creator was passed a function which took a `ContainerBuilder`. This is our first sight of the IoC container at the core of Orchard. Orchard uses [Autofac][] to provide IoC capabilities internally and to any modules which you might write to extend Orchard. At the time of creation, the function passed to the creator of our host instance is registering the `RouteCollection`, `ModelBinderDictionary` and  `ViewEngineCollection` so they're available via IoC to any component which might need them.

At this point Orchard is using Autofac directly, but when creating dependencies in custom modules, it's usual to use the abstraction that Orchard provides over Autofac by making a service implement `IDependency` or similar.

Discussing [Autofac][] is outside the scope of these posts, but there's a good wiki at the [Autofac][] site if you want to get a better understanding of IoC in general or the specifics of this container.

## The Host

Let's move away from our `MvcApplication` class now and see what the host is and how it's created. The host is created by the `OrchardStarter` class, which can be found in the Orchard.Framework project in the Environment folder (namespaces don't always match folder structure in Orchard, as a side note!).

{% highlight csharp %}
// Comment here
public static void Method(int arg1)
{
    var x = 5;
    var output = arg1 + x;
    Console.WriteLine(output);
}
{% endhighlight %}

When `CreateHost(...)` is called on OrchardStarter (which is a static class), it creates a new Autofac container. `OrchardStarter` is primarily about registering all of the Orchard core dependencies with the IoC container.

[How Orchard Works]: http://www.orchardproject.net/docs/How-Orchard-works.ashx
[Autofac]: http://code.google.com/p/autofac/
