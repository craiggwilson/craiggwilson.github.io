---
layout: post
title: MongoDB Drivers: Introduction
description: Series introduction on building a MongoDB Driver.
tags: [mongodb, .net, infrastructure]
image:
  feature: abstract-10.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: true
---

> Series introduction on building a MongoDB Driver.

Building a driver for [MongoDB](http://mongodb.org) encompasses a wide variety of topics and deals with quite a few different areas of programming.  I'd like to walk through the basics of building a driver in MongoDB over a series of posts.  We'll start with [BSON](http:bsonspec.org) and go until I run out of things to talk about.  Hopefully, you'll discover that building a simple driver isn't all that difficult.  Of course, the devil is in the details and I'll attempt to point out some of these details as I go although I won't always discuss solutions.  In addition, I'll attempt to point out differences between drivers that are async vs. sync, single-threads vs. multi-threaded although, dynamic vs. static, etc...

To start off, I'll take a little time to discuss the purpose of a MongoDB driver as well as our [existing drivers](http://docs.mongodb.org/ecosystem/drivers/).

There are really 2 components to MongoDB, the server and the driver.  The server can take a number of forms; a [standlone server](http://docs.mongodb.org/manual/reference/glossary/#term-standalone), a [replica set member](http://docs.mongodb.org/manual/core/replica-set-members/), a [config server](http://docs.mongodb.org/manual/core/sharded-cluster-config-servers/), or a [mongos](http://docs.mongodb.org/manual/reference/program/mongos/#bin.mongos).  With the exception of the config server, a driver is responsible for communicating effectively with each of these different variations of a server.  

Just as importantly, a driver is also responsible for making an API that is idiomatic to its target user base.  In other words, the .NET driver should look like a .NET library, not a python module.  There are most certainly concepts, algorithms, and high-level API decisions that are shared across our many drivers, but each driver distinctively looks likes native libraries to its target user base.  I believe this is one of the reasons users love MongoDB so much; working with MongoDB feels like working with everything else in their framework of choice.  But going this route presents some other challenges.  Instead of building a single interface for all languages over HTTP (for instance), we need to deal with the idiosyncrasies of each language/framework.

For instance, drivers need to be able to talk to multiple versions of each of the 4 variations of a MongoDB server.  So, given the 4 forms of server and support for the current server version as well as the prior 2 major releases [^1], each driver supports roughly 12 form/version.

But that isn't all.  Each version, as can be expected, supports different and sometimes additional configurations.  For instance, server 2.4 introduced support for [kerberos authentication](http://docs.mongodb.org/manual/tutorial/control-access-to-mongodb-with-kerberos-authentication/).  This means that most of our drivers were refactored to support a brand-new authentication mechanism which isn't supported in the previous versions.

But that isn't all.  Many of drivers support multiple platforms.  Our ruby driver, for instance, runs on both linux and windows. But in addition to that, ruby supports C-extensions if those are installed.  So, a user with the mongodb-c-extensions installed will occasionally run very different (and faster) code than a user without them.  In this case, the ruby driver team has written certain things twice in order to support both use cases.  What's more is that there are different ruby runtimes.  The basic one is MRI, but JRuby is also very popular and the differences have to be tested and are sometimes optimized for in code.  Not to pile more on to this list, but ruby has a number of different versions out there all on its own.  Factor all this in, you can get quite a large testing matrix.  Complements to the ruby driver engineers for keeping this manageable.  

But that isn't all.  Actually, it is, for now.  My point is that while I discuss the basics of a MongoDB driver, I'm not going to go down each and every rabbit hole.  I'll update the below list as new articles are published.  Please ask questions as we are going and I'll do my best to answer them.  I build the .NET driver, so my expertise lies definitively in that arena.

[Introduction]({% post_url software/2013-10-28-mongodb-drivers-introduction %})


[^1]: At this time of writing, server 2.4 is the current version. We also support the 2.2 and 2.0 lines.  As each major version is release roughly every 6 months, this amounts to around 18 months of support.