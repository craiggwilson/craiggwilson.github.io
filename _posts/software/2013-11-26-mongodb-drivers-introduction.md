---
layout: post
title: MongoDB Drivers - Introduction
description: Series introduction on building a MongoDB Driver.
tags: [mongodb, .net, drivers]
image:
  feature: abstract-10.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

> Series introduction on building a MongoDB Driver.

Building a driver for [MongoDB](http://mongodb.org) encompasses a wide variety of topics and deals with quite a few different areas of programming.  I'd like to walk through the basics of building a driver in MongoDB over a series of posts.  We'll start with [bson](http://bsonspec.org) and go until I run out of things to talk about.  In particular though, I'll focus on the aspects and decisions that were made and are continuing to be made while composing the .NET driver.  I'll also discuss some of the places where the .NET driver diverges from other drivers.  To begin with though, I'd like to give an introduction of what a driver is for MongoDB and why it's important.

### What is a Driver?

First, it's the piece of software that sits between your application and the server.  It talks to the server using a specific [protocol](http://docs.mongodb.org/meta-driver/latest/legacy/mongodb-wire-protocol/) over TCP/IP.  Underneath, most drivers pool connections, handle authentication, perform automatic member discovery and failover, etc... In short, drivers manage all of the network stuff so you can focus on building your applications without needing to worry about the details of what is involved in communicating with MongoDB.

Second, a driver is responsible for making an API that is idiomatic to its target user base.  The .NET driver should look like a .NET library (not a python module) and respect the conventions that typical .NET libraries use. There are most certainly concepts, algorithms, and high-level API decisions that are shared across our many [drivers](http://docs.mongodb.org/ecosystem/drivers/), but each one distinctively looks like a native library to its target user base.  I believe this is one of the reasons users love MongoDB - working with a driver feels like working with everything else in their framework of choice.

### Going Forward

I'll update the below list as new articles are published.  Please ask questions in the comments and I'll do my best to answer them.  The .NET driver team (Robert Stam, [Sridhar Nanjundeswaran](https://twitter.com/snanjund), and I) are currently refactoring large portions of our code base for our next major release, so many of the questions and decisions that need to be made are relatively fresh in my mind.

* [Introduction]({% post_url software/2013-11-26-mongodb-drivers-introduction %})
* [Bson]({% post_url software/2013-12-02-mongodb-drivers-bson %})
* [Wire Protocol - 1]({% post_url software/2013-12-10-mongodb-drivers-wire-protocol-1 %})
* [Wire Protocol - 2]({% post_url software/2014-01-21-mongodb-drivers-wire-protocol-2 %})