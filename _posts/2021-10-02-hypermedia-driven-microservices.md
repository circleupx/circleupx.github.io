---
title: Hypermedia driven microservices
tags: ["REST"]
published: false
---

At a previous job we had hypermedia driven api that followed the [JSON:API](https://jsonapi.org/) specification. The API was a monolith, it handled all business logic, transactions and workflows. When the time came to talk about possibly switching to a microservice oriented architecture, an idea emerged, to use hypermedia to stitch up the different microservices, the same idea that led to the world wide web become such a successful distributed hypermedia system. 


Unfortunately, the team was never ever to complete the implementation. Lately, I've seen some chatter on twitter on hypermedia driven APIs that includes using hypermedia to stitch up multiple APIs.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr"><a href="https://twitter.com/hashtag/hypermedia?src=hash&amp;ref_src=twsrc%5Etfw">#hypermedia</a> <a href="https://twitter.com/hashtag/API?src=hash&amp;ref_src=twsrc%5Etfw">#API</a> people out there: have you talked about or heard someone talk about &quot;intra-API&quot; links and &quot;inter-API&quot; links? just wondering if that perspective might be useful as a way to highlight one possible advantage of hypermedia: providing seamless <a href="https://twitter.com/hashtag/DX?src=hash&amp;ref_src=twsrc%5Etfw">#DX</a> in API landscapes.</p>&mdash; Erik Wilde (@dret) <a href="https://twitter.com/dret/status/1388871515136012291?ref_src=twsrc%5Etfw">May 2, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

For a while now I've been meaning to build a hypermedia driven microservice proof of concept to understand the possible benefits and drawbacks of such systems. To build a hypermedia driven microservice architecture, a few problems will need to be solved. They are as follows.

1) Discoverability - the API's need to be dynamically linked together.
2) Linkability - Each API should be able to build link to resources that are located in another API.


The first problem I have to solve is recoverability, I need the APIs to be somewhat aware of each other. HashiCorp's [Consul](https://www.hashicorp.com/products/consul) is a great tool to use for service discovery. Each API can be registered with Consul, then on service startup I can query Consule for metadata, like URIs.

