---
title: Sort Functions Are Non-Deterministic 
tags: [REST, SQL Server]
author: "Yunier"
date: "2021-11-13"
description: "Why sorting must be deterministic."
---

When building a Web API, RESTful or GraphQL, you may want to expose some functionality that allows a client application to sort data.

From my experience, this is often not implemented correctly. Many developers fail to realize that sorting should always be sort plus one. The plus one is a unique value, like a primary key or identifier. The reason for this is that sorting in most databases, [like SQL Server](https://docs.microsoft.com/en-us/sql/t-sql/queries/select-order-by-clause-transact-sql?redirectedfrom=MSDN&view=sql-server-ver15#arguments), is nondeterministic, meaning the sort function may return different results each time they are called with a specific set of input values even if the database state that they access remains the same.

It is important to not make this mistake in a Web API, especially in a RESTful system given that RESTful APIs rely upon HTTP. In HTTP, the GET, HEAD, PUT, and DELETE methods are [idempotent](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent), the same request should always return the same value.

Having an API that uses idempotency correctly becomes super useful when you have API caching. Idempontacy will give you greater cache hits. For example, take the following data set based on one of my favorite TV shows, [WestWorld](https://en.wikipedia.org/wiki/Westworld_(TV_series)).

```json
[
    {
        "id": 869671,
        "url": "https://www.tvmaze.com/episodes/869671/westworld-1x01-the-original",
        "name": "The Original",
        "type": "regular",
        "runtime": 60,
    },
    {
        "id": 911201,
        "url": "https://www.tvmaze.com/episodes/911201/westworld-1x02-chestnut",
        "name": "Chestnut",
        "type": "regular",
        "runtime": 60
    },
    {
        "id": 911204,
        "url": "https://www.tvmaze.com/episodes/911204/westworld-1x03-the-stray",
        "name": "The Stray",
        "type": "regular",
        "runtime": 60
    }
]
```

This data is exposed by the [tvmaze API](https://www.tvmaze.com/api), imagine now that this API allows you to sort the data using the following request.

```text
https://api.tvmaze.com/episode?sortDesc=runtime,type
```

You may get the following response.

```json
[
    {
        "id": 869671,
        "url": "https://www.tvmaze.com/episodes/869671/westworld-1x01-the-original",
        "name": "The Original",
        "type": "regular",
        "runtime": 60
    },
    {
        "id": 911204,
        "url": "https://www.tvmaze.com/episodes/911204/westworld-1x03-the-stray",
        "name": "The Stray",
        "type": "regular",
        "runtime": 60,
    },
    {
        "id": 911201,
        "url": "https://www.tvmaze.com/episodes/911201/westworld-1x02-chestnut",
        "name": "Chestnut",
        "type": "regular",
        "runtime": 60
    }
]
```

The request is executed again, it may now return the following response.

```json
[
    {
        "id": 911201,
        "url": "https://www.tvmaze.com/episodes/911201/westworld-1x02-chestnut",
        "name": "Chestnut",
        "type": "regular",
        "runtime": 60,
    },
    {
        "id": 869671,
        "url": "https://www.tvmaze.com/episodes/869671/westworld-1x01-the-original",
        "name": "The Original",
        "type": "regular",
        "runtime": 60,
    },
    {
        "id": 911204,
        "url": "https://www.tvmaze.com/episodes/911204/westworld-1x03-the-stray",
        "name": "The Stray",
        "type": "regular",
        "runtime": 60,
    }
]
```

The same request returned a different result each time it was executed even though the data did not change. This is because all three records have the same runtime and the same type, the sort function will correctly sort the data by runtime and type, but since the request wasn't specific enough the other properties are randomly sorted. Again, to avoid this type of issue, always include an extra sort property that is guaranteed to be unique, like an identifier. If caching is supported then the two API requests above would have all resulted in a cache miss, since the data was in a different order, resulting in a different ETAG for each request, thus a miss.

The API request above should really be.

```text
https://api.tvmaze.com/episode?sortDesc=runtime,type,id
```

By including the Id property, the sorting order will now always be guaranteed, because the id field will always be unique. By always adding a unique field to a sort operation, the API will achieve greater cache hits and provide an overall better experience for the consumer of the API.
