---
layout: post
title: MongoDB Drivers - Wire Protocol Part 1
description: How drivers talk to MongoDB using the Wire Protocol - Part 1.
tags: [mongodb, .net, drivers]
image:
  feature: abstract-1.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

> MongoDB Wire Protocol Part 1

[Last time]({% post_url software/2013-11-28-mongodb-drivers-wire-protocol-1 %}), we began discussing the [MongoDB Wire Protocol]() with a look at how [BSON](http://bsonspec.org) ties in and some restrictions imposed by MongoDB.  In this post, we'll continue that discussion with a look at querying.

As mentioned previously, there are 7 different types of messages used by drivers.  4 of these messages relate to querying; OP_QUERY, OP_REPLY, OP_GETMORE, and OP_KILLCURSORS.  Let's go through the lifecycle of a query.

The first message we send to the server is an OP_QUERY.

{% highlight c %}
struct OP_QUERY {
    MsgHeader header;                 // standard message header
    int32     flags;                  // bit vector of query options.  See below for details.
    cstring   fullCollectionName ;    // "dbname.collectionname"
    int32     numberToSkip;           // number of documents to skip
    int32     numberToReturn;         // number of documents to return
                                      //  in the first OP_REPLY batch
    document  query;                  // query object.  See below for details.
  [ document  returnFieldsSelector; ] // Optional. Selector indicating the fields
                                      //  to return.  See below for details.
}
{% endhighlight %}

An OP_QUERY message will always have a reply of type OP_REPLY.
{% highlight c %}
struct {
    MsgHeader header;         // standard message header
    int32     responseFlags;  // bit vector - see details below
    int64     cursorID;       // cursor id if client needs to do get more's
    int32     startingFrom;   // where in the cursor this reply is starting
    int32     numberReturned; // number of documents in the reply
    document* documents;      // documents
}
{% endhighlight %}

As OP_REPLY is also under the maximum message size restriction, then it goes to reason that if a query returns more results than can fit inside the reply message, there would need to be away to get the next batch.  The cursorID field is present so that the driver can then go back to the same server on the same or a different TCP connection and ask for the next batch.  It does this by sending the OP_GETMORE message.

{% highlight c %}
struct {
    MsgHeader header;             // standard message header
    int32     ZERO;               // 0 - reserved for future use
    cstring   fullCollectionName; // "dbname.collectionname"
    int32     numberToReturn;     // number of documents to return
    int64     cursorID;           // cursorID from the OP_REPLY
}
{% endhighlight %}

The OP_GETMORE will also come back with an OP_REPLY and is handled the same way a response from an OP_QUERY message would be.  This will continue until either the cursorID equals 0, meaning no more results, or until the user decides they don't want any more results.  If the user terminates while a cursorID is non-zero, then it is up to the driver to send an OP_KILLCURSORS message to tell the server it can release the resources it is using the maintain the cursor.

{% highlight c %}
struct {
    MsgHeader header;            // standard message header
    int32     ZERO;              // 0 - reserved for future use
    int32     numberOfCursorIDs; // number of cursorIDs in message
    int64*    cursorIDs;         // sequence of cursorIDs to close
}
{% endhighlight %}

In the .NET driver, this is managed by using a custom class implementing built-in `IEnumerator<T>` class.  The great thing about `IEnumerator<T>` is that it implements `IDisposable`.  The C#/VB compilers handle calling `Dispose()` once the foreach loop has been terminated/abandoned.  In addition, the LINQ methods also handle this correctly.  Therefore, it is very natural and simple to expose these semantics to our users in a natural way.

### Update/Delete

### Insert

### Write Commands