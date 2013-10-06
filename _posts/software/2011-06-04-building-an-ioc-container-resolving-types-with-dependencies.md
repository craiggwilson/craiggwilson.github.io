---
layout: post
title: Building an IoC Container - Resolving Types with Dependencies
description: How to resolve types with dependencies.
tags: [.net, ioc, solid]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: true
---

The code for this step is located on [github](https://github.com/craiggwilson/presentations/tree/master/BuildYourOwnIoC/code/Stage%202%20-%20With%20Dependencies).

In the [previous post]({% post_url software/2011-06-04-building-an-ioc-container-resolving-types-with-dependencies %}), we created our first test and our container implementation.  It is very simple and only resolves types with default constructors. In the installment, we’ll take this a step further and resolve types by discovering it’s constructors parameters and resolving them as well.

But first, let’s write a test to show us exactly what we are trying to do.

{% highlight csharp %}
public class when_resolving_a_type_with_dependencies : ContainerSpecBase
{
    static DummyService _result;

    Because of = () =>
        _result = (DummyService)_container.Resolve(typeof(DummyService));

    It should_not_return_null = () =>
        _result.ShouldNotBeNull();

    It should_return_an_instance_of_the_requested_type = () =>
        _result.ShouldBeOfType<DummyService>();

    It should_return_an_instance_with_the_dependency_created = () =>
        _result.A.ShouldNotBeNull();

    private class DummyService
    {
        public DepA A { get; private set; }

        public DummyService(DepA a)
        {
            A = a;
        }
    }

    private class DepA { }
}
{% endhighlight %}

As you can see, when constructing DummyService, we must provide it with an instance of DepA.  Currently, this test will fail because our current implementation assumes a default constructor exists.  Let’s remove that assumption.

{% highlight csharp %}
public class Container
{
    public object Resolve(Type type)
    {
        var ctor = type.GetConstructors()
            .OrderByDescending(c => c.GetParameters().Length)
            .First();

        var parameters = ctor.GetParameters();
        var args = new object[parameters.Length];
        for (int i = 0; i < args.Length; i++)
            args[i] = Resolve(parameters[i].ParameterType);

        return ctor.Invoke(args);
    }

    public T Resolve<T>()
    {
        return (T)Resolve(typeof(T));
    }
}
{% endhighlight %}

Simpler than you thought, huh?  Let’s walk through this real quick.  First, we get the constructor that has the most parameters.  If we wanted, this could be published as an extensibility point to allow a consumer to alter the strategy of how to select a constructor.  However, this fits our needs just fine.  After this, we simply iterate through all the parameters of the constructor and recursively call Resolve for each and store the result in the args array.  Finally, we invoke the constructor with all the args.

In the [next post]({% post_url software/2011-06-06-building-an-ioc-container-resolving-types-with-unresolvable-dependencies %}), we’ll discuss how to deal with types that can’t be resolved using the above method.