---
title: Practical API validation in .NET
tags: [REST, FluentValidation, ProblemDetails]
author: "Yunier"
date: "2021-10-13"
description: "Guide on how to create practical API validation in .NET"
draft: true
---

In my [last post](../a-better-api-validation-strategy/) I wrote about using [JSON Schema](https://json-schema.org/) to do API validation. The main benefit being that the API can expose the schema as an API resource that can be consumed by clients. Using this approach client do not need to duplicate validation logic, they only need to execute the schema. For this post, I would like to explore API validation in .NET, especically using the [FluentValidation](https://fluentvalidation.net/) library and how we can expose validation error generated using FluentValidation with [Problem Details](https://datatracker.ietf.org/doc/html/rfc7807).

As always, I am going to enhance my [Chinook](https://chinook-jsonapi.herokuapp.com/) API project. I don't want to use the cutomer resource since that uses JSON Schema to do validation. Instead, I will enhance the invoice controller by allowing HTTP POST request. Allowing a client to create a new invoice. I am going to follow the same process I outlined in my [JSON:API - Creating new resources](../../2021/json-api-creating-new-resources) post.

Per the Chinook SQLite database project, the API should handle the following validation rules.

1. The invoice data cannot be null, meaning it is required.
2. Billing address may not exceed 70 characters.
3. Billing city may not exceed 40 characters.
4. Billing state may not exceed 40 characters.
5. Billing country may not exceed 40 characters.
6. Billing code may not exceed 40 characters.
7. Invoice total can only be numeric.


