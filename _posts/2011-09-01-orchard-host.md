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

We saw that `Initialize` gets called as part of the startup process &mdash; that's pretty simple, and aside from some logging it just calls `BuildCurrent`. That looks a bit more interesting though, like this:

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

We'll ignore the locking as that's self-explanatory &mdash; we only want to initialize `_current` once regardless of threading. So we're left with a couple of calls relating to extensions, and then a call which creates "some" `ShellContext`s. It looks like there's a lot to this &mdash; we'll take a look at extensions first.

## Extensions

Thankfully the naming of the methods in Orchard is generally self-explanatory. `SetupExtensions` does what we'd expect &mdash; it calls a method to setup extensions on `_extensionLoaderCoordinator`, one of the dependencies that the host acquired on construction. What does that actually do though? First we'll need to look at what implementation of `IExtensionLoaderCoordinator` we're getting and we can see from a look at our dependency registrations (see [part 1][Orchard Startup] for where these are) that this is being implemented by `ExtensionLoaderCoordinator` which is at **Environment/Extensions/ExtensionLoaderCoordinator.cs** in the **Orchard.Framework** project.

Now we can see that it's starting to get a bit more complex! To sum it up, this is where one of the features of Orchard comes in to play, the dynamic compilation, loading and management of extensions &mdash; at runtime. What's an extension in this case? Well we're probably more used to saying Modules and Themes, but they're both a form of extension. 

I'm not going to dive in to the details of the extension discovery, compilation and loading mechanisms as part of this article &mdash; it's a big enough area that it needs an article all to itself which will be coming later on. For now we'll skim over this and say that Orchard is finding any Modules and Themes by looking in some predefined directories and then loading them in to the AppDomain so that they're available for Orchard to use. `MonitorExtensions` is part of this same process &mdash; it monitors the available extensions on disk (in the previously mentioned directories) and reloads them if they change. 

That's simplifying quite a large and complex process, but it's enough to give us a feel for what's happening at the host level. We can now move on to the last key line of `BuildCurrent`, the call to `CreateAndActivate`.

## Shells

Let's start by looking at that `CreateAndActivate` method:

{% highlight csharp %}

IEnumerable<ShellContext> CreateAndActivate() {
    Logger.Information("Start creation of shells");

    IEnumerable<ShellContext> result;
    var allSettings = _shellSettingsManager.LoadSettings();
    if (allSettings.Any()) {
        result = allSettings.Select(
        settings => {
            var context = CreateShellContext(settings);
            ActivateShell(context);
            return context;
        });
    }
    else {
       var setupContext = CreateSetupContext();
       ActivateShell(setupContext);
       result = new[] {setupContext};
    }

    Logger.Information("Done creating shells");
    return result;
}
		
{% endhighlight %}

We'll ignore the obvious bits like logging. We start by getting any `ShellSettings` instances that the `_shellSettingsManager` (an instance of `IShellSettingsManager` that came in as a constructor dependency). If we have any instances, we create one ShellContext for each instance, activating each as we go. We then return that list of activated shell contexts. If we didn't have any instances of `ShellSettings` we just create a new shell context (with some default settings), activate it, and return that (as the only member of an array of shell contexts).

We'll look at what creating a shell entails in a moment but let's look first at why sometimes we might have some instances of `ShellSettings` and sometimes not. Our instance of `IShellSettingsManager` is responsible here, so let's see what implementation is being used. Looking at our dependencies it's `ShellSettingsManager` which can be found in **Environment/Configuration/ShellSettingsManager.cs**. 

Let's take a look in there and see why we would (or wouldn't) have any shell settings.

## ShellSettingsManager

We get our shell settings by calling the  `LoadSettings` method in this class. Let's have a look:

{% highlight csharp %}

IEnumerable<ShellSettings> LoadSettings() {
    var filePaths = _appDataFolder
        .ListDirectories("Sites")
        .SelectMany(path => _appDataFolder.ListFiles(path))
        .Where(path => string.Equals(Path.GetFileName(path), "Settings.txt", StringComparison.OrdinalIgnoreCase));

    foreach (var filePath in filePaths) {
        yield return ParseSettings(_appDataFolder.ReadFile(filePath));
    }
}

{% endhighlight %}

While it uses quite a few Orchard classes to do it, what this method is doing is actually quite straightforward. It's finding all of the directories in the **Sites** directory in our App_Data folder  which contain a file called **Settings.txt**. For each one that it finds, it parses the **Settings.txt** file in to a `ShellSettings` instance and returns it. That's pretty straightforward, so in what cases would we end up with differing numbers?

The first case, where we have no instances of `ShellSettings`, is simple. The Orchard installer process has not yet created the default site (usually a directory called "Default" under the **Sites** directory). Once that's been run and the site has gone through the initial setup, you'll always have at least one instance of `ShellSettings` when this gets called. So we'd only see no instances of `ShellSettings` on our first run (or until we've got a site set up).

The final case is having *more than one* instance. For this to happen we'd need to have more than one site set up in our **Sites** directory. Why would we have this? This would be the case if we were running Orchard with multi-tenancy enabled, and we'd set up multiple tenants for our Orchard site. By default Orchard doesn't load this module, but it can be turned on like any other if it's installed.

Multi-tenancy is another big topic to cover, and it will get an article all to itself later on, but if you're curious now, the **Orchard.MultiTenancy** project (module) is where the magic happens for this. For now, we'll take it as read that we might have multiple shells running, each with their own settings, allowing us to run multiple distinct sites within one instance of Orchard.

[Orchard Startup]: /orchard/2011/08/30/orchard-startup-process.html
[Orchard Internals]: /orchard/2011/08/26/orchard-internals-series.html
