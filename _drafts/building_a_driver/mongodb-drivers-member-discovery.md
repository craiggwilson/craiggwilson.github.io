---
layout: post
title: MongoDB Driver Member Discovery
description: How mongodb drivers auto-discover replica set members.
tags: [mongodb, .net, infrastructure]
image:
  feature: abstract-9.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

> How mongodb drivers auto-discover replica set members.

One of the things [MongoDB](http://mongodb.org) does great is high availability.  It uses a concept called [replica sets](http://docs.mongodb.org/manual/replication/) such that there are N nodes that all have the same data eventually, depending on network latency and configuration.  A replica set itself is a single master system such that only 1 of N nodes will accept writes.  The master in this system is called a "primary".  As these changes happen in the primary, they get replicated to what we call secondaries. In addition to primaries and secondaries, there are also non-data bearing nodes called "arbiters" which are used to break ties in voting.  (It follows that N should always be odd when using replica sets).

As far as a [driver](http://docs.mongodb.org/ecosystem/drivers/) is concerned, it needs to be aware of all the nodes in a system to which it is allowed to talk. MongoDB has a standard [connection string](http://docs.mongodb.org/manual/reference/connection-string/) which all drivers support that allows for the specification of a number of hosts.  However, it isn't required that all hosts be on the connection string in order for them to get utilized.  The driver should be able to pick up the members that were not part of the connection string.  In addition, it is entirely possible for members to be added and removed from a replica set at runtime.  The driver needs to be able to handle these situations.  To do this, there are two commands that provide the information we need.  

Our first option, [replSetGetStatus](http://docs.mongodb.org/manual/reference/command/replSetGetStatus/), is a command that many of you may be familiar with.  When using the mongo shell, you can use the helper rs.status() which invokes this command for you.  It provides back all the members, their state, and lots of other information.  Below is an example from a replica set running on my local box right after I started up a the replica set (the secondaries haven't finished starting up).

{% highlight js %}
{
        "set" : "funny",
        "date" : ISODate("2013-10-22T14:44:25Z"),
        "myState" : 1,
        "members" : [
                {
                        "_id" : 0,
                        "name" : "localhost:40000",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 97,
                        "optime" : Timestamp(1382453061, 1),
                        "optimeDate" : ISODate("2013-10-22T14:44:21Z"),
                        "self" : true
                },
                {
                        "_id" : 1,
                        "name" : "localhost:40001",
                        "health" : 1,
                        "state" : 5,
                        "stateStr" : "STARTUP2",
                        "uptime" : 10,
                        "optime" : Timestamp(0, 0),
                        "optimeDate" : ISODate("1970-01-01T00:00:00Z"),
                        "lastHeartbeat" : ISODate("2013-10-22T14:44:25Z"),
                        "lastHeartbeatRecv" : ISODate("2013-10-22T14:44:23Z"),
                        "pingMs" : 0
                },
                {
                        "_id" : 2,
                        "name" : "localhost:40002",
                        "health" : 1,
                        "state" : 6,
                        "stateStr" : "UNKNOWN",
                        "uptime" : 4,
                        "optime" : Timestamp(0, 0),
                        "optimeDate" : ISODate("1970-01-01T00:00:00Z"),
                        "lastHeartbeat" : ISODate("2013-10-22T14:44:23Z"),
                        "lastHeartbeatRecv" : ISODate("1970-01-01T00:00:00Z"),
                        "pingMs" : 0,
                        "lastHeartbeatMessage" : "still initializing"
                }
        ],
        "ok" : 1
}
{% endhighlight %}

Seems like a winner to me, except for the reason we don't use it.  In a configuration where authentication is required, replSetGetStatus requires authentication.  And it's not just any authentication, but admin-level authentication.  As most applications will not be provided with admin-level authentication, this is a no-go.

Our second option is another command called [isMaster](http://docs.mongodb.org/manual/reference/command/isMaster/).  This provides different information, although the parts we care about still exist - namely the hosts field. In other setups, other fields with members may be present as well.  These would show up as "passives" and "arbiters".

{% highlight js %}
{
        "setName" : "funny",
        "ismaster" : true,
        "secondary" : false,
        "hosts" : [
                "localhost:40000",
                "localhost:40001",
                "localhost:40002"
        ],
        "primary" : "localhost:40000",
        "me" : "localhost:40000",
        "maxBsonObjectSize" : 16777216,
        "maxMessageSizeBytes" : 48000000,
        "localTime" : ISODate("2013-10-22T14:49:20.316Z"),
        "ok" : 1
}
{% endhighlight %}

So, that's great.  We have all the information we need.  Each driver periodically calls isMaster on each of the members it knows about and uses that information to determine any members it doesn't know about or members that it knows about that it shouldn't.  As new members get added, or old members get removed, the 3 fields containing members(hosts, passives, arbiters) change and the driver can then add or remove the member from it's internal members list.  Piece of cake, right?

Actually, yes; it's really pretty simple.  But there are most certainly some issues that come up.  For instance:

- What if two members are reporting different lists of members?
	- This is basically just a big distributed race condition.
- What if a new primary shows up before you receive notice that the old primary has stepped down?
	- In other words, we have 2 servers currently claiming to be the primary in a single master system.
- What if the user places 2 servers on the connection string with the same name?
	- This might not be obvious, as one could be localhost:27017 and the other 127.0.0.1:27017.
- What if the user places a non-replica set member (i.e. a standalone or a mongos) on the connection string along with a replica set member?

None of these are difficult to solve in isolation.  We just need a documented way to handle each case.  Currently, as each driver has been developed semi-independently by different authors, both inside and outside MongoDB, some of these edge cases are handled a bit differently.  In addition, some of these edge cases can be non-trivial to test and require manually setting up and messing with servers to watch how a driver reacts.  All in all though, this works extremely well and many of the differences in how drivers handle different situations stems from whether a runtime supports multiple-threads or not.  For instance, .NET can be calling isMaster on all the members it knows about simultaneously and be much more responsive to user requests where a PHP or Node.js application, which only has a single thread, is in a holding pattern until it hears back from each server.

As always, if you have any questions, feel free to post a comment or ask on [mongodb-user](https://groups.google.com/forum/?hl=en#!forum/mongodb-user).