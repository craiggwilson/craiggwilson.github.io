---
layout: post
title: MongoDB Drivers: Bson
description: Series entry on bson.
tags: [mongodb, .net, infrastructure]
image:
  feature: abstract-1.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

> Bson (Binary Json) is the document representation for MongoDB.

MongoDB uses [bson](http://bsonspec.org) as its document representation.  It is basically a binary form of [json](http://json.org/).  This makes perfect sense because json is a nice compact way of representing documents.  Since MongoDB is a document-oriented database, seems like a natural fit.  However, json has some problems on its own.  First, it's expensive to store.  Storing json in a binary format allows us to save a lot of space.  Second, json is expensive to parse.  If we had to parse a document everytime we need to compare it for a query, that could be quite a performance hit.  All these problems were solved by creating bson.  

But using bson means that all our drivers need to understand bson.  As bson was created for the purposes of MongoDB, when the first drivers were being written, no shared libraries existed.  Therefore, the first thing a driver needs to do well is be able to read and write bson.  As of this writing, MongoDB has written a c library called [libbson](https://github.com/mongodb/libbson).  Any driver that can use c libraries underneath can take advantage of this.  A benefit of using this library is that [Christian Hergert](https://twitter.com/hergertme) has done some pretty significant micro-optimizations related to memory management and performance.  As MongoDB is also an extremly fast database, our drivers need to be able to keep up to allow user applications to get the most out of the database.  But if you need to write your own bson library, it isn't that hard to get a basic one working.  It's just a binary format, so if your language can read and write bytes from a tcp socket, then you are good to go.  

Once we can read and write bson, the real question is about how to represent bson to our users.  Some languages have types that naturally map to json structures.  For instance, Node.js doesn't really have to do much.  Python uses the built-in dict type.  .NET could represent a bson document as a Dictionary<string, object>.  But each of these fail just a little bit.  Certain behaviors exist in json and others exist in MongoDB.  

#### Json/Bson Behaviors

First, bson allows for the same key to exist multiple times.  The [json spec](http://www.ietf.org/rfc/rfc4627.txt) states that "names within an object should be unique," but this doesn't always hold even for json documents.  Now, this is stupid, but it is possible and MongoDB doesn't prohibit it which means that drivers need to be able to handle this situation.  Second, bson elements are ordered and keeping the order definitely aids the database in storage optimization.  Given these examples, the .NET driver chose to create a new type called a BsonDocument that allow some of these esoteric behaviors. In some cases, there are settings.  For instance, it has a setting which allows for duplicate keys to exist.

#### Dates

Another interesting thing the .NET driver needs to deal with in relation to bson is dates.  The native DateTime type in .NET has a different resolution than that which can be represented in bson.  As such, round-tripping a date from bson -> DateTime -> bson can cause a loss of data.  This is no good.  So we have a type called a BsonDateTime which complies with the resolution of a bson date and time, but is then user-convertible to a native DateTime type with possible data loss.  It is a users decision and we won't do it automatically unless the user has instructed us to do so.

#### Static Languages

One last thing to point out is that users choose to work in a static language for a couple of reasons; either they like static languages (me) or they are forced to by their company.  Either way, they are working in a static language where as bson/json/MongoDB has a dynamic schema.  How can we reconcile these concepts?

Well, it's highly improbable that a number of documents stored together will vary wildly.  You wouldn't store a bunch of documents about your pets in the same place as a bunch of documents related to the measurment of your higgs-boson expiriments.  They would be stored in different places and, as such, each of the places would have what I call an application-defined schema.  In other words, documents collected together look mostly the same.  

Given this information, the .NET driver team felt it important to have a way to work with plain old csharp objects (POCOs).  And we still need them to be fast.  For instance, reading bson into a BsonDocument and then into a POCO wasn't acceptable.  Therefore, we enabled to possibility of mapping POCOs and then reading bson directly into those POCOs (and also in reverse, POCOs directly to bson).  I could write a number of posts on this subject (and maybe I will), but not as part of this series.  This decision enabled a number of other benefits, namely a foundation for supporting automatic query generation via LINQ, a feature which .NET developers rely on heavily.

#### Conclusion

This concludes this post on the role of bson in a driver.  It is basically the first step that needs to happen.  At this point, there might already be a bson library build for your language or framework, although a simple one isn't too difficult to get up and running.

Next time, we'll talk about the TCP and the MongoDB wire protocol.