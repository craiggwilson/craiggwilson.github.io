---
layout: post
title: Disconnecting from MongoDB
description: "Calling Disconnect in the MongoDB .NET Driver is Probably the Wrong Decisions."
tags: [mongodb, .net, best-practice]
image:
  feature: abstract-10.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: false
---

> Calling Disconnect in the MongoDB .NET Driver is probably the wrong decision.

It's kinda funny.  Developers see something called Disconnect and just assume they should call it when they are done.  Maybe that's reasonable.  If so, we made a mistake by including it in the driver.  What's more is that good API design should make it [easy to do the right thing and difficult to do the wrong thing](http://www.codinghorror.com/blog/2007/08/falling-into-the-pit-of-success.html).

API design is something we work really hard at in building the drivers at MongoDB.  Lots of time is spent going over options and lots of options exist.  When we are adding new features, all of the drivers are generally doing it concurrently, so decisions generally need to be applicable across all the drivers we maintain, from .NET and Java to Ruby and Python to Node.js and Erlang.  Obviously, finding commonality in these is sometimes difficult and our overarching goal is that each driver should provide something idiomatic to the target audience, something that feels natural.

I think we missed the boat here with Disconnect.  It is the source of numerous support tickets for developers who blindly call something without knowing what it does.  Many times, the cause is the copy-and-paste coder who doesn't even understand what the code does.  Other times, it's a junior dev who simply doesn't know to think about certain things.  However, the commonality between the problems is that we provided this method and so it is ultimately our fault.

We have begun working on the next major revision of the driver.  In fact, all the drivers supported by [MongoDB](http://www.mongodb.com) are working on their next major revisions.  We are all working through similar issues and trying to unify our designs.  The .NET driver is going to try to maintain backwards compatibility and we have a great mechanism to do so, but there are certain things I question.  Should Disconnect simply be removed?  I really can't think of any good reason for it to exist.