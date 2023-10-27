---
title: Building A Web Server In Go
tags: [Go]
author: "Yunier"
date: "2023-08-15"
description: Building a Web Server in Go
draft: true
---

As mentioned in [Go - Multiple Return Values](/content/post//2023/go-multiple-return-values/) I have been learning Go, I've since finished reading [Learning Go](https://a.co/d/2B7htYx) by [Jon Bodner](https://www.amazon.com/stores/Jon-Bodner/author/B08SWGN5NN) and moved on to reading [Concurrency in Go](https://a.co/d/3ItFK4R) by [
Katherine Cox-Buday](https://www.amazon.com/stores/Katherine-Cox-Buday/author/B07567T8NX) and [Black Hat Go]() by [Chris Patten](https://www.amazon.com/stores/Chris-Patten/author/B08511B8M3) and [Tom Steele](https://www.amazon.com/stores/Tom-Steele/author/B084WN415T), all excellents books, Black Hack Go specifically as it provides many example on how to use Go's [net package](https://pkg.go.dev/net) to interact with a TCP connection. Interacting with TCP connections is key foundational process of any web server, so I wanted to put what I have learn so far about Go in this blog post, to build my own version of a Web Server even including my own custom routers.

Let's get started.


