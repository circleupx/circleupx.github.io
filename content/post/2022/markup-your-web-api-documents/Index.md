---
title: Markup Your Web API Documents
tags: [REST]
author: "Yunier"
date: "2022-03-31"
description: "Enrich your Web API documents."
---

I've been thinking about what it takes to build a good Web API, regardless of the technology (REST vs GraphQL) or philosophy used. One concept that has been stuck on my head is the idea of marking up API documents to provide more context around the data. 

A Web API document is the response returned by the API itself, you will often see this term used in API specifications like [GraphQL](https://spec.graphql.org/October2021/#sec-Document), [HAL](https://stateless.group/hal_specification.html), [JSON-LD](https://w3c.github.io/json-ld-syntax/#loading-documents), and [JSON:API](https://jsonapi.org/format/#document-structure). Web API documents can be very simple, for example, imagine working with a Web API that manages users. This API may choose to represent the user resource using the following JSON.

```json
{
    "id": 1234,
    "name": "Emilio",
    "age": 23,
    "dateOfBirth": "2020-03-09T22:18:26.625Z" 
}
```

You may be building Web APIs like this example, if you are, know that technically there is nothing wrong with this approach, but I would advise you to shift away from building Web APIs like this. From my experience, the example above is often true of Web APIs that simply take a data model, the database representation of some business entity then dumped said entity onto the clients as the API response. Please understand that <b>the database is not your API</b>. 

Web APIs are more than just your data, Web APIs should be composed between your data, API semantics, and the [actions that can be performed on that data](http://amundsen.com/blog/archives/1167). Essentially, you should be enriching your API responses, you should be providing context within your Web API documents.

Providing more than just data is a good way to ensure that the Web APIs you build is good, usable, and long-lasting. Don't believe me? Take a good look at the World Wide Web. The Web is just a large collection of APIs linked together through hypermedia, these APIs retrieve data in the following format.

```html
<!DOCTYPE html>
<html>
    <body>
        <p>I'm <em>so</em> happy to meet you</p>
        <p></p>
    </body>
</html>
```

What we have here is a simple HTML document. Now imagine a world where HTML did not exist, instead, the World Wide Web relied on just plain text, just the raw data. The HTML document above in an HTML-less world would simply look like this.

```
I'm so happy to meet you
```

Do you see the difference? 

In the two examples above, the data is the same, but the context was lost in the second example. With the HTML, the data was enriched, the html provided a context that is then given to all consumers of this document, and that context was that the client should put an emphasis, [see em tag](https://www.w3schools.com/tags/tag_em.asp), on the fact that I truly was happy to meet you.

This very basic example illustrates the power of HTML, the idea is that data should be [marked up](https://en.wikipedia.org/wiki/Markup_language) for ease of consumption both by humans and computers.

Switching back to our imaginary user Web API. How should it's document be marked up? 

Well, how about pagination data? That is always good. How about including schema definitions? Hey, schemas are great, SQL has taught us that since the 1970s. Our user Web API document could now potentially be represented using the following JSON.

```JSON
{
    "schema": "https://api.example.com/users/schema",
    "meta": {
        "page": 1,
        "next": null,
        "last": null,
        "total": 1
    },
    "data" : {
        "id": 1234,
        "name": "Emilio",
        "age": 23,
        "dateOfBirth": "2020-03-09T22:18:26.625Z" 
    },
    "relationships" : {
        "permissions": "https://api.example.com/users/1/permissions"
    }
}
```

This is a better approach to building Web APIs because the Web API document is no longer just data. The document is composed of data and it's context which gives the client additional information. Additionally, your mind starts to shift away from data models to document models which will lead you down the path of building a flexible API document. This is where a spec like [JSON:API](https://jsonapi.org/) shines because you never think about what data each API endpoints return, they are all just documents, the only difference is the data within each of those documents. 