---
layout: post
title: MongoDB Drivers - Bson
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

In the [introduction]({% post_url software/2013-11-04-mongodb-drivers-introduction %}), I discussed the purpose of this series.  I'm going to kick-off the series with a look at one of the foundational aspects of MongoDB - bson.

MongoDB uses [bson](http://bsonspec.org) as its document representation.  It is basically a binary form of [json](http://json.org/).  This makes perfect sense because json is a nice compact way of representing documents and since MongoDB is a document-oriented database, it seems like a natural fit.  However, json has some problems on its own.  For one, it's expensive to store.  Storing json in a binary format allows us to save a lot of space.  Json is also expensive to parse.  If we had to parse a document everytime we need to compare it for a query, that could be quite a performance hit.  All these problems were solved by creating bson.  

But using bson means that all our drivers need to understand bson.  When the .NET driver began, there were no shared libraries for .NET that knew how to read or write bson, and so we created our own.  It is represented in the namespace [MongoDB.Bson](http://api.mongodb.org/csharp/current/?topic=html/04010a3d-243e-b4e5-8b36-742a145d2fee.htm) and can be used separately from the driver for storage or messaging protocols.  All in all, implementing a binary storage format is relatively trivial.  However, there is one question that isn't exactly trivial to answer - how to represent bson to our users?

### How to Represent Bson

Some languages have types that naturally map to json structures.  For instance, Node.js doesn't really have to do much.  Python uses the built-in dict type.  .NET could represent a bson document as a Dictionary<string, object>.  Unfortunately, this fails in .NET land because of some special behaviors that exist in bson.

First, bson allows for the same key to exist multiple times.  The [json spec](http://www.ietf.org/rfc/rfc4627.txt) states that "names within an object should be unique," but this doesn't always hold even for json documents.  Since MongoDB doesn't prohibit this (although we seriously discourage it), drivers need to be able to handle this situation.  Second, bson elements are ordered and keeping the order definitely aids the database in storage optimization.  

#### Closed Type System

All of this led us to create a type called a BsonDocument.  It however, does not implemented IDictionary<string, object>, but rather IEnumerable<BsonElement>.  Why?  There are a lot of reasons, but the main one is that we wanted a closed type system. Why?  Let's take an example of something that might seem trivial, Dates.

The native DateTime type in .NET has a different resolution than that which can be represented in bson.  So, if we allowed you to just put an arbitrary DateTime instance into your BsonDocument, then there is a good chance that data loss could occur. This is no good.  So we have a type called a BsonDateTime which complies with the resolution of a bson date and time, but is then user-convertible to a native DateTime type (with possible data loss).  It is a user's decision and we won't do it automatically unless the user has instructed us to do so.

An additional benefit of a closed type system means that we can highly optimize serialization and deserialization.  We don't have to worry about how to convert the world, just the types that are part of our closed type system.

#### Static Languages

Users choose to work in a static language for a couple of reasons; either they like static languages (me) or they are forced to by their company.  Either way, they are working in a static language where as bson/json/MongoDB has a dynamic schema.  How can we reconcile these concepts?

Well, it's highly improbable that a number of documents stored together will vary wildly.  You wouldn't store a bunch of documents about your pets in the same place as a bunch of documents related to the measurment of your higgs-boson expiriments.  They would be stored in different places and, as such, each of the places would have what I call an application-defined schema.  In other words, documents collected together look mostly the same.  

Given this information, the .NET driver team felt it important to have a way to work with plain old csharp objects (POCOs).  And we still need them to be fast.  For instance, reading bson into a BsonDocument and then into a POCO wasn't acceptable.  Therefore, we enabled to possibility of mapping POCOs and then reading bson directly into those POCOs (and also in reverse, POCOs directly to bson).  I could write a number of posts on this subject (and maybe I will), but not as part of this series.  This decision enabled a number of other benefits, namely a foundation for supporting automatic query generation via LINQ, a feature which .NET developers rely on heavily.

### Conclusion

This concludes this post on the role of bson in a driver.  It is basically the first step that needs to happen.  At this point, there might already be a bson library build for your language or framework, although a simple one isn't too difficult to get up and running.

Next time, we'll talk about the TCP and the MongoDB wire protocol.