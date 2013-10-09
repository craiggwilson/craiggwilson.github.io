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

MongoDB is getting a new write api to support bulk writes.  Currently, we support bulk inserts, but the request has come in that users want bulk updates and bulk removes.  I don't mean things like removing a bunch of documents based on a query.  Rather, I'm talking about specifying a bunch of individul removes and executing them in bulk.

The server supports the 3 different types of operations (insert, update, remove) and each of these new commands contain all of one type.  So, if you want to do a bunch of updates and removes, then that will be 2 different commands.  However, I don't think that is how it will be in the far future.  It's likely we'll be able to interleave different types of write operations and let the server execute those in bulk.

So we as a drivers team are going to make the client API appear as if this was already the case.  Developers will specify all the operations they want to do and we'll handle figuring out how to split them up and send them to the server.  If you think about it, this isn't all that difficult and really comes down to chunking an IEnumerable<WriteRequest> into it's sub-batches.  It gets tricky when we want to be efficient with memory because the entire list could be very large.

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

Ok, so, how do we chunk these together?  We are going to create an extension method called Chunk that will take a Func...