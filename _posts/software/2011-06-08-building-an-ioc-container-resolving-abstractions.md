---
layout: post
title: Building an IoC Container - Resolving Abstractions
description: How to resolve abstractions.
tags: [.net, ioc, solid]
image:
  feature: abstract-5.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: true
---

The code for this step is located on [github](https://github.com/craiggwilson/presentations/tree/master/BuildYourOwnIoC/code/Stage%204%20-%20Resolving%20Abstractions).

From our [previous post]({% post_url software/2011-06-06-building-an-ioc-container-resolving-types-with-unresolvable-dependencies %}), we added the ability to register dependencies with dependencies that couldn't be resolved by the container.  These would be dependencies like primitives or abstractions like interfaces.  In this post, we are going to solve our inability to resolve an abstraction by adding aliases to the Registration class.

Below is our test for this functionality:

{% highlight csharp %}
public class when_resolving_a_type_by_its_alias : ContainerSpecBase
{
    static object _result;

    Establish context = () =>
        _container.Register<DummyService>().As<IDummyService>();

    Because of = () =>
        _result = _container.Resolve(typeof(IDummyService));

    It should_not_return_null = () =>
        _result.ShouldNotBeNull();

    It should_return_an_instance_of_the_requested_type = () =>
        _result.ShouldBeOfType<DummyService>();

    private interface IDummyService { }
    private class DummyService : IDummyService { }
}
{% endhighlight %}

Above, the only difference from an API standpoint is the addition of the "As" method.  This is basically telling the container that DummyService should be returned when IDummyService is requested.

So, the first change we'll make is on the Registration class.  We'll add a property called Aliases.  Aliases will include all the types that should resolve to the same concrete class including the concrete version.  So, Register().As().As() will resolve when any of the types ISomeServiceA, ISomeServiceB, or SomeService is requested.  Our new registration class looks like this:

{% highlight csharp %}
public class Registration
{
    public Type ConcreteType { get; private set; }

    public IActivator Activator { get; private set; }

    public ISet<Type> Aliases { get; private set; }

    public Registration(Type concreteType)
    {
        ConcreteType = concreteType;
        Activator = new ReflectionActivator();

        Aliases = new HashSet<Type>();
        Aliases.Add(concreteType);
    }

    public Registration ActivateWith(IActivator activator)
    {
        Activator = activator;
        return this;
    }

    public Registration ActivateWith(Func<Type, Func<Type, object>, object> activator)
    {
        Activator = new DelegateActivator(activator);
        return this;
    }

    public Registration As<T>()
    {
        Aliases.Add(typeof(T));
        return this;
    }
}
{% endhighlight %}

The only other change we need to make is in the container where we are trying to find a registration.

{% highlight csharp %}
private Registration FindRegistration(Type type)
{
    var registration = _registrations.FirstOrDefault(r => r.Aliases.Contains(type));
    if (registration == null)
        registration = Register(type);

    return registration;
}
{% endhighlight %}

That's it!  All the tests should still pass and all is good with the world. In the [next post]({% post_url software/2011-06-12-building-an-ioc-container-adding-lifetimes %}), we are going to talk about lifetimes (Singleton, Transient, PerRequest, etc...) and how to add them into our container.

Stay Tuned.