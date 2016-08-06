---
layout: post
title:  "Hello, world!"
date:   2016-07-22 04:46:01 +0500
tags: []
---

I've been meaning to start this for a while and finally got around to setting up all the required boilerplate.
The reason for this is ~~I want to find a good job and all the cool developers have a blog~~ for the last year or so I've noticed that the most challenging things I had to do at work were not related to writing new code, but to investigating issues in production and dev environments.

During the last year I've had to

* learn to use hardcore Unix tools like perf and strace
* learn to use hardcore Windows tools like PerfView, APIMon, Procmon (and other SysInternals tools) and ETW
* reduce a system's latency by replacing compare-and-set with a global lock
* submit some code to the Mono repository (ok, ok, just a failing [test](https://github.com/mono/mono/blob/5e81ed1e03581885585e819c3dc14e60455cf42d/mcs/tests/gtest-634.cs) for a [bug](https://bugzilla.xamarin.com/show_bug.cgi?id=33669) :))
* use WinDbg to debug MS Word
* trace the cause of an issue to one of several Windows bugs 
	* [Error messages when a 32-bit application has the /LARGEADDRESSAWARE option](https://support.microsoft.com/en-us/kb/2588507)
	* [Unexpected ASP.Net application shutdown after many App_Data file changes occur](https://support.microsoft.com/en-us/kb/3052480)

	and my favorite one
	
	* [All the TCP/IP ports that are in a TIME_WAIT status are not closed after 497 days from system startup in Windows Server 2008](https://support.microsoft.com/en-us/kb/2553549)
* and generally spend time lots of time chasing tricky bugs in our code and in almost every library we use. 

In almost all of these cases a key insight into the problem came from someone's _blog_. Not from StackOverflow, not a book, but from some developer somewhere writing about their experience investigating a technology or chasing down a problem.

The most eye-opening moment came when in two neighbouring weeks a colleague and me came across two particularly gnarly bugs
and in both cases after hours of pulling hair and debugging the world were saved from going crazy only by bumping into a blog post from <https://www.zpqrtbnk.net/> describing the bug.
They were:
 
 * a .NET Framework bug related to CultureInfos crossing an appdomain (<https://www.zpqrtbnk.net/posts/appdomains-threads-cultureinfos-and-paracetamol>)
 * the IIS configuration change bug mentioned earlier (<https://www.zpqrtbnk.net/posts/iis-configuration-change-not>)

Before that I could justify to myself that people who’s blogs I read have the moral right to write because they are unusually smart and famous, while if I wrote anything it would only amount to internet pollution. But this guy does not strike as someone doing rocket science software development -- just working on a CMS and some libraries. 
And he saved our asses because he encountered these issues and decided to write about them. If he didn't, we would have lost much more hair and nerves.
Thank you, Stéphane Gay.

So, here goes.
I hope that in the near future I will have the time to write up some of the issues we've encountered and maybe this will save someone some time if they run across something similar.

