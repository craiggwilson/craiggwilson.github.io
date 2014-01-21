---
layout: post
title: MongoDB Drivers - Bson
description: Bson is the document representation for MongoDB.
tags: [mongodb, .net, drivers]
image:
  feature: abstract-1.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

> Bson (Binary Json) is the document representation for MongoDB.

In the [introduction]({% post_url software/2013-11-26-mongodb-drivers-introduction %}), I discussed the purpose of this series as well as the purpose and responsibility of a MongoDB Driver.  We'll continue the series with a look at one of the foundational aspects of MongoDB - BSON.

MongoDB uses [bson](http://bsonspec.org) as its document representation.  It is basically a binary form of [json](http://json.org/).  This makes perfect sense because json is a nice compact way of representing documents and since MongoDB is a document-oriented database, it seems like a natural fit.  However, json has some problems on its own.  For one, it's expensive to store.  Storing json in a binary format allows us to save a lot of space.  Json is also expensive to parse.  If we had to parse a document everytime we need to compare its contents to satisfy a query, that could be quite a performance hit.  All these problems were solved by creating bson.  

But using bson means that all our drivers need to understand bson.  When the .NET driver began, there were no shared libraries for .NET that knew how to read or write bson, and so we created our own.  It is represented in the namespace [MongoDB.Bson](http://api.mongodb.org/csharp/current/?topic=html/04010a3d-243e-b4e5-8b36-742a145d2fee.htm) and can be used separately from the driver for storage or messaging protocols.  All in all, implementing bson is relatively trivial.  However, there is one question that isn't exactly trivial to answer - how to represent bson to our users?

### How to Represent Bson

Some languages have types that naturally map to json structures.  For instance, Node.js doesn't really have to do much.  Python uses the built-in dict type.  .NET could represent a bson document as a Dictionary<string, object>.  Unfortunately, this fails in .NET land because of some special behaviors that exist in bson.

First, bson allows for the same key to exist multiple times.  The [json spec](http://www.ietf.org/rfc/rfc4627.txt) states that "names within an object should be unique," but this doesn't always hold even for json documents.  Since MongoDB doesn't prohibit this (although we seriously discourage it), drivers need to be able to handle this situation.  Second, bson elements are ordered and keeping the order definitely helps the database in storage optimization.  

All of this led us to create a type called a BsonDocument.  
{% highlight csharp %}
public class BsonDocument : BsonValue, 
	IComparable<BsonDocument>, 
	IEnumerable<BsonElement>, 
	IEquatable<BsonDocument>
{
	public BsonValue this[string name] { get; set; }

	public BsonValue this[int index] { get; set; }


	public bool TryGetValue(string name, out BsonValue value)
	{ 
		// ...
	}

	// etc...
}

{% endhighlight %}
The semantics of a dictionary are preserved.  You can still do value lookups using indexers or TryGetValue.  In addition, you can be order conscious and use indexes.  But the most interesting item in the above code is the BsonValue type.

#### Closed Type System

One of the biggest problems we had to overcome was the lack of round-tripability (word?) into native .NET types.  For instance, the native DateTime type in .NET has a different resolution and range than the bson date type. If we allowed you to just put an arbitrary DateTime instance into your BsonDocument, then there is a chance that data loss could occur, either when going to MongoDB or when coming from MongoDB.  In addition, all dates in MongoDB are stored as UTC time. If you put in a localized DateTime, we could definitely convert it to UTC, but we wouldn't know whether or not to convert it back to a local time on the way out.  All of this is no good.  So we have a type called a BsonDateTime which complies with the resolution of a bson date and time and is always in UTC, but is then user-convertible to a native DateTime type (with possible data loss).  It is a user's decision and we won't do it automatically unless the user has instructed us to do so.

An additional benefit of a closed type system means that we can highly optimize serialization and deserialization for these specific types.  We don't have to worry about how to convert the world, just the types that are part of our closed type system.  And for those who need every bit of performance out of their system, then using the BsonValue type system is definitely the way to go.

#### Static Languages

Another thing the .NET bson library needs to handle is static, user-constructed types.  Users choose to work in a static language for a couple of reasons; either they like static languages (me) or they are forced to by their company.  Either way, they are working in a static language where as bson/json/MongoDB has a dynamic schema.  How can we reconcile these concepts?

It's highly improbable that a number of documents stored together will vary wildly.  You wouldn't store a bunch of documents about your pets in the same place as a bunch of documents related to the measurment of your higgs-boson expiriments.  They would be stored in different places and, as such, each of the places would have what I call an application-defined schema.  In other words, documents collected together look mostly the same.  

Given this information, the .NET driver team felt it important to have a way to work with plain old csharp objects (POCOs).  And we still need them to be fast.  For instance, reading bson into a BsonDocument and then into a POCO wasn't acceptable.  Therefore, we enabled the possibility of mapping POCOs and then reading bson directly into those POCOs (and also in reverse, POCOs directly to bson).  This decision also enabled a number of other benefits we weren't fully cognizant of at the time, namely providing a foundation for supporting automatic query generation via LINQ, a feature which .NET developers rely on heavily.  For example, the .NET driver will map the below class to the shown json automatically, without the user needing to do anything.
{% highlight csharp %}
public class Person
{
	public string Name { get; set; }

	public DateTime BirthDate { get; set; }
}
{% endhighlight %}
{% highlight json %}
{
        "Name" : "Jack",
        "BirthDate" : ISODate("2000-01-01T00:00:00Z")
}
{% endhighlight %}

As far as the user is concerned, they are simply working with their POCO and never need to care that it is persisted to MongoDB or represented as bson.  The LINQ query below is translated into MongoDB natively because we have implemented the [IQueryable interface](http://msdn.microsoft.com/en-us/library/system.linq.iqueryable(v=vs.110).aspx) in the driver.

{% highlight csharp %}
var people = from person in personCollection
             where person.Name.StartsWith("J")
             select person;
{% endhighlight %}
{% highlight json %}
{ "Name": /^J*/ }
{% endhighlight %}

However, some users will care and will want to customize this experience.  Internally, a number of conventions are used and users can customize everything from what is serialized to how it is serialized either via attributes or in-code configuration.  I won't go into how this works now because our [documentation](http://docs.mongodb.org/ecosystem/tutorial/serialize-documents-with-the-csharp-driver/) is very complete on this topic.  Perhaps a future blog post on conventions and mapping can delve into the internals.

### Conclusion

Bson is the first step that needs to happen when building a driver.  Without it, a driver can't talk to MongoDB.  At this point in time, there might already be a bson library built for your language or framework, although a simple one isn't overly difficult to get up and running.  For instance, if you are writing a driver that can utilized C libraries, MongoDB makes the tremendous [libbson](https://github.com/mongodb/libbson) which has had a number of micro-optimizations to make it extremely fast and reliable.

[Next time]({% post_url software/2013-12-10-mongodb-drivers-wire-protocol-1 %}), we'll talk about the TCP and the MongoDB wire protocol.