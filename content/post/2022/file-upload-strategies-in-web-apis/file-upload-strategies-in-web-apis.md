---
title: File upload strategies in Web APIs
tags: [REST, .NET, HTTP, S3]
author: "Yunier"
date: "2022-01-01"
description: "How to upload files in Web APIs"
draft: true
---

How are you handling file uploads on HTTP APIs? This question was asked by David Fowler, he provided some option, but the community came back with some interesting alternatives, like using signed URLs to upload a file directly to an S3 bucket.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">How are you doing file uploads for your HTTP API?</p>&mdash; David Fowler 🇧🇧🇺🇸💉💉💉 (@davidfowl) <a href="https://twitter.com/davidfowl/status/1427420506597191699?ref_src=twsrc%5Etfw">August 17, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

The main consesus appears to be that you should use multipart/form-data. Let's see what that would look like in a .NET Web API. To start, I will create a new Web API project that will explorer the top options mentioned on that twitter thread.

First, I'll create the project with this command.

```console
 dotnet new webapi --name fileupload
```

I'm using .NET 6 for this project, the new API templates use [minimal APIs](https://docs.microsoft.com/en-us/aspnet/core/tutorials/min-web-api?view=aspnetcore-6.0&tabs=visual-studio), which is very nice. I am going to add new controller, I'll call it fileupload.

