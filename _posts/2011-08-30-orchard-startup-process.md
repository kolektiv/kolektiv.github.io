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

Everything starts off in the web application itself. If you've got the source code handy, this is the Orchard.Web project. Looking in here we see... not a lot. This is because Orchard only really uses this as a boot up point for the application proper. There's no content in here or traditional models, controllers and views &mdash; only the code needed to call out to Orchard and get things running.

If we look in the global.asax.cs file in the root of the Orchard.Web project, this is where we'll find our Orchard code. We don't even have some of the code which would normally be present in a default ASP.NET MVC 3 web app &mdash; there's no registration of Areas or Filters in here as Orchard deals with these itself. It doesn't map any Routes either, only ignoring the Route for resource.axd, as Orchard deals with Routes too.

The first place to look is in the application start method, which looks like this:

{% highlight csharp %}

protected void Application_Start() {
    RegisterRoutes(RouteTable.Routes);
    _starter = new Starter<IOrchardHost>(HostInitialization, HostBeginRequest, HostEndRequest);
    _starter.OnApplicationStart(this);
}

{% endhighlight %}

The call to `RegisterRoutes` is normal, but then we're straight in to Orchard code. We create a new `Starter<IOrchardHost>` which is a static member of the application, so it's always around.

This gets passed a few arguments on construction, each of which is a method to call at different application stages &mdash; on start, beginning a request, and ending a request. The one which really matters to us here is the method which is going to get called on application start &mdash; and which is actually invoked as part of calling `OnApplicationStart` on our instance of `Starter<IOrchardHost>`.

[Orchard Internals]: /orchard/2011/08/26/orchard-internals-series.html
