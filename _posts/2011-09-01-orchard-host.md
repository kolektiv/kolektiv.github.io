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

## Orchard Shells

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

The first case, where we have no instances of `ShellSettings`, is simple. The Orchard installer process has not yet created the default site (a directory called "Default" under the **Sites** directory). Once that's been run and the site has gone through the initial setup, you'll always have at least one instance of `ShellSettings` when this gets called. So we'd only see no instances of `ShellSettings` on our first run (or until we've got a site set up).

The final case is having *more than one* instance. For this to happen we'd need to have more than one site set up in our **Sites** directory. Why would we have this? This would be the case if we were running Orchard with multi-tenancy enabled, and we'd set up multiple tenants for our Orchard site. Each tenant would be another directory in **Sites** with separate settings. By default Orchard doesn't load this module, but it can be turned on like any other if it's installed.

Multi-tenancy is another big topic to cover, and it will get an article all to itself later on, but if you're curious now, the **Orchard.MultiTenancy** project (module) is where the magic happens for this. For now, we'll take it as read that we might have multiple shells running,  allowing us to run multiple distinct sites within one instance of Orchard.

<aside>
<p>
A shell has a one to one correspondence with a tenant in Orchard, so if you're running a multi-tenanted site, be aware that things are generally "shell local" &mdash; you won't usually be sharing instances across tenants. If tenants need to talk to each other, you might need to write your own way of doing that! 
</p>
</aside>

Now that we've seen where shell settings might come from (and why) let's take a look at what happens when shells are created and initialized. Back to our `DefaultOrchardHost` implementation...

## ShellContextFactory

We saw earlier that our shell contexts (instances of `ShellContext`) are created by calling either `CreateSetupContext` (where we have no shell settings) or `CreateShellContext` (where we do have shell settings). Both of these methods make calls to `_shellContextFactory`, an instance of `IShellContextFactory`, which in this case is implemented by `ShellContextFactory`. We call either `CreateSetupContext` or `CreateShellContext` on that instance, depending on whether or not we have settings (or if the settings we have say they haven't been initialized yet).

Let's go and take a look in `ShellContextFactory`, which can be found in **ShellBuilders/ShellContextFactory.cs**. It's worth looking at what happens in here as it's key to allowing Orchard to work the way it does with regard to IoC, and understanding that can save avoid confusion and problems down the line, especially when working with multiple tenants.

We'll take our usual stance of ignoring logging and that kind of thing, but we'll compare the two methods that are called, `CreateShellContext` and `CreateSetupContext`. Let's look at the start of both.

First `CreateShellContext`:

{% highlight csharp %}

Logger.Debug("Creating shell context for tenant {0}", settings.Name);

var knownDescriptor = _shellDescriptorCache.Fetch(settings.Name);
if (knownDescriptor == null) {
    Logger.Information("No descriptor cached. Starting with minimum components.");
    knownDescriptor = MinimumShellDescriptor();
}

{% endhighlight %} 

and `CreateSetupContext`:

{% highlight csharp %}

Logger.Warning("No shell settings available. Creating shell context for setup");

var descriptor = new ShellDescriptor {
    SerialNumber = -1,
    Features = new[] {
        new ShellFeature { Name = "Orchard.Setup" },
        new ShellFeature { Name = "Shapes" },
        new ShellFeature { Name = "Orchard.jQuery" },
    },
};

{% endhighlight %}

The obvious point here is that in a setup context we're creating a new descriptor for the shell we're creating, with only the features (modules) required to run the setup process, and to create a tenant for future use. Where we already have a tenant set up, we expect that we can get a shell descriptor from our cache, but if not we create one which is the minimum required at this stage. (At this point it's worth noting that a shell descriptor is basically a list of some metadata about a shell/tenant, such as the name, a unique identifier, and a list of features which should be enabled when available).

Once we've got a shell descriptor, let's see what happens to it:

{% highlight csharp %}

var blueprint = _compositionStrategy.Compose(settings, descriptor);
var shellScope = _shellContainerFactory.CreateContainer(settings, blueprint);

{% endhighlight %}

Those lines are the same in both methods (except for the name of the descriptor instance which is passed in &mdash; these lines were taken from `CreateSetupContext`).

Now we're getting to the critical part of the shell architecture. First of all, we create a `ShellBlueprint`, by calling `Compose` on our `_compositionStrategy` instance, passing in our settings and the descriptor that we've just created or acquired. What is that actually creating at this point? To find out, we'll have a quick glance in to **ShellBuilders/CompositionStrategy.cs** which is where we find `CompositionStrategy`, the implementation of `ICompositionStrategy` in use here.

The `Compose` method looks like this:

{% highlight csharp %}

public ShellBlueprint Compose(ShellSettings settings, ShellDescriptor descriptor) {
    Logger.Debug("Composing blueprint");

    var enabledFeatures = _extensionManager.EnabledFeatures(descriptor);
    var features = _extensionManager.LoadFeatures(enabledFeatures);

    if (descriptor.Features.Any(feature => feature.Name == "Orchard.Framework"))
        features = features.Concat(BuiltinFeatures());

    var modules = BuildBlueprint(features, IsModule, BuildModule);
    var dependencies = BuildBlueprint(features, IsDependency, (t, f) => BuildDependency(t, f, descriptor));
    var controllers = BuildBlueprint(features, IsController, BuildController);
    var records = BuildBlueprint(features, IsRecord, (t, f) => BuildRecord(t, f, settings));

    var result = new ShellBlueprint {
        Settings = settings,
        Descriptor = descriptor,
        Dependencies = dependencies.Concat(modules).ToArray(),
        Controllers = controllers,
        Records = records,
    };

    Logger.Debug("Done composing blueprint");
    return result;
}

{% endhighlight %}

This looks a little bit daunting at first glance &mdash; the `BuildBlueprint` method throwing generics and `Func<...>` around all over the place but it isn't as complicated as it seems. In fact almost everything here is about working out what dependencies need to be registered with our IoC container so that the shell can provide any dependencies as they're needed &mdash; for the right tenant and with the right *scope*. First of all there's some checking to see which features in our descriptor are actually available through the extension manager (and recording the ones that are). Next we make sure that if **Orchard.Framework** is required, all of the built in features from that project are recorded as being required too.

Once that's done, we start to call `BuildBlueprint`. While it's called with various arguments and functions, it's always doing roughly the same thing &mdash; building up a specification of what dependencies need to be registered for this shell. It does this through assembly scanning for certain types, with specific bits of logic for the varying things that might need to be registered (checking for a valid controller, a valid dependency). Once that's done we end up with an aggregation of all of the types of dependencies that this shell is going to need, based on the features that have been described.

This is now what we need to actually build our shell scope through the call we saw earlier to `_shellContainerFactory.CreateContainer`.

<aside>
<p>
At this point it's probably a good idea to make sure you're familiar with the concepts of Lifetime/Scope in Autofac if you want this to make good sense. It's not trivial but it is rather well defined and Nick Blumhardt has an excellent article laying it all out: <a href="http://nblumhardt.com/2011/01/an-autofac-lifetime-primer/">An Autofac Lifetime Primer</a>. It's worth reading that if you really want to understand Orchard Internals &mdash; Orchard is using Autofac directly in this area.
</p>
</aside>

That call to `CreateContainer` gives us an Autofac `ILifetimeScope` back, and that becomes a crucial part of our shell context. It means that pretty much anything the tenant needs to resolve (including the shell itself as we create a new `ShellContext` instance) will be resolved at the level of the shell lifetime scope, giving segregation and the possibility of very different configurations and loaded features between tenants.

The `ShellContainerFactory` class (**ShellBuilders/ShellContainerFactory.cs**) is the implementation used to build our `ILifetimeScope` from our shell settings and shell blueprint. It's worth a read through but I'm not going to go through it all here &mdash; there's quite a lot of code, most of which will be quickly readable if you're familiar with Autofac. It is worth pulling out one part though, which is where we see how Orchard lets you implement your own dependencies with modules (and control their lifetimes).

This code (around line 66 onwards) is **important**:

{% highlight csharp%}

foreach (var interfaceType in item.Type.GetInterfaces()
    .Where(itf => typeof(IDependency).IsAssignableFrom(itf) 
               && !typeof(IEventHandler).IsAssignableFrom(itf))) {
    registration = registration.As(interfaceType);
    if (typeof(ISingletonDependency).IsAssignableFrom(interfaceType)) {
        registration = registration.InstancePerMatchingLifetimeScope("shell");
    } 
    else if (typeof(IUnitOfWorkDependency).IsAssignableFrom(interfaceType)) {
        registration = registration.InstancePerMatchingLifetimeScope("work");
    }
    else if (typeof(ITransientDependency).IsAssignableFrom(interfaceType)) {
        registration = registration.InstancePerDependency();
    }
}

{% endhighlight %}

This is where Orchard is looking for the interfaces you would implement when creating your own module containing dependencies that you wish to make available through IoC. We can see clearly here how implementing `IDependency`, or `ISingletonDependency` will impact the registration of the type you provide within the scope of the shell. It's also useful to remember for the future &mdash; if you need to one day extend the Orchard core to deal with more complex registrations and scope within the IoC container this would be the place to start (although it's unlikely that's going to be needed for most people!)

<aside>
<p>
<strong>Note</strong>: if you're used to Autofac you'll be used to the default scope of a registration being `InstancePerDependency`, so you might assume that this is what you'd get if you implemented `IDependency` in Orchard. It's not though &mdash; the default in Orchard is a LifetimeScope which is the scope of a Request. To get the normal Autofac behaviour of an instance per dependency you should implement `ITransientDependency`.
</p>
</aside>

We can now go back to our `ShellContextFactory`. When we're creating a setup context, we're basically done now. The last part of that method looks like this:

{% highlight csharp %}

return new ShellContext {
    Settings = settings,
    Descriptor = descriptor,
    Blueprint = blueprint,
    LifetimeScope = shellScope,
    Shell = shellScope.Resolve<IOrchardShell>(),
};

{% endhighlight %} 

We can see that we're storing our scope in there, and we're also creating our actual shell instance by resolving an `IOrchardShell` &mdash; and importantly we're doing that in our newly created scope, meaning that we've got just the right registrations and setup for our tenant, isolated from other tenants and their own shells.

We can finish up looking at shell and shell context creation now by just tying up a loose end &mdash; the other difference between a setup context and a normal shell context. The following code appears in `CreateShellContext` before we return a newly created `ShellContext` with our shell scope:

{% highlight csharp %}

ShellDescriptor currentDescriptor;
using (var standaloneEnvironment = shellScope.CreateWorkContextScope()) {
    var shellDescriptorManager = standaloneEnvironment.Resolve<IShellDescriptorManager>();
    currentDescriptor = shellDescriptorManager.GetShellDescriptor();
}

if (currentDescriptor != null && knownDescriptor.SerialNumber != currentDescriptor.SerialNumber) {
    Logger.Information("Newer descriptor obtained. Rebuilding shell container.");

    _shellDescriptorCache.Store(settings.Name, currentDescriptor);
    blueprint = _compositionStrategy.Compose(settings, currentDescriptor);
    shellScope = _shellContainerFactory.CreateContainer(settings, blueprint);
}

{% endhighlight %}

At this point we check if there's actually a more up to date shell descriptor already in the environment. If there is, we recompose our blueprint and construct a new shell scope to use rather than the one we created with our cached shell descriptor (as well as sticking the descriptor back in the cache so it's there for next time).

From then on, we're back to just returning our newly created shell context, with a shell resolved from the new shell scope.

## DefaultOrchardHost

Finally (this has been a rather long article) we can take ourselves back to our `DefaultOrchardHost`, and see that the only really important thing that happens now is our call to `ActivateShell`. That looks like this: 

{% highlight csharp %}

private void ActivateShell(ShellContext context) {
    context.Shell.Activate();
    _runningShellTable.Add(context.Settings);
}

{% endhighlight %}

We can see that we're actually calling `Activate` on our instance of `IOrchardShell`, before registering that shell in our table of running shells.

While we could start to dive in to what activating a shell means, that's best served with a separate article &mdash; this one is more than long enough. So let's wrap that up there. We've seen how shells and shell contexts get created based on the site/tenant settings in our **App_Data/Sites** directory, and we've seen how the settings there go towards making up an IoC scope for the shell and context.

We've also seen how the dependencies we create in our own custom modules get registered with the underlying shell and scope, so we know what we can do by default with Orchard for dependency specification.

Next up we'll look at activating the shell, so check back to the [main introductory post and the list of articles][Orchard Internals] to see when that gets posted (or subscribe to the [Atom feed][Atom] if that's you're thing).

[Orchard Startup]: /orchard/2011/08/30/orchard-startup-process.html
[Orchard Internals]: /orchard/2011/08/26/orchard-internals-series.html
[Atom]: /atom.xml
