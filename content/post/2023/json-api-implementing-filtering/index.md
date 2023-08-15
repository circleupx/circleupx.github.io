---
title: JSON:API Implementing Filtering
tags: [JSON:API]
author: "Yunier"
date: "2023-05-18"
description: "Implement filtering in JSON:API"
draft: true
---

It has been over [a year since I last wrote](/post/2022/json-api-pagination-links/) about JSON:API, since then the [team](https://jsonapi.org/about/#editors) behind JSON:API has published verision 1.1. I want to continue my journey of documenting JOSN:API in .NET by introducing a really cool feature into my [Chinook JSON:API](https://github.com/circleupx/Chinook) project, filterting.

The first thing to know about filtering in JSON:API is that the spec itself is agnostic to any filtering strategies. Meaning it is up to you to define how filtering should be handled by your API. In my opinion has alwasy been a drawback in JSON:API, I believe in that it would have been a better choice for the spec if it had decided on a filtering strategy, but that is discussion for another day. While the spec does not favor any filtering strategy it does have some [recommendations](https://jsonapi.org/recommendations/#filtering).

The spec recommends using the LHS bracket syntax to denote which resource the filtering should be applied to. For example, imagine you are dealing with a post resource and each post resource can expose a relationship to an author resource, that is to say, each post has a 1 to 1 relationship with with the authors resource. 

As a client of the API if you wanted to find out which post has an author with a firstName of "Dan", you would first need to filter the authors resource to only those that have "Dan" as a first name, then once you have those resources, you can filters the posts resource to only post that have a matching author's resource. This type of filtering is not ideal, and from my experience, it is how many REST APIs out in the wild implemented, this is the well-know problem of overfetching and underfetching, a selling point of GraphQL.

We can do better, in fact JSON:API makes it easy for us to implement filter, if you have properly defined relationships between resource then your client should be able to make the following request to handle the scenarion I just described.

```shell
GET /posts?filter[post]=published eq true&filter[author]=firstName eq 'Dan'&include=author HTTP/1.1
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

In a single request, the client has requested that the API should get all posts where the published field is true and to include the all the related authors, the request also states that out of that list of posts, the API should only return posts where the author is name Dan. Using a nested filtering allows the API to overcome the overtching and underfetching problems that many REST APIs have.

So, how can we implement this in feature in .NET? Let's take a look.

The first step in implementing filtering will be to take the filter query paremeter from the incoming HTTP request URL and parse the request. You have the option to implement a customer parse, see my [Parsing in C#](/content/post/2021/parsing-in-csharp/) post, or use one of the many awesome parsing libraries that exit in .NET. Personally, I have always relied on [SuperPower](https://github.com/datalust/superpower) but for this implementation I'm going to use [Pidgin](https://github.com/benjamin-hodgson/Pidgin). Pidgin will allow you to parse and tokenize the filter query parameter. Once we have the a tokenized URL, we can build in Abstract Syntax Tree the generate runtime expression that can be then be given to an ORM system like EF Core. 

By the way, if you are interesting in learning more about parsers then you can enroll in the class [Building a Parser from scratch](http://dmitrysoshnikov.com/courses/parser-from-scratch/) by [Dmitry Soshnikov](http://dmitrysoshnikov.com/).

