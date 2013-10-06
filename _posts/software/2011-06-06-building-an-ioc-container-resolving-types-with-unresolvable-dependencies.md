---
layout: post
title: Building an IoC Container - Resolving Types with Unresolvable Dependencies
description: How to resolve types that are unresolvable.
tags: [.net, ioc, solid]
image:
  feature: abstract-4.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: true
---

The code for this step is located on [github](https://github.com/craiggwilson/presentations/tree/master/BuildYourOwnIoC/code/Stage%203%20-%20With%20Unresolvable%20Dependencies).

From our previous [post]({% post_url software/2011-06-04-building-an-ioc-container-resolving-types-with-dependencies %}), we are able to resolve types with dependencies that they themselves can be resolved.  We all know from experience that this is hardly ever true without some help.  For instance, a type that takes an integer in its constructor would be impossible to resolve in our current state.  To combat this, we are going to create something called an activator.  It’s entire purpose is to, given a type, give back an instance of that type through construction.  The interface is defined below:

{% highlight csharp %}
public interface IActivator
{
    object Activate(Type type, Func<Type, object> resolver);
}
{% endhighlight %}

Nothing special here, but what is special is the abstraction we are creating between the activation of an instance and the implementation.  Our first implementation will be to refactor the current activation code embedded in the Resolve method of the container into a new activator called a ReflectionActivator.

{% highlight csharp %}
public class ReflectionActivator : IActivator
{
    public object Activate(Type type, Func<Type, object> resolver)
    {
        var ctor = type.GetConstructors()
                       .OrderByDescending(c => c.GetParameters().Length)
                       .First();

        var parameters = ctor.GetParameters();
        var args = new object[parameters.Length];
        for (int i = 0; i < args.Length; i++)
            args[i] = resolver(parameters[i].ParameterType);

        return ctor.Invoke(args);
    }
}
{% endhighlight %}

With the exception of the resolver Func that is passed in and its usage, this code is identical to the Resolve method.  The resolver argument has the same signature as the Resolve method, thereby allowing us to perform the same recursion we were using previously.  We can now replace the existing code in Resolve with the below method:

{% highlight csharp %}
public object Resolve(Type type)
{
    var activator = new ReflectionActivator();
    return activator.Activate(type, Resolve);
}
{% endhighlight %}

At this point, all our existing tests should still pass (because we really haven’t done anything).  Let's go ahead and refactor some more allowing different types to be registered with their own settings.  This is the final refactoring before we can solve the original stated problem.

Below is the Registration class.  It has a property called ConcreteType.  I’m naming it that because I know some things about the future that you don’t, but regardless, it holds the actual type that can be constructed.

{% highlight csharp %}
public class Registration
{
    public Type ConcreteType { get; private set; }

    public IActivator Activator { get; private set; }

    public Registration(Type concreteType)
    {
        ConcreteType = concreteType;
        Activator = new ReflectionActivator();
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
}
{% endhighlight %}

The take away from the class above is that we have a ConcreteType that has an instance of IActivator associated with it.  Therefore, an unresolvable ConcreteType can be activated differently than one that is resolvable.  For instance, the implementation of the IActivator that solves our original problem is called the DelegateActivator.

{% highlight csharp %}
public class DelegateActivator : IActivator
{
    private readonly Func<Type, Func<Type, object>, object> _activator;

    public DelegateActivator(Func<Type, Func<Type, object>, object> activator)
    {
        _activator = activator;
    }

    public object Activate(Type type, Func<Type, object> resolver)
    {
        return _activator(type, resolver);
    }
}
{% endhighlight %}

Yes, that is a Func that takes a Func as an argument.  This will get cleared up in a future refactoring, but for now, this is actually quite simple.  The nested Func argument is the Resolve method (give me a type, and I’ll give you back an instance).  So, that last thing to do is to refactor the Container class to use the Registration class and the Activators.

{% highlight csharp %}
public class Container
{
    private List<Registration> _registrations = new List<Registration>();

    public Registration Register(Type type)
    {
        var registration = new Registration(type);
        _registrations.Add(registration);
        return registration;
    }

    public Registration Register<T>()
    {
        return Register(typeof(T));
    }

    public object Resolve(Type type)
    {
        var registration = FindRegistration(type);
        return registration.Activator.Activate(type, Resolve);
    }

    public T Resolve<T>()
    {
        return (T)Resolve(typeof(T));
    }

    private Registration FindRegistration(Type type)
    {
        var registration = _registrations.FirstOrDefault(r => r.ConcreteType == type);
        if (registration == null)
            registration = Register(type);

        return registration;
    }
}
{% endhighlight %}

Just to note, we are allowing the resolution of types that have not been explicitly registered with the container.  I believe that some other notable IoC container authors disagree with this practice, namely [Krzysztof Koźmic](http://kozmic.pl) and [Nicolas Blumhardt](http://nblumhardt.com).  Their reasoning is that without explicit configuration, it is easy for the container to misbehave.  I say, have enough tests so that doesn’t become an issue.  Regardless, this is how it is coded and if you disagree, rip out that part and throw an exception.

One other thing to note is that we are not threadsafe. This would be relatively simple to add using the new concurrent collection types in .NET 4.0. I'll leave that to you to play around with.

Below is the new test we have added for this batch of functionality:

{% highlight csharp %}
public class when_resolving_a_type_with_unresolvable_dependencies : ContainerSpecBase
{
    static object _result;

    Establish context = () =>
        _container.Register<DummyService>()
            .ActivateWith((t, r) => new DummyService(9, (DepA)r(typeof(DepA))));

    Because of = () =>
        _result = _container.Resolve(typeof(DummyService));

    It should_not_return_null = () =>
        _result.ShouldNotBeNull();

    It should_return_an_instance_of_the_requested_type = () =>
        _result.ShouldBeOfType<DummyService>();

    private class DummyService
    {
        public DummyService(int i, DepA a)
        { }
        }

    private class DepA { }
}
{% endhighlight %}

Like I said above, a little ugly with that ActivateWith call. It is saying to use 9 for the integer and resolve DepA from the container.

In the [next post]({% post_url software/2011-06-08-building-an-ioc-container-resolving-abstractions %}), we'll get into how to handle resolving IDummyService, which is basically the whole point of this exercise. Stay tuned...