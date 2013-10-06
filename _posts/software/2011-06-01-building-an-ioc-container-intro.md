---
layout: post
title: Building an IoC Container - Intro
description: Series introduction to building an IoC container.
tags: [.net, ioc, solid]
image:
  feature: abstract-1.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: false
share: true
---

As I have moved into a consulting role, I have become more and more surprised with the number of .NET developers who have not heard of Dependency Injection or Inversion of Control.  Talking to different people about the concept is met with a few reactions.  While I can’t do anything about the people who simply don’t care, there are two groups of people that I can help.

I began giving talks entitled "Build Your Own IoC Container" at local events here in Dallas.  This isn't because I'm a proponent of actually building your own and using it production, but rather as a way to educate the two different groups of people.  (Just to reiterate; please don't build your own for production.)

The first group are those who simply don't know about IoC.  The talk (and this series) simply start with a blank slate and begin building in a test-driven(sort of) manner.  I'll use Machine.Specifications as it makes clearer the intent (plus, I just really like it).  I've found that by removing the "black-box", there is more understanding and adoption.

The second group are those who already know about how containers work, but don't know how to explain them to others.  I was once in this place and needed to figure out how to talk about it with someone who didn't know.  So, I did what I always do when I don't understand something; I build it.  This helped me understand enough of the insides to answer the more difficult questions.  Hopefully, by building one together, you and I will gain a greater understanding of the basics of a container.

At the end of the series, we’ll have a fully functional container that supports pluggable lifetimes, multiple registrations for a type, factories, and cyclic dependencies.  There are 218 lines and 1 file.

- [Resolving Types without Dependencies]({% post_url software/2011-06-03-building-an-ioc-container-resolving-types-without-dependencies %})
- [Resolving Types with Dependencies]({% post_url software/2011-06-04-building-an-ioc-container-resolving-types-with-dependencies %})
- [Resolving Types with Unresolvable Dependencies]({% post_url software/2011-06-06-building-an-ioc-container-resolving-types-with-unresolvable-dependencies %})
- [Resolving Abstractions]({% post_url software/2011-06-08-building-an-ioc-container-resolving-abstractions %})
- [Adding Lifetimes]({% post_url software/2011-06-12-building-an-ioc-container-adding-lifetimes %})
- [Refactoring]({% post_url software/2011-06-15-building-an-ioc-container-refactoring %})
- [Cyclic Dependencies]({% post_url software/2011-06-17-building-an-ioc-container-cyclic-dependencies %})