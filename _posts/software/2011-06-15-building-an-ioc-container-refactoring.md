---
layout: post
title: Building an IoC Container - Refactoring
description: Refactoring the container.
tags: [.net, ioc, solid]
image:
  feature: abstract-6.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: true
---

The code for this step is located on [github](https://github.com/craiggwilson/presentations/tree/master/BuildYourOwnIoC/code/Stage%206%20-%20Refactor%20to%20ResolutionContext).

In the [previous post]({% post_url software/2011-06-12-building-an-ioc-container-adding-lifetimes %}), we added support for singleton and transient lifetimes. But the last couple of posts have made our syntax look a bit unwieldy and is somewhat limitting when looking towards the future, primary when needing to detect cycles in the resolution chain. So today, we are going to refactor our code by introducing a new class ResolutionContext. This will get created everytime a Resolve call is made. There isn't a lot to say without looking at the code, so below is the ResolutionContext class.

{% highlight csharp %}
public class ResolutionContext
{
    private readonly Func<Type, object> _resolver;

    public Registration Registration { get; private set; }

    public ResolutionContext(Registration registration, Func<Type, object> resolver)
    {
        Registration = registration;
        _resolver = resolver;
    }

    public object Activate()
    {
        return Registration.Activator.Activate(this);
    }

    public object GetInstance()
    {
        return Registration.Lifetime.GetInstance(this);
    }

    public object ResolveDependency(Type type)
    {
        return _resolver(type);
    }

    public T ResolveDependency()
    {
        return (T)ResolveDependency(typeof(T));
    }
}
{% endhighlight %}

Nothing in this is really that special. The Activate method and the GetInstance method are simply here to hide away the details so the caller doesn't need to dot through the Registration so much (Law of Demeter). The Funcis still here, but this is where it stops. Shown below, our IActivator and ILifetime interfaces now take a ResolutionContext instead of the delegates.

{% highlight csharp %}
public interface IActivator
{
    object Activate(ResolutionContext context);
}

public interface ILifetime
{
    object GetInstance(ResolutionContext context);
}
{% endhighlight %}

Now, they look almost exactly the same, so, as we discussed in the last post, the difference is purely semantic. Activators construct things and Lifetimes manage them.

Finally, no new tests have been added, but a number have changed due to this refactoring. I'd advise you to check out the full source and look it over yourself. In our [final post]({% post_url software/2011-06-17-building-an-ioc-container-cyclic-dependencies %}), we'll be handling cyclic dependencies now that we have an encapsulated ResolutionContext to track calls.