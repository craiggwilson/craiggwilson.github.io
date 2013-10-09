---
layout: post
title: Chunking in LINQ
description: Splitting an IEnumerable into sub sequences.
tags: [mongodb, .net, linq, api]
image:
  feature: abstract-10.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: false
---

MongoDB 2.6 is getting a new write api to support bulk writes.  Currently, we support bulk inserts, but the request has come in that users want bulk updates and bulk removes.  I don't mean things like removing a bunch of documents based on a single query though.  Rather, I'm talking about specifying a bunch of individul removes and executing them in bulk.

The server supports the 3 different types of operations (insert, update, remove) with 3 new commands, each containing 1 or more operations to perform.  So, if you want to do a bunch of updates and removes, then that will be 2 different commands.  I don't think that is how it will be in the far future.  It's likely we'll be able to interleave different types of write operations and let the server execute those in bulk.

Therefore, the drivers team (of which I am a part) are planning to make the client API appear as if this was already the case.  Developers will specify all the operations they want to do in 1 list and we'll handle figuring out how to split them up and send them to the server.  If you think about it, this isn't all that difficult and really comes down to chunking an IEnumerable<WriteRequest> into it's sub-batches.  It gets tricky when we want to be efficient because the entire list could be very large.

As an example, let's consider the following:

{% highlight csharp %}

var list = new List<WriteRequest>
{
    new InsertRequest(new BsonDocument("x", 1)),
    new InsertRequest(new BsonDocument("x", 2)),
    new UpdateRequest(new BsonDocument("x", 1), new BsonDocument("x" 3")),
    new InsertRequest(new BsonDocument("x", 1)),
    new RemoveRequest(new BsonDocument("x", 2)
};

{% endhighlight %}

This should result in 4 batches sent to the server.  The first 2 inserts followed by an update, an insert, and a remove.  The end result should be, assuming we started with an empty collection, 1 document { x : 1 } and 1 document { x : 3 }.

Ok, so, how do we chunk these together?  In plain english, you just start taking a document at a time until you pull one off of a different type. In code, it looks like the below:

{% highlight csharp %}

IEnumerable<List<T>> BatchByType(IEnumerable<T> list)
{
    var enumerator = list.GetEnumerator();
    try
    {
        Type currentType;
        List<T> same = new List<T>();
        while(enumerator.MoveNext())
        {
            var type = enumerator.Current.GetType();
            if(currentType == null)
            {
                currentType = type;
            }
            else if(currentType != type)
            {
                yield return list;
                list = new List<T>();
                currentType = type;
            }

            list.Add(enumerator.Current);
            }
        }
        finally
        {
            enumerator.Dispose();
        }
    }
}
{% endhighlight %}

So, this works quite well.  However, my collegue believes we can do better for two reasons.  First, we are going to iterate every item twice.  For instance, we are going to iterate it to build up the list and then we'll be iterating each list to build up the commands.  Second, we are creating a new list for every group. I think both of these are just fine because the average/most common bulk operation isn't going to be massive and neither one of these problems is going to cause that much (measurable?) of a speed slowdown, especially when we factor in the latency of the network.

But, if we can do it without an increase in complexity, then fine.  Below is the outcome of this:

{% highlight csharp %}
private IEnumerable<Run<T>> FindRuns(IEnumerable<T> requests)
{
    using (var enumerator = requests.GetEnumerator())
    {
        if (enumerator.MoveNext())
        {
            var firstElement = enumerator.Current;
            while (firstElement != null)
            {
                var run = new Run(firstElement, enumerator);
                yield return run;
                firstElement = run.FirstElementOfNextRun;
            }
        }
    }
}

 // nested classes
private class Run<T> 
{
    private readonly IEnumerator<ModifyRequest> _enumerator;
    private readonly Type _type;
    private ModifyRequest _firstElement;
    private ModifyRequest _firstElementOfNextRun;
    private bool _hasBeenEnumerated;

    public Run(ModifyRequest firstElement, IEnumerator<T> enumerator)
    {
        _type = firstElement.GetType();
        _firstElement = firstElement;
        _enumerator = enumerator;
    }

    public ModifyRequest FirstElementOfNextRun
    {
        get { return _firstElementOfNextRun; }
    }

    public IEnumerable<T> EnumerateRequests()
    {
        if (_hasBeenEnumerated)
        {
            throw new InvalidOperationException("EnumerateRequests can only be called once.");
        }
        _hasBeenEnumerated = true;

        yield return _firstElement;
        while (_enumerator.MoveNext())
        {
            var element = _enumerator.Current;
            if (element.GetType() == _type)
            {
                yield return element;
            }
            else
            {
                _firstElementOfNextRun = element;
                yield break;
            }
        }
    }
}
{% endhighlight %}

This is obviously a private implementation because generalizing this for a wider audience is not trivial.  If you'd like to try, please do so and let me know.  Basically, the above is only going to iterate each item once.  I think this is a relative increase in complexity, but not so much that I'm adamantly against it.