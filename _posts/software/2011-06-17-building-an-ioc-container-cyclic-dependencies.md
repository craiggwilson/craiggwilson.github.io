---
layout: post
title: Building an IoC Container - Cyclic Dependencies
description: Support for cyclic dependencies.
tags: [.net, ioc, solid]
image:
  feature: abstract-8.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: true
---

The code for this step is located on [github](https://github.com/craiggwilson/presentations/tree/master/BuildYourOwnIoC/code/Stage%207%20-%20Cyclic%20Dependencies).

We [just finished]({% post_url software/2011-06-15-building-an-ioc-container-refactoring %}) doing a small refactoring to introduce a ResolutionContext class.  This refactoring was necessary to allow us to handle cyclic dependencies.  Below is a test that will fail right now with a StackOverflowException because the resolver is going in circles.

{% highlight csharp %}
public class when_resolving_a_type_with_cyclice_dependencies : ContainerSpecBase
{
    static Exception _ex;

    Because of = () =>
        _ex = Catch.Exception(() => _container.Resolve(typeof(DummyService)));

    It should_throw_an_exception = () =>
        _ex.ShouldNotBeNull();

    private class DummyService
    {
        public DummyService(DepA a)
        { }
    }

    private class DepA
    {
        public DepA(DummyService s)
        { }
    }
}
{% endhighlight %}

Technically, a StackOverflowException would get thrown and caught. However, this type of exception is going to take out the whole app and the test runner won't be able to complete.  Regardless,  it shouldn't take a minute to take out the heap and this should fail almost instaneously.

With a slight modification to our ResolutionContext class, we can track whether or not a cycle exists in the resoluation chain and abort early.  There are two methods that need to be modified.

{% highlight csharp %}
public object ResolveDependency(Type type)
{
    var registration = _registrationFinder(type);
    var context = new ResolutionContext(registration, _registrationFinder);
    context.SetParent(this);
    return context.GetInstance();
}

private void SetParent(ResolutionContext parent)
{
    _parent = parent;
    while (parent != null)
    {
        if (ReferenceEquals(Registration, parent.Registration))
            throw new Exception("Cycles found");

        parent = parent._parent;
    }
}
{% endhighlight %}

We begin by allowing the ResolutionContext to track it's parent resolution context.  This, as you can see in the SetParent method, will allow us to check each parent and see if we have tried to resolve a given type already.  Other than that, nothing special is going on and everything else still works correctly.

At this point, we are at the end of the [Building an IoC Container series]({% post_url software/2011-06-01-building-an-ioc-container-intro %}).  I hope you have learned a little more about how the internals of your favorite containers work and, even more so, that there isn't a lot of magic going on.  This is something you can explain to your peers or mentees and hopefully allow the use of IoC to gain an acceptance in area that has once been off-limits because it was a "black-box".  Be sure to leave me a comment if you have any questions or anything else you'd like to have done to our little IoC container.