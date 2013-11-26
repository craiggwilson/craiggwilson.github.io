
---
layout: post
title: MongoDB Drivers - Wire Protocol
description: How drivers talk to MongoDB using the Wire Protocol.
tags: [mongodb, .net, drivers]
image:
  feature: abstract-1.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

> MongoDB Wire Protocol

[Last time]({% post_url software/2013-11-28-mongodb-drivers-bson %}), we discussed [BSON](http://bsonspec.org) and how it relates to drivers.  In this post, we'll continue our driver building discussion with how that BSON gets transferred over the wire using the MongoDB Wire Protocol.

The [MongoDB Wire Protocol](http://docs.mongodb.org/meta-driver/latest/legacy/mongodb-wire-protocol/) is a binary protocol communicated over TCP/IP.  It doesn't really need to use TCP/IP per se, but TCP is the reliable transport protocol that everyone already understands.  Almost every language/framework has a way to use TCP and, as such, was a rather obvious choice.  There are a couple of optimizations that have been made by most drivers using TCP, such as packing multiple messages together and disabling [Nagle](http://en.wikipedia.org/wiki/Nagle's_algorithm).  For the most part though, using TCP is straightforward.

However, the wire protocol is a slightly different story.  It is composed of messages, 8 in total but only 7 used by the driver (OP_MSG is the guy left out).  Each has a common header and then appends extra data on depending on what needs to be done.  For the most part, it looks rather trivial.  Until we get into the specifics.  

### The Bson Overlap

As [previously discussed]({% post_url software/2013-11-28-mongodb-drivers-bson %}), bson plays an important role.  But because of how Bson works, we are a little constrained in how we can write to a TCP stream.  Specifically, Bson encodes the size of a document *before* the document itself.  It's a chicken and egg problem.  We need to write the document to the TCP stream, but until we do that, we don't know the size of the document.

Therefore, each document must be written to a buffer first to get its size, and then rewritten to the TCP stream starting with the discovered size.  In .NET, this could mean converting a POCO (plain old csharp object) into bytes twice.  Obviously, this could be extremely expensive, so we don't do it this way.  Instead, still using a buffer, we write a placeholder for length, write the document, and then go back and backpatch the length.  Therefore, the only real overhead here is that we now have the document in memory twice, once as the POCO itself and once as it's rendered byte array.  In other words, to send a 10MB document to the server requires 20MB of memory.

Unfortunately, there isn't much we can do about this.  We can alleviate some of the pressure on the GC by reusing the buffers used to temporarily serialize our documents.  The .NET driver has pinned byte buffers set aside so that the GC is mostly unaffected because we aren't constantly allocating and releasing memory.  This is a common pattern called [object pooling](http://en.wikipedia.org/wiki/Object_pool_pattern), only we are doing it with bytes instead of objects.

### MongoDB Imposed Limitations

MongoDB also imposes some requirements on the drivers in the form of maximum sizes.  Derived from the [isMaster command](http://docs.mongodb.org/manual/reference/command/isMaster/), MongoDB communicates both a maximum bson document size (maxBsonObjSize) as well as a maximum message size (maxMessageSizeBytes).  If the server receives a message larger than the maximum message size or a document larger than the maximum document size, it will reject the message.  

Technically, a driver doesn't need to care about this and can let the server handle abnormalities.  But this isn't a great experience for users.  It would be annoying for a user to find out that they just sent a 48MB message over the wire only to find out it was rejected by the server.  Rather, it is much better if the driver keeps track of this information and rejects messages or documents before sending them.  

### Querying

### Update/Delete

### Insert

### Write Commands