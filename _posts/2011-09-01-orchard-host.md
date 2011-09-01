---
title: Orchard Host
layout: post
categories:
- orchard
---

This is part 2 of a series on [Orchard Internals][]. See [the introductory post][Orchard Internals] for previous posts (and a full post listing).

## Overview

In [part 1][Orchard Startup] of this series we looked at the startup process in Orchard, as far as obtaining and initializing a host (`IOrchardHost`, by default an instance of `DefaultOrchardHost`). We saw it get initialized and become a key root point of the application but we didn't look inside it and see what's happening on initialization. Well &mdash; now we will.

## DefaultOrchardHost

Let's go and have a look at `DefaultOrchardHost` &mdash; it's in the **Orchard.Framework** project in **Environment/DefaultOrchardHost.cs**. It's a good job Orchard is using IoC to create instances of this &mdash; looking at the constructor it probably wouldn't be fun to manage that many dependencies manually!

We say that `Initialize` gets called as part of the startup process - that's pretty simple, and aside from some logging it just calls `BuildCurrent`. That looks a bit more interesting though, like this:

{% highlight csharp %}

IEnumerable<ShellContext> BuildCurrent() {
    if (_current == null) {
        lock (_syncLock) {
            if (_current == null) {
                SetupExtensions();
                MonitorExtensions();
                _current = CreateAndActivate().ToArray();
            }
        }
    }
    return _current;
}

{% endhighlight %}

We'll ignore the locking as that's pretty self-explanatory - we only want to initialize `_current` once regardless of threading. So we're left with a couple of calls relating to extensions, and then a call which creates "some" `ShellContext`s. It looks like there's a lot to this &mdash; we'll take a look at extensions first.

## Extensions

Thankfully the naming of the methods in Orchard is generally self-explanatory. `SetupExtensions` does what we'd expect &mdash; it calls a method to setup extensions on `_extensionLoaderCoordinator`, one of the dependencies that the host acquired on construction. What does that actually do though? First we'll need to look at what implementation of `IExtensionLoaderCoordinator` we're getting and we can see from a look at our dependency registrations (see [part 1][Orchard Startup] for where these are) that this is being implemented by `ExtensionLoaderCoordinator` which is at ** Environment/Extensions/ExtensionLoaderCoordinator.cs** in the **Orchard.Framework** project.

Now we can see that it's starting to get a bit more complex! To sum it up, this is where one of the features of Orchard comes in to play, the dynamic compilation, loading and management of extensions &mdash; at runtime. What's an extension in this case? Well we're probably more used to saying Modules and Themes, but they're both a form of extension.

[Orchard Startup]: /orchard/2011/08/30/orchard-startup-process.html
[Orchard Internals]: /orchard/2011/08/26/orchard-internals-series.html
