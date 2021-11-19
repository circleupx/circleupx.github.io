---
title: Idempotency
tags: [REST]
author: "Yunier"
date: "2021-11-17"
description: "Making API calls safe to retry."
---

Idempotency, one of the key features any Web API should have. The idea is that [software is unrealiable](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing), the network can fail, the database the API connects to could be offline, the API itself could be performing an intense operation that impacts performance. For all these reasons an API client may resubmit a request, not much of a problem if you are dealing with [GET](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/GET), [HEAD](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/HEAD), [PUT](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/PUT) or [DELETE](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/DELETE), these HTTP methods are idempotent, [POST](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/post) and [PATCH](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/PATCH) on the other hand are not. 

An HTTP POST to an orders API will create a new order every time the API gets called. This behavior is not desired, after all, your customers are not going to be happy to see that they have been charged for multiple orders. For this reason, it is essential that when you create a Web API, an effort should be made into making all POST and PATCH requests idempotent. 

Two techniques have surfaced over the years on how to make POST and PATCH idempotent. The first technique involves having the API provide the client with a one-time URI. This one-time URI will come with an embedded token. The idea here is that the API can track each token in order to determine if the request is being submitted for the first time or if the request has already been processed or is being processed. 

A more popular technique, evangelised by [Stipe](https://stripe.com/), comes in the form of using an [Idempotancy-Key](https://tools.ietf.org/id/draft-idempotency-header-00.html) [header](https://tools.ietf.org/id/draft-idempotency-header-00.html). In this approach, the key is generated using V4 [UUIDs](https://datatracker.ietf.org/doc/html/rfc4122), to guarantee enough randomness, the client is then responsible for including this header on all POST and PATCH requests, and like in the first technique, the server is responsible for determining if the request is being submitted for the first time or if the request has already been processed.