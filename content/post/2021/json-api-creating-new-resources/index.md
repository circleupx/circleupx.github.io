---
title: JSON:API - Creating New Resources
tags: [JSON:API, REST]
author: "Yunier"
date: "2021-08-08"
description: "Guide on how to create new resources"
series: ['JSON:API in .NET']
---

So far in my JSON:API series I've covered the [home resource](https://www.yunier.dev/2020-09-14-Adding-Home-Resource/), adding [your own resource](https://www.yunier.dev/2020-10-30-Adding-Customer-Resource/), adding an [exception handling middleware](https://www.yunier.dev/2020-10-19-Exception-Handling-Middleware/) and how to [expose relationship](https://www.yunier.dev/2020-12-06-Exposing-Relationships/) between resources. For the today's post, I would like to cover creating resources. I will update the chinook project by allowing [POST](https://datatracker.ietf.org/doc/html/rfc2616/#section-9.5) request on the customers collections to add new customers.

To get started, the customer controller needs to have a method that will accept the incoming POST request. I've decided to call the method **CreateCustomerResource**, the method will accept a [JSON:API document](https://jsonapi.org/format/#document-structure) from the request body. The full method signature is defined below.

```c#
[HttpPost]
[Route(CustomerRoutes.CustomerResourceCollection)]
public async Task<IActionResult> CreateCustomerResource([FromBody] Document jsonApiDocument)
{
    var document = await _customerResource.CreateCustomerResource(jsonApiDocument);
    return Created(document.SelfLink(), documnet);
}
```

Notice that the method has been decorated with the [HttpPost attribute](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.httppostattribute?view=aspnetcore-5.0) and it is using the same route as the customer resource collection. If no error are encountered, then API returns a [201 Created](https://datatracker.ietf.org/doc/html/rfc2616/#section-10.2.2) HTTP status code with a JSON:API document along with a [location header](https://datatracker.ietf.org/doc/html/rfc2616/#section-14.30) pointing to the location of the newly created resource. The helper function, [SelfLink](https://github.com/scott-mcdonald/JsonApiFramework/blob/47d970cae1297d4341bd7cdb1c992d35428ec34d/Source/JsonApiFramework.Core/JsonApi/GetLinksExtensions.cs#L133) is part of JsonApiFramework.

Next step, updating the customer resource class to take in the incoming JSON:API document from the controller so that it can be dispatched via Mediatr. The response from the mediatr handler is then used to create the response JSON:API document.

```c#
public async Task<Document> CreateCustomerResource(Document jsonApiDocument)
{

    var createdCustomerResource = await _mediator.Send(new CreateCustomerResourceCommand(jsonApiDocument));
    var currentRequestUri = _httpContextAccessor.HttpContext.GetCurrentRequestUri();

    using var chinookDocumentContext = new ChinookJsonApiDocumentContext(currentRequestUri);
    var document = chinookDocumentContext
        .NewDocument(currentRequestUri)
        .SetJsonApiVersion(JsonApiVersion.Version10)
            .Links()
                .AddSelfLink()
                .AddUpLink()
            .LinksEnd()
            .Resource(createdCustomerResource)
                .Relationships()
                    .AddRelationship(InvoiceResourceKeyWords.ToManyRelationShipKey, new[] { Keywords.Related })
                .RelationshipsEnd()
                .Links()
                    .AddSelfLink()
                .LinksEnd()
            .ResourceEnd()
        .WriteDocument();

    _logger.LogInformation("Request for {URL} generated JSON:API document {doc}", currentRequestUri, document);
    return document;
}
```

The Mediatr command class is also very simple, it accepts an incoming JSON:API document and stores it on a public field that can be access by the handler. Here is the class definition for the command class.

```c#
public class CreateCustomerResourceCommand : IRequest<Customer>
{
    public CreateCustomerResourceCommand(Document document)
    {
        Document = document;
    }

    public Document Document { get; }
}
```

Now, on to the handler. The job of the handler is to extract the new resource out of the JSON:API document, to apply any business rules/logic, and then to finally save the new resource on the database. The class definition for the handler is as follows.

```c#
public class CreateCustomerResourceHandler : IRequestHandler<CreateCustomerResourceCommand, Customer>
{
    private readonly ChinookDbContext _chinookDbContext;

    public CreateCustomerResourceHandler(ChinookDbContext chinookDbContext)
    {
        _chinookDbContext = chinookDbContext;
    }

    public async Task<Customer> Handle(CreateCustomerResourceCommand request, CancellationToken cancellationToken)
    {
        var jsonApiDocumentContext = new ChinookJsonApiDocumentContext(request.Document);
        var resource = jsonApiDocumentContext.GetResource<Customer>();

        _chinookDbContext.Customers
            .Add(resource);
            
        await _chinookDbContext
            .SaveChangesAsync(cancellationToken);

        return resource;
    }
}
```

Nothing to exciting, the resource data is extracted out of the JSON:API document and it gets attached to EF Core, which then saves it to the database. Simple stuff, it will get a little more complicated later, but for now this will work. Time to test our code and what better tool than POSTMAN to do the test.

You should know that POSTMAN comes with a CLI, [newman](https://github.com/postmanlabs/newman), a great option for executing test written in POSTMAN on your CI/CD pipeline. See [this](https://learning.postman.com/docs/running-collections/using-newman-cli/command-line-integration-with-newman/) blog post more details.

I'll go ahead an open up postman. I'm going to add new API request of type POST, using [https://chinook-jsonapi.herokuapp.com/customers](https://chinook-jsonapi.herokuapp.com/customers) as the request URL. Under the headers tab, I will add a new header, content-type, and set the value to [application/vnd.api+json](https://www.iana.org/assignments/media-types/application/vnd.api+json), since this is required by JSON:API. Next, the request body will use the following json as the request body.

```json
{
    "data": {
        "type": "customers",
        "attributes": {
            "firstName": "$randomFirstName",
            "lastName": "$randomLastName",
            "company": "$randomCompanyName",
            "address": "$randomStreetAddress",
            "city": "$randomCity",
            "state": "$randomCountryCode",
            "country": "$randomCountry",
            "postalCode": "$randomCountryCode",
            "phone": "$randomPhoneNumber",
            "fax": "$randomPhoneNumber",
            "email": "$randomEmail"
        }
    }
}
```

The postman syntax includes opening and closing brackets. They were excluded above because Jekyll, the engine that powers this blog interprets them as empty strings, so it will not render them on the page.

I am providing all attributes here since there are no validation or rules yet. Note the use of postman's dynamic variable syntax. Postman uses [faker.js](https://github.com/Marak/Faker.js) under the hood to generate this random data. Do forgive me for using random country code for the state property. Didn't feel like making a helper function that generates a random state.

.NET has a copy of faker called [Bogus](https://github.com/bchavez/Bogus), an excellent library to use in your Unit/Integration test whenever you need to generate data. You could even it use it to seed a test database.

When I execute the POSTMAN request I get a 201 Created as the response code with the newly created user on the response body. You can execute the test yourself by pulling the chinook repository down and importing the test into POSTMAN. All tests are located under the [test folder](https://github.com/circleupx/Chinook/tree/master/test).

Great, so the API now supports creating new customers. All is great in the world, well, almost all. See what we have here is the most basic example, it is simple and easy, starting with the fact that in this sample app there are no validation or business rules, but what really complicates thing is having resources like customers, that have related resources.

The customer resource we just created did not have any relationships, meaning the customer being created did not have any invoices. There might be instances where a new customer has one or many invoices, to support this type of request the API should be enhanced to support adding new invoices, the link between the new customer and the new invoice can be established by including the relationship on the HTTP request body for the customer resource or vice versa. The response body of such request may look like the following JSON document.

```json
{
    "data": {
        "type": "customers",
        "attributes": {
            "firstName": "john",
            "lastName": "smith",
            "company": "Auth0",
            "address": "1234 Sesame Street",
            "city": "El Dorado",
            "state": "NY",
            "country": "United States of America",
            "postalCode": "33543",
            "phone": "999-871-0900",
            "fax": "345-987-7890",
            "email": "ssmith@auth0.com"
        }
    },
    "relationships": {
        "invoices": {
            "data": { 
                "type": "invoices", 
                "id": "9" 
            }
        }
    }
}
```

Another option would be to support [sideposting](https://github.com/json-api/json-api/issues/1216), that is being able to create multiple resources of different types in a single request so that the client application doesn't have to send multiple POST request for every new resource it needs to link. This is where JSON:API [extensions](https://jsonapi.org/extensions/) for [atomic operations](https://jsonapi.org/ext/atomic/) come into play. It establishes a contract on how the client should tructure a request that contains multiple resource and how the server should handle these type of request.

The release of JSON:API v1.1 should simplify side posting due to the introduction of [lid](https://github.com/json-api/json-api/pull/1244). An lid is an id generated on the client, this id is then used to link resources on a client document. Remember the great thing about JSON:API is that it is a [wire protocol for incrementally fetching and updating a graph over HTTP](https://youtu.be/LLe7Fi-wM3Q?t=922). The keywords here being updating a graph, by combining lid with resource linkage a client can create a JSON:API documents with a complex hierarchy, for example.

```text
 POST /customer HTTP/2.0
 Content-Type: application/vnd.api+json"
 Accept: application/vnd.api+json

{
	"data": {
		"lid": "1",
		"type": "customers",
		"attributes": {
			"firstName": "john",
			"lastName": "smith",
			"company": "Auth0",
			"address": "1234 Sesame Street",
			"city": "El Dorado",
			"state": "NY",
			"country": "United States of America",
			"postalCode": "33543",
			"phone": "999-871-0900",
			"fax": "345-987-7890",
			"email": "ssmith@auth0.com"
		},
		"relationships": {
			"invoices": {
				"data": {
					"type": "invoices",
					"lid": "9"
				}
			}
		},
		"included": [{
			"type": "invoices",
			"lid": "9",
			"attributes": {
				"invoiceDate": "MjAwOS0wMS0wMSAwMDowMDowMA==",
				"billingAddress": "Theodor-Heuss-Straße 34",
				"billingCity": "Stuttgart",
				"billingState": null,
				"billingCountry": "Germany",
				"billingPostalCode": "70174",
				"total": "MS45OA=="
			}
		}]
	}
}

```

In the example above, the included resource, invoices, could also have a relationship member that links to another resource on the included array, and that resource could also have a relationship member that links to another resource on the included array and so on. When the server receives this JSON:API document, it can create all those resources on the database and establish relatioships as defined on the client document.
