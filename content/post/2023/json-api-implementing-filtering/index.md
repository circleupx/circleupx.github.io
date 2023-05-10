---
title: JSON:API Implementing Filtering
tags: [JSON:API]
author: "Yunier"
date: "2023-5-1"
description: "Implementing filtering with JSON:API"
published: false
---

It has been over [a year since I last wrote](/post/2022/json-api-pagination-links/) about JSON:API, since then the [team](https://jsonapi.org/about/#editors) behind JSON:API has published verision 1.1. I want to continue my journey of document JOSN:API in .NET by introducing a really cool feature into my [Chinook JSON:API](https://github.com/circleupx/Chinook) project, filterting. 

The first thing to know about filtering in JSON:API is that the spec itsel its agnostic to any filtering strategies. Meaning it is up to you to define how filtering should. This in my opinion has alwasy been a drawback in JSON:API, I believe in that it would have been a better choice for the spec if it had decided on a filtering strategy, but that is discussion for another day. While the spec does not favor any filtering strategy it does have some [recommendations](https://jsonapi.org/recommendations/#filtering).

The spec recommends using the LHS bracket syntax to denote which resource the filtering should be applied to. For example, imagine you are dealing with a post resource and each post resource can expose a relationship to an author resource, that is to say, each post has a 1 to 1 relationship with with the authors resource. 

As a client of the API if you wanted to find out which post has an author with a firstName of "Dan" you may need to pull all traverse through all post resource, then in-memory filter on only the post that match your search predicate. This type of filtering is not idea, and I feel like most REST APIs out in the wild implemen filtering using the strategy I just described. 

We can do better, in fact JSON:API makes it easy for us to implement filter, if you have properly defined relationships between resource then your client should be able to make the following request.

```shell
GET /comments?filter[post]=published eq true&filter[author]=firstName eq 'Dan' HTTP/1.1
```

Note the usage of LSH brackets and ODATA syntax, we'll talk about that later on this post. If the request above is valid, then it should yield the following JSON:API response.

```JSON
{
  "data": [
    {
      "type": "post",
      "id": "1",
      "attributes": {
        "title": "JSON:API paints my bikeshed!",
        "published" : true
      },
      "links": {
        "self": "http://example.com/post/1"
      },
      "relationships": {
        "author": {
          "links": {
            "self": "http://example.com/post/1/relationships/author",
            "related": "http://example.com/post/1/author"
          },
          "data": {
            "type": "author",
            "id": "9"
          }
        }
      }
    }
  ],
  "included": [
    {
      "type": "author",
      "id": "9",
      "attributes": {
        "firstName": "Dan",
        "lastName": "Gebhardt",
        "twitter": "dgeb"
      },
      "links": {
        "self": "http://example.com/author/9"
      }
    }
  ]
}
```

The reason I say that JSON:API makes filtering easier is that each JSON:API document can be viewed as a graph. Take the the [compound document](https://jsonapi.org/format/#document-compound-documents) above