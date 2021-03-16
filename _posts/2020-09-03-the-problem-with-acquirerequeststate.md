---
title: The Problem With AcquireRequestState
layout: post
tags: [Session, AcquireRequestState, .NET Framework]
readtime: true
---

In my second post, I wanted to cover AcquireRequestState. In my four years as a developer I have encountered issues with AcquireRequestState twice. So, what in the world is AcquireRequestState.

AcquireRequestState is part of the ASP.NET [Life Cycle](https://docs.microsoft.com/en-us/previous-versions/aspnet/bb470252(v=vs.100)), this is an event raised by the [HttpApplication](https://docs.microsoft.com/en-us/dotnet/api/system.web.httpapplication?redirectedfrom=MSDN&view=netframework-4.8), it keeps session state in sync. Though I suspect that most developers are familiar with this event for being a major performance pain in their .NET Framework application, as documented [here](https://stackoverflow.com/questions/30066925/long-delays-in-acquirerequeststate), [here](https://discuss.newrelic.com/t/acquirerequeststate-is-delaying-response-times-web-api/38229), [here](https://stackoverflow.com/questions/3629709/i-just-discovered-why-all-asp-net-websites-are-slow-and-i-am-trying-to-work-out), [here](https://stackoverflow.com/questions/8349033/storing-anything-in-asp-net-session-causes-500ms-delays) and [here](https://stackoverflow.com/questions/35133150/newrelic-async-http-handler-and-acquirerequeststate).

As seen on the various link above, the problem is always the same. Someone notices HTTP request spending a large amount of time in AcquireRequestState. For example,

![A test image](https://nr-production-discourse.s3.amazonaws.com/original/2X/e/ed7a8022b1f8f75cf51decbe7e4d767750f8692a.png)

this transaction took ~95 seconds to complete, out of 95, 94 were spent inside AcquireRequestState. That is a ridiculous amount of time spend in code that you didn't write. So what is the problem? Just what in the world is going on inside the AcquireRequestState.

The problem is that **AcquireRequestState reads session variables in a queue** rather than in parallel, this is done to maintain thread safety and to prevent one session from overwriting the work done in another thread. I recommend reading this awesome [post](http://tech-journals.com/jonow/2011/10/22/the-downsides-of-asp-net-session-state) by Jono to understand more.

Solutions?

Aside from the obvious ones, like turning off session or setting session to read-only, which by the way, in my experience does not make much of a difference.

When I first encountered AcquireRequestState, turning off session was not an option. The project I was working on at the time required session to be on, in fact without session the payment system would not work at all, ouch. So we had to look for alternative solution. We noticed red-gate had a [post](https://www.red-gate.com/simple-talk/dotnet/asp-net/single-asp-net-client-makes-concurrent-requests-writeable-session-variables/) on writing your own custom Session provider, which made us wonder if anyone had created one. Turns out someone did. [Here](https://github.com/leewang0/RedisSessionProvider) is the project, it is a custom Session provider with Redis and boy it turned out to be great project. 

We immediately modified our .NET Framework project to utilize RedisSessionProvider and the benefits were noticed right away. I believe, we dropped our response time by about a second across all pages while lowering .NET CLR from about 200ms to about ~58ms.

During my second encounter with AcquireRequestState, I did not implement a customer Session provider. It was not needed, the application I was working on was only storing some basic information to session, and those values were available through some other means. So we turned off Session completely, thus eliminating AcquireRequestState. Once again, I saw response time improvement across the entire application, our .NET CLR went from about ~250ms to about ~30ms.