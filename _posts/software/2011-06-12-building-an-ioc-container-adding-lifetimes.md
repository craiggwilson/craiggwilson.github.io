---
layout: post
title: Building an IoC Container - Adding Lifetimes
description: How to add lifetimes to IoC registrations.
tags: [.net, ioc, solid]
image:
  feature: abstract-6.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: true
---

The code for this step is located on [github](https://github.com/craiggwilson/presentations/tree/master/BuildYourOwnIoC/code/Stage%205%20-%20Lifetimes).

We left off the [previous post]({% post_url software/2011-06-08-building-an-ioc-container-resolving-abstractions %}) with the need to support different lifetime models such as Singleton, Transient, or per Http Request. After our last refactoring, this is actually a fairly simple step if we follow the single responsiblity principle and define the roles each of our abstractions play.

Currently, we only have 1 abstraction which is around activation. It would be fairly simple to use activators to handle lifetime as well, but things will quickly become complex if we go this route. Therefore, we will define an activator's job as activating an object. In other words, it's only job is to construct an object using whatever means it wants, but it should never store the instance of an object in order to satisfy a lifetime requirement.

In order to manage lifetimes, we will introduce a second abstraction called an ILifetime. It's sole job is to manage the lifetime of an activated object. It can(and will) use an activator to get an instance of an object, but it will never construct one itself.

By keeping these two ideas seperate, it makes this a fairly trivial addition. Below are the two tests for this functionality, one for transient and one for singletons.

{% highlight csharp %}
public class when_resolving_a_transient_type_multiple_times : ContainerSpecBase
{
    static object _result1;
    static object _result2;

    Because of = () =>
    {
        _result1 = _container.Resolve(typeof(DummyService));
        _result2 = _container.Resolve(typeof(DummyService));
    };

    It should_not_return_the_same_instances = () =>
    {
        _result1.ShouldNotBeTheSameAs(_result2);
    };

    private class DummyService { }
}

public class when_resolving_a_singleton_type_multiple_times : ContainerSpecBase
{
    static object _result1;
    static object _result2;

    Establish context = () =>
        _container.Register().Singleton();

    Because of = () =>
    {
        _result1 = _container.Resolve(typeof(DummyService));
        _result2 = _container.Resolve(typeof(DummyService));
    };

    It should_return_the_same_instances = () =>
    {
        _result1.ShouldBeTheSameAs(_result2);
    };

    private class DummyService { }
}
{% endhighlight %}

You see above that we are making transient the default lifetime and adding a method to set a singleton lifetime.

{% highlight csharp %}
public interface ILifetime
{
    object GetInstance(Type type, IActivator activator, Func<Type, object> resolver);
}

public class TransientLifetime : ILifetime
{
    public object GetInstance(Type type, IActivator activator, Func<Type, object> resolver)
    {
        return activator.Activate(type, resolver);
    }
}

public class SingletonLifetime : ILifetime
{
    private object _instance;

    public object GetInstance(Type type, IActivator activator, Func<Type, object> resolver)
    {
        if (_instance == null)
            _instance = activator.Activate(type, resolver);
        return _instance;
    }
}
{% endhighlight %}

So, an ILifetime has a single method that takes the type, the activator, and the resolver. We are going to clean this up in the next installment, but regardless, these implementations are still relatively simple.

The TransientLifetime is basically a pass-through on the way to the activator. It doesn't store anything, so it has no need to do any interception. The SingletonLifetime, however, only activates an object once, stores the instance, and then returns that instance everytime. I haven't included threading in here, but a simple lock would suffice for most cases.

We need to add a Lifetime and a Singleton method onto our Registration class:

{% highlight csharp %}
public class Registration
{
    //other properties

    public ILifetime Lifetime { get; private set; }

    public Registration(Type concreteType)
    {
        ConcreteType = concreteType;
        Activator = new ReflectionActivator();
        Lifetime = new TransientLifetime();

        Aliases = new HashSet();
        Aliases.Add(concreteType);
    }

    //other methods

    public Registration Singleton()
    {
        Lifetime = new SingletonLifetime();
        return this;
    }
}
{% endhighlight %}

Finally, we just need to change the Resolve method in our Container class to use the Lifetime as opposed to the Activator and we are all done.

{% highlight csharp %}
public object Resolve(Type type)
{
    var registration = FindRegistration(type);
    return registration.Lifetime.GetInstance(registration.ConcreteType, registration.Activator, Resolve);
}
{% endhighlight %}

In our [next post]({% post_url software/2011-06-15-building-an-ioc-container-refactoring %}), we'll do some refactoring ultimately to support handling cyclic dependencies. A by product of this is a better coded container. See you then...