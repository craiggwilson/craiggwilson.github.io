---
layout: post
title: Disconnecting from MongoDB
description: Calling Disconnect in the MongoDB .NET Driver is probably the wrong decision.
tags: [mongodb, .net, best-practice, api]
image:
  feature: abstract-10.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

> Calling Disconnect in the MongoDB .NET Driver is probably the wrong decision.

It's kinda funny.  Developers see something called Disconnect and just assume they should call it when they are done.  Maybe that's reasonable.  If so, we made a mistake by including it in the driver.  What's more is that good API design should make it [easy to do the right thing and difficult to do the wrong thing](http://www.codinghorror.com/blog/2007/08/falling-into-the-pit-of-success.html).

Imagine a [Nancy](http://nancyfx.org/) module:
{% highlight csharp %}
public class AuthorsModule : NancyModule
{
    public AuthorsModule()
    {
        Get["/"] = _ => 
       	{
       	    var client = new MongoClient(connectionString);
       	    var server = client.GetServer();
       	    server.Connect();
       	    var library = server.GetDatabase("library");
       	    var authors = library.GetCollection("authors");

       	    var allAuthors = authors.FindAll();

       	    ///
       	    server.Disconnect();
       	    ////

       	    return allAuthors.ToList();
        };
    }
}
{% endhighlight %}

Believe it or not, we see this type of code a lot (not generally by people using Nancy); lots of code duplication, lack of abstraction.  Anyways, as it is, the above will be detrimental to a site's performance and cause all kinds of errors that are occasionally difficult to diagnose.  Disconnect will cause the connection pool to be emptied and all current in flight operations to be cut off.  This may not seem that bad until you realize that all clients using the same connection string are sharing a connection pool. Boom! [^1]

API design is something we work really hard at in building the drivers at MongoDB.  Lots of time is spent going over options. When we are adding new features, all of the drivers are generally doing it concurrently, so decisions generally need to be applicable across all the drivers we maintain, from .NET and Java to Ruby and Python to Node.js and Erlang.  Obviously, finding commonality in these is sometimes difficult and our over-arching goal is that each driver should provide something idiomatic to the target audience, something that feels natural.  So while the code above may feel natural and idiomatic (and perhaps even obvious), it definitely misses on the [Principle of Least Astonishment](http://en.wikipedia.org/wiki/Principle_of_least_astonishment).

We have begun working on the next major revision of the driver.  In fact, all the drivers supported by [MongoDB](http://www.mongodb.com) are working on their next major revisions.  We are all working through similar issues and trying to unify our designs.  The .NET driver is going to try to maintain backwards compatibility (we have a great mechanism to do so [^2]) but there are certain things where I think backwards compatibility may be a problem.  Should Disconnect simply be removed?  I really can't think of any good reason for it to exist.

[^1]: Even Connect is unnecessary, as the driver will connect automatically when the first operation is conducted.
[^2]: You'll notice in the code above that we have a client and then a server.  We're going to leave the GetServer() method for backwards compatibility, but we'll add a new GetDatabase method to MongoClient, allowing us to unify it with the other drivers.