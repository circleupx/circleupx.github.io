---
title: GraphQL is protocol agnostic
tags: ["GraphQL"]
readtime: true
---

I'm seeing many API developers, specially those that come from a REST background, struggle with GraphQL simply because they are introducing protocol concepts into their GraphQL documents. 

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">give it a REST... <a href="https://t.co/sUxqL4ACdj">pic.twitter.com/sUxqL4ACdj</a></p>&mdash; I Am Devloper (@iamdevloper) <a href="https://twitter.com/iamdevloper/status/1327190006520221696?ref_src=twsrc%5Etfw">November 13, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

GraphQL is not bound to any network protocol, it is most often implemented on top of the HTTP protocol and it only uses the most basic features of HTTP. That is because GraphQL treats HTTP as a dum pipe. Introducing protocol concept such as a 404 Not Found status code into your GraphQL documents will only cause you development pain.

Now, I know what you are thinking, "but I don't want to reinvent the wheel", normally you would be correct, after all the http spec takes care of many things that developers often take for granted, things like authorization and caching, but in the case of GraphQL ignoring the HTTP spec is the **price of admission to creating a good GraphQL API.**

**Credits:**
- [A few things to think about before blindly dumping REST for GraphQL.](https://apihandyman.io/and-graphql-for-all-a-few-things-to-think-about-before-blindly-dumping-rest-for-graphql/)
- [Serving over HTTP](https://graphql.org/learn/serving-over-http/)
- [Modeling Errors in GraphQL](https://engineering.zalando.com/posts/2021/04/modeling-errors-in-graphql.html)
