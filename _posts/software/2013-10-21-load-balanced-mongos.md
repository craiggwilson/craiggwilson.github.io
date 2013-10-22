---
layout: post
title: Load Balancing Mongos
description: "Care must be taken when putting a load balancer in front of a bunch of mongos."
tags: [mongodb, .net, talesfromsupport, infrastructure]
image:
  feature: abstract-8.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

> Care must be taken when putting a load balancer in front of a bunch of [mongos](http://docs.mongodb.org/master/reference/program/mongos/)'.

A common desire when using mongos' is to place a load balancer between an application and the mongos'.  There are a number of reasons for doing so, many of which are perfectly fine.  One example would be acting as a router if it can be configured to have sticky behavior where a single client is always routed to the same server.  Another example would be if the load balancer was configured to facilitate failover between a couple of mongos'.  These two examples have a common theme where connections from a single client always go to the same mongos.

For the rest of this post, I'm going to explore the problem with a load balancer that is actually doing load balancing, i.e., every new connection from a client gets routed to the mongos with the least load or is used in a round-robin fashion.

The outcome of a load balancer configured this way is that queries may work initially and then stop working even though more data is available.  In the .NET driver, this would manifest itself with an exception message "Cursor not found".  Before I go into the reasons for this behavior, some background is required.

### Background

First, a [mongos](http://docs.mongodb.org/master/reference/program/mongos/) acts like a router in front of a partitioned system.  A driver, the name for mongodb's client, will connect to a mongos and let it handle which partition to route a CRUD operation.  This works extremely well.  MongoDB has a number of [different drivers](http://docs.mongodb.org/ecosystem/drivers/) for different languages and frameworks. 

Second, for queries whose results are larger than a single batch, a cursor is created.  It is basically a correlating identifier for a number of batches in a driver.  A driver issues a query ([OP_QUERY](http://docs.mongodb.org/meta-driver/latest/legacy/mongodb-wire-protocol/#op-query)) and receives the first batch of results.  If the response from the OP_QUERY included a cursor id, it means there are more results and a driver can issue an [OP_GET_MORE](http://docs.mongodb.org/meta-driver/latest/legacy/mongodb-wire-protocol/#op-get-more) to retrieve the next batch.  The only requirement here is that the OP_GET_MORE be issued to the same server that received it initially. 

### Reason

So, what is the problem?  Well, given that a load balancer has been placed in front of a bunch of different servers, and that mongodb requires OP_GET_MOREs to be sent to the same server as the initial OP_QUERY, the load balancer would be required to understand the MongoDB wire protocol to guarantee this behavior.  The problem is, of course, that load balancers don't understand the MongoDB wire protocol.  So, simply put, load balancing mongos' simply won't work, right?

Well, it's a little trickier than that because each of our many drivers was coded a little differently.  They have different authors, different needs, and most started outside of [MongoDB, Inc](http://mongodb.com) as open-source projects.  Let's take a few examples:

#### PHP

The PHP driver will always send OP_GET_MOREs down the same connection it used for the initial OP_QUERY.  If the connection dies, then even though the cursor could continue to be used by a different connection, it considers the query dead.

#### Node.js

The Node driver uses (by default) 5 connections period.  It also pins the connection it uses for the initial OP_QUERY to the cursor such that each successive OP_GET_MORE always uses the same connection.  This would mean all OP_GET_MOREs always go to the same server.  However, there's always a gotcha.  If the connection used for the initial OP_QUERY happens to die, then the node driver will replace it with a different, new connection.  There is no guarantee that this connection will be for the same server.  While this is a somewhat exceptional circumstance, it holds that the node driver can't be guaranteed to work with a load balancer.

#### Java

Many of the drivers copied our Java driver's modus operandi which uses thread affinity.  This means that under normal operation conditions, each thread is bound to a connection.  As such, each OP_QUERY and OP_GET_MORE will most likely use the same connection and therefore the same mongos behind the load balancer.  However, under heavy loads where the number of threads is greater than the maximum size of the connection pool, a thread without a connection may steal one from a different thread.  This, of course, has a nice ripple effect where threads are all stealing from each other and there is no guarantee anywhere.

#### .NET

The .NET driver doesn't do this thread affinity thing unless requested.  Instead, everytime a connection is needed, it pulls an available one out of the connection pool.  This gives a number of benefits, the greatest being that less connections are needed because each is utilized more frequently.  The downside, of course, is that it doesn't work with load balancers.  We do [provide a way](http://docs.mongodb.org/ecosystem/tutorial/use-csharp-driver/#requeststart-requestdone-methods) for users to perform a "batch" of operations on the same connection by opting-in to the thread affinity mode.  This alone solves the problem for .NET users who are forced to work in this type of environment because we don't steal connections from other threads.  This does limit overall throughput because connections are now exclusively checked-out until they are released.  So, if your max connection pool size is 100, then only 100 requests can be handled at a given time.

### Summary

Long story short, using a load balancer to actualy do load balancing between mongos' is likely going to cause some problems. Rather, I recommend that a mongos exists on each application server.  This has a few benefits, the foremost being that there is no network involved between the application and the mongos.  If this is not possible due to security configurations or policies, mongodb most certainly supports connecting to one or more mongos across a network and most of our drivers support automated failover between them.  In fact, some drivers already support load balancing between mongos themselves and the rest will likely support this in the future as well. 

So, if you are planning on using a load balancer, take care during configuration.  If you have any questions, please post them to [mongdb-user](https://groups.google.com/forum/#!forum/mongodb-user).