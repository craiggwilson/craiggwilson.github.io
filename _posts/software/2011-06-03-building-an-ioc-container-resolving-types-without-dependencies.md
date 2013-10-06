---
layout: post
title: Building an IoC Container - Resolving Types without Dependencies
description: How to resolve types without any dependencies.
tags: [.net, ioc, solid]
image:
  feature: abstract-2.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: true
---

The code for this step is located on [github](https://github.com/craiggwilson/presentations/tree/master/BuildYourOwnIoC/code/Stage%201%20-%20No%20Dependencies).

As noted in the [introduction]({% post_url software/2011-06-01-building-an-ioc-container-intro %}), we are going to take this in steps and go in a test first manner.  So, let’s write our first test.

{% highlight csharp %}
public class when_resolving_a_type_with_zero_dependencies : ContainerSpecBase
{
    static object _result;

    Because of = () =>
        _result = _container.Resolve(typeof(DummyService));

    It should_not_return_null = () =>
        _result.ShouldNotBeNull();

    It should_return_an_instance_of_the_requested_type = () =>
        _result.ShouldBeOfType();

    private class DummyService { }
}
{% endhighlight %}

If you haven’t used [Machine.Specifications](https://github.com/machine/machine.specifications), this may seem like odd syntax.  But what I like about it is that it is easy to see what is going on when the container is resolving a type with zero dependencies.  First,  the container shouldn’t return null and second, the container should return an instance of the requested type.

Now that we have a test, we are going to start coding the implementation.  We’ll start with the simplest thing that could possibly work here.

{% highlight csharp %}
public class Container
{
    public object Resolve(Type type)
    {
        return Activator.CreateInstance(type);
    }

    public T Resolve()
    {
        return (T)Resolve(typeof(T));
    }
}
{% endhighlight %}

Nothing special, and nothing that you couldn’t have figured out on your own.  In the [next post]({% post_url software/2011-06-04-building-an-ioc-container-resolving-types-with-dependencies %}), we’ll see how to handle this when the requested type has dependencies.