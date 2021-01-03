---
title: Asynchronous Request in GraphQL
layout: post
tags: [GraphQL, Subscriptions, REST, HTTP, Websocket, Asynchronous Jobs]
readtime: true
---

When I first started to learn about GraphQL I was somewhat surprise to learn that the [GraphQL specification](https://spec.graphql.org/June2018/) did not provide any guidance or spoke of any methods to handle asynchronous request. By asynchronous request, I mean request that cannot be completed within your normal request-response context. 

For example, take an API that aggregates orders by combining various types of filters, the API may allow you to filter by only orders that are greater than $100.00, or orders placed in certain date range, or orders that have a particular product and so on. Depending on the amount of data and filters used, the query to get the data may take a couple of minutes, maybe even hours. The question now becomes how to best handle long-running request in GraphQL.

If the API were RESTful then one solution to this problem might be to allow the client to [poll the API](http://restalk-patterns.org/long-running-operation-polling.html). The RESTful API may expose a "jobs" resource, remember anything in rest can be a resource, even concepts such as a long running job. The jobs resource would then accept a job request from the client, the API would respond to the client with a [202 Status code](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/202) instead of a [200 OK](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200). The API server would then begin processing the client's request, and the client can then query the status of the job until it receives a [303 See Other](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/303) HTTP respond from the API server with a location header indicating to the client where it can retrieve the output of the job. 

In a GraphQL API there are two solutions to this problem. The first solution is the same solution used on our RESTful API example, the client submits a long running request to the API then polls the status of the request on a set interval. The trick with GraphQL is that GraphQL is a funnel API, every request in GraphQL goes through the same endpoint, hence the name funnel API, as GraphQL does not have the concept of using individual URIs to represent resources, therefore, a GraphQL API that whishes to support long-running request would need to expose an endpoint that handle long-running request.

{: .box-note} 
The methodology described above is also mention on the book, [Production Ready GraphQL](https://book.productionreadygraphql.com/) by [Marc-André Giroux](https://twitter.com/__xuorig__). I highly recommend this book to anyone wanting to learn about GraphQL.

The second solution would be to use [GraphQL subscriptions](https://www.apollographql.com/docs/react/data/subscriptions/). GraphQL subscriptions are awesome, they can leverage the power of [Websocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket). The client application and the GraphQL API can switch protocols from HTTP to Websocket anytime there is need to process a long-running request, thanks to the [full-duplex](https://en.wikipedia.org/wiki/Duplex_(telecommunications)#FULL-DUPLEX) nature of Websocket, the GraphQL API can use the existing connection opened by the client to send back the output of the job request to the client.

{: .box-note} 
[Shopify's Admin API](https://shopify.dev/tutorials/perform-bulk-operations-with-admin-api) is a good example of a GraphQL API that can query a large collection of data using a polling mechanism.

In GraphQL we have two options to help us deal with long-running request, the question now is when to use each option. At first, I was under the impression that Subscriptions should only be used from here on out, that polling was a dead mechanism, after all why would you want to create a specific endpoint that only deals with a certain type of request. As it turns out, it depends, as discussed by Marc-André on [this](https://twitter.com/__xuorig__/status/1343297735596908546) tweet it all depends on the request-response context. If our use case is one where the client may not get a response for a couple of minutes, maybe even an hour, then the best solution is to go with polling, after all no client would want to keep a WebSocket connection open to the server for that long. On the other hand, if our use case involves request-respond context that only last a few seconds, then I recommend using a GraphQL Subscription.