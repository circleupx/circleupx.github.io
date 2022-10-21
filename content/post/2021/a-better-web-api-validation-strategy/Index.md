---
title: A Better Web API Validation Strategy
tags: [REST, GraphQL, JSON Schema]
author: "Yunier"
date: "2021-10-09"
description: "Guide on how to improve API validation"
---

As an API developer, you will eventually need to determine how to handle data validation. 

The .NET ecosystem offers a few options, the first option, [validation attributes](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/validation?view=aspnetcore-6.0#validation-attributes), can be used to annotate how a model should be validated. Validation attributes are great, they don't require any external dependencies, you can specify error messages, create your own [custom validator](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/validation?view=aspnetcore-6.0#validation-attributes), validate against many data types. 

For example, take the following Movie class, notice how the properties have been annotated with validation rules.

```c#
public class Movie
{
    public int Id { get; set; }

    [Required]
    [StringLength(100)]
    public string Title { get; set; }

    [ClassicMovie(1960)]
    [DataType(DataType.Date)]
    [Display(Name = "Release Date")]
    public DateTime ReleaseDate { get; set; }

    [Required]
    [StringLength(1000)]
    public string Description { get; set; }

    [Range(0, 999.99)]
    public decimal Price { get; set; }

    public Genre Genre { get; set; }

    public bool Preorder { get; set; }
}
```

Another popular option in .NET is using the library [Fluent Validation](https://fluentvalidation.net/), this library allows you to define validation rules that are then applied against a data class. Fluent Validation offers the same capabilities as validation attributes, with one additional benefit, using the Fluent Validation library means that your classes are kept clean. The validation rules are defined in a configuration class. 

For example, take the following CustomerValidator class, which defines rules that can be applied to the Customer class.

```c#
public class CustomerValidator : AbstractValidator<Customer> {
  public CustomerValidator() {
    RuleFor(x => x.Surname).NotEmpty();
    RuleFor(x => x.Forename).NotEmpty().WithMessage("Please specify a first name");
    RuleFor(x => x.Discount).NotEqual(0).When(x => x.HasDiscount);
    RuleFor(x => x.Address).Length(20, 250);
    RuleFor(x => x.Postcode).Must(BeAValidPostcode).WithMessage("Please specify a valid postcode");
  }
}
```

These validation techniques are often used the same way in a Web API, be that REST, GraphQL, or even GRPC. The API client, usually a UI application running on the browser, sends an HTTP request to the API to create or update a resource. The API then runs the request through validation to determine if the request contains any input that is outside of the accepted range. For example, imagine the case where a REST API exposes an endpoint, customers, the endpoint accepts an HTTP POST as a way to create new customer resources, now in this fictional API, all customers must have a valid email address. When the API receives the request to create a new customer, it must ensure that the email address is valid before the customer can be created. This presents a problem, the end-user using the UI app doesn't have immediate feedback. Meaning, the end-user had to probably fill out a registration form, the form is then submitted on behalf of the end-user, who must then wait for the API to validate the data, if there is a problem, then the UI is notified, which in turn alerts the user that there was a problem. Repeating the process until there are no more validation errors.

There is nothing necessarily wrong with doing validation as described above, but it can be better. The UI application can apply the same validation rules as the API. Thus avoiding sending a request to the API that will just fail with a validation error. Typically, the UI application will have obtained the validation rules through some form of documentation, like an [Open API](https://www.openapis.org/) definition file or word of mouth, the API developer informing the UI developer. I believe this approach to doing validation is the most common, at least it is the one I have encountered the most and it works. Especially if you are running a small system where the API developer and UI developer are the same developer or if your API only has one client.  

There are some drawbacks to doing validation as described above, that is when the API has many clients. Each client has to implement validation, and it must be done correctly. Should the API validation rules change, then so must each client, they each need to change their validation rules, which requires syncing up and redeploying each client application. Not an easy task, even if you yourself developed all the clients, it would still require a coordinated effort to update all clients. Some developers would argue that this is when API versioning should be done, true, but only in the case that the resource itself requires modification. The validation rules are what changed, not the underlying resource.

To summarize, our problems are as follows.

1. Client applications have to implement validation rules, often it involves the client copying the validation rules that the API uses.
2. Validation rules need to be more dynamic.
3. Major changes must be versioned.

How can these problems be solved, while arriving at a better API architecture?

To avoid the first problem, client applications duplicating validation rules, the rules themselves should come from the API. One of the beauties of the REST architectural style is that everything is centered around resources. What are resources? Well, just about anything can be a resource, even validation rules. By having the API exposes the validation rules as a resource, the UI application does not have to duplicate the validation rules. Instead, the API can provide the UI with the validation code, the UI can then execute the code to confirm the data is valid. Having the API provide the UI with code to execute is not a new idea. In fact, it is one of the key constraints in the REST architecture, [code on demond](https://en.wikipedia.org/wiki/Representational_state_transfer#Code_on_demand_(optional)). The idea is that the UI application is extended by having the API providing additional logic. This idea is not just limited to REST, there is no rule that prevents you from doing the same in GraphQL or GRPC.   

So, validation rules will be exposed as a resource. How? Well, most APIs are built with JSON, and [JSON Schema](https://json-schema.org/) can be used to described JSON, plus it allows us to provide human-readable documentation. That sounds familiar, describing data formats while providing human-readable messages is the key essence of data validation. JSON Schema is the perfect tool for API validation, the schema can be exposed as an API resource, the client application can then consume the schema, execute it and validate the data. This is perfect, JSON Schema solves all of the problems identified above. The client doesn't have to duplicate any validation rules. The schema resource can be versioned in case of a major change. The rules are now more dynamic and are easily changed.

I want to create an example to see how it all would work. 

A [while back](/Adding-Customer-Resource) I exposes a customers resource on my [Chinook](https://chinook-jsonapi.herokuapp.com/) [JSON:API](https://jsonapi.org/) project. The endpoint does not have any data validation, you can send a POST request with an empty firstName or lastName property and the API won't stop the request, it will fail due to a database constraint. I purposely skipped over validation when I created the endpoint since I figure I would eventually write this post and would tackle data validation at that time. 

I am going to modify the Chinook API project by exposing a JSON Schema resource that can be retrieved by a client app. The client application can then execute the schema on their end to confirm that the data being submitted adheres to the schema definition. For those unaware, here is how a customer is represented as of October 2021 on the Chinook API.

```json
{
  "data": {
    "type": "customers",
    "id": "1",
    "attributes": {
      "firstName": "Luís",
      "lastName": "Gonçalves",
      "company": "Embraer - Empresa Brasileira de Aeronáutica S.A.",
      "address": "Av. Brigadeiro Faria Lima, 2170",
      "city": "São José dos Campos",
      "state": "SP",
      "country": "Brazil",
      "postalCode": "12227-000",
      "phone": "+55 (12) 3923-5555",
      "fax": "+55 (12) 3923-5566",
      "email": "luisg@embraer.com.br"
    }
  }
}
```

Base on the Chinook database schema, at least the following validation should exist.

1. The field firstName cannot be null, must be no more than 40 characters.
2. The field lastName cannot be null, must be no more than 20 characters.
3. The field company, is an optional field, when provided, it cannot be more than 80 characters.
4. The field address, is an optional field, when provided, it cannot be more than 70 characters.
5. The field city, is an optional field, when provided, it cannot be more than 40 characters.
6. The field state, is an optional field, when provided, it cannot be more than 40 characters.
7. The field country, is an optional field, when provided, it cannot be more than 40 characters.
8. The field postalCode, is an optional field, when provided, it cannot be more than 10 characters.
9. The field phone, is an optional field, when provided, it cannot be more than 24 characters.
10. The field fax, is an optional field, when provided, it cannot be more than 24 characters.
11. The email field is required, must be no more than 60 characters, must be a valid email.

Now, time to write the JSON schema that will be used for validation. You could choose to write it by hand, but there are better ways. I find [stoplight](stoplight.io) to be the perfect tool to use to create REST APIs.

![Stoplight](/post/2021/a-better-web-api-validation-strategy/stoplight-vendor.PNG)

What is even better is that they support writing JSON schemas, they make it super simple, give it a resource model in JSON, they'll generate the model, then you'll be given the opportunity to annotate the model with validation rules. For example, I can take the customer model above and import it into their system.

![Model in Stoplight](/post/2021/a-better-web-api-validation-strategy/model-in-stoplight.PNG)

I'll add all the rules to the model, like which field is required, the minimum and maximum length of a field, a regex expression for the email field. That will generate the following JSON file.

```json
{
  "$schema": "https://json-schema.org/draft/2019-09/schema",
  "description": "Chinook customer model.",
  "type": "object",
  "properties": {
    "data": {
      "type": "object",
      "required": [
        "type",
        "attributes"
      ],
      "properties": {
        "type": {
          "type": "string",
          "minLength": 1
        },
        "attributes": {
          "type": "object",
          "required": [
            "firstName",
            "lastName",
            "email"
          ],
          "properties": {
            "firstName": {
              "type": "string",
              "minLength": 1,
              "maxLength": 40
            },
            "lastName": {
              "type": "string",
              "minLength": 1,
              "maxLength": 20
            },
            "company": {
              "type": "string",
              "minLength": 1,
              "maxLength": 80
            },
            "address": {
              "type": "string",
              "minLength": 1,
              "maxLength": 70
            },
            "city": {
              "type": "string",
              "minLength": 1,
              "maxLength": 40
            },
            "state": {
              "type": "string",
              "minLength": 1,
              "maxLength": 40
            },
            "country": {
              "type": "string",
              "minLength": 1,
              "maxLength": 40
            },
            "postalCode": {
              "type": "string",
              "minLength": 1,
              "maxLength": 10
            },
            "phone": {
              "type": "string",
              "minLength": 1,
              "maxLength": 24
            },
            "fax": {
              "type": "string",
              "minLength": 1,
              "maxLength": 24
            },
            "email": {
              "type": "string",
              "minLength": 1,
              "pattern": "^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~-]+@[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(?:\\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$",
              "maxLength": 60,
              "format": "email",
              "example": "yunier@hey.com"
            }
          }
        }
      },
      "additionalProperties": false
    }
  },
  "required": [
    "data"
  ]
}
```

> Info: You can see the schema in action by visiting [this](https://www.jsonschemavalidator.net/s/dFXvl5kF) link. Validation should be successful, to see a failed example, go to [this](https://www.jsonschemavalidator.net/s/Gcrdhha3) link. Validation failed in the second link because the id property should not be included when creating a resource. Since the id is determined by the database.

The question now becomes how to expose the JSON schema on the API. Base on section [9.5.1.1](https://json-schema.org/draft/2020-12/json-schema-core.html#rfc.section.9.5.1.1) of the JSON Schema Core [specification](https://json-schema.org/draft/2020-12/json-schema-core.html). JSON schemas should be exposed through a relationship link, using the [describedBy](https://www.w3.org/TR/powder-dr/#assoc-linking) link, see [IANA link relations page for more details](https://www.iana.org/assignments/link-relations/link-relations.xhtml). Which is great, linking from one resource to another is the essence of [hypermedia driven APIs](https://restfulapi.net/hateoas/) like the [Chinook API](https://chinook-jsonapi.herokuapp.com/) project. Now unfortunately, the Chinook API project implements [version 1.0](https://jsonapi.org/format/1.0/) of the [JSON:API](https://jsonapi.org/) specification. This version of JSON:API does not provide any guidance on how to use JSON:API with JSON Schema. Luckily for us, the [upcoming 1.1 version](https://jsonapi.org/format/1.1/) of JSON:API does provide us with guidance. Per the 1.1 specifications a JSON:API document may contain a [Links](https://jsonapi.org/format/1.1/#document-links) object as a [top level member](https://jsonapi.org/format/1.1/#document-top-level). Links object may contain a describedBy field, which is a link to a description document (e.g. OpenAPI or JSON Schema) for the current document. I am going to apply the recommendation made in version 1.1 of JSON:API to the Chinook API, even though Chinook implements version 1.0.

To keep everything simple, the schema will be hosted directly on the Chinook Project. You should know that there is a JSON Schema [store](https://www.schemastore.org/json/). The store can be used to host any JSON schema you defined or browser for JSON Schemas, for example, [appsettings.json](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-6.0#appsettingsjson-1), a file used by .NET Core can be [found](https://json.schemastore.org/appsettings.json) on the store.

After modifying the Chinook API project to support JSON Schema, the customer resource will now include the newly added describedBy link. See the response document below or visit [https://chinook-jsonapi.herokuapp.com/customers](https://chinook-jsonapi.herokuapp.com/customers) to see it in action.

```json
{
  "jsonapi": {
    "version": "1.0"
  },
  "links": {
    "self": "https://localhost:44323/customers",
    "up": "https://localhost:44323",
    "describedBy": "https://localhost:44323/customers/schemas"
  },
  "data": [
    // data omitted for brevity.
  ]
}
```

With this change, clients of the API can now retrieve a schema definition file that they can execute against data to confirm that the data captured by the client is valid. The API and the client will now be in sync with each other when it comes to validation. Should that validation change, then the validation resource would change, this is where versioning would become an important feature.

One last question remains, how should the client execute the schema? 

Well, that would depend on the client, JSON Schema is [supported in many languages](https://json-schema.org/implementations.html), find your language in their list of supported validators, and use the library that corresponds to your language to execute the schema. Most of you will probably be working in JavaScript, if so, consider using the library [ajv](https://github.com/ajv-validator/ajv). It is highly popular, blazing fast and should cover all your needs.