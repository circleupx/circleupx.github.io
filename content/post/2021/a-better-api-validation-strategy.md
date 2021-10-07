---
title: A better API validation strategy
tags: [REST, GraphQL]
draft: true
---

As an API developer you will eventually need to determine how to handle data validation. 

The .NET ecosystem offers a few options, the first option, [validation attributes](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/validation?view=aspnetcore-6.0#validation-attributes), can be used to annotate how a model should be validated. Validation attributes are great, they don't require any external dependencies, you can specify error messages, create your own [custom validator](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/validation?view=aspnetcore-6.0#validation-attributes), validate against many data types. For example,

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

Another popular option in .NET is using the library [Fluent Validation](https://fluentvalidation.net/), this library allows you define validation rules that are then applied against a data class. Fluet validation offers the same capabilities as validation attributes, with one additional benefit, using the Fluent Validation library means that your data models are kept clean. Since the validation rules are defined in a configuration class. For example,

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

These validation techniques are often used the same way. The API client, UI app running on the browser, sends a request to the API to create or update a resource. The API then runs the request throught validation to determine if the request contains any input that is outsided of the accepted range. For example, imagine the case where a REST API exposes an endpoint, customers, this endpoint accepts an HTTP POST as a way to create customer, now in this fictional API, all customers must have a valid email address. When the API receives the request to create a new customer, it must ensure that the email address is valid before the customer can be created. This presents a problem, the end-user using the UI app doesn't have immediate feedback. Meaning, the end-user had to probably fill out a form, submit it, wait for the API to validate the data, if there is a problem, then the UI is notified, which in turn alerts the user that there was a problem. 

For example,

There is nothing necessarily wrong doing validation as described above, but it can be better. The UI application can apply the same validation rules as the API. Thus avoiding sending a request to the API that will just fail with a validation error. Typically, the UI will have obtained the validation rules from some form of documentation, like an [Open API](https://www.openapis.org/) definition file or word of mouth, the API developer informing the UI developer. I believe this approach to doing validation is the most common, at least it is the one I have encountered the most and it works. Specially if you are running a small system where the API developer and UI developer are the same developer or if your API only has one client.  

For example,

There are some drawbacks to doing validation as described above comes when the API has many clients. Since each client will need to implement validation and implemented it correctly. For example, if the API validation rules change, say that our imagenary customer API now requires each customer's email to be, then each client will need to update their validation rules. Not an easy task, even if you yourself developed all the client, it would still require a coordinated effort to update all clients. Some developers would argue that this is when API versining should be done, true, but only in the case that the resource itself requires modification. The validation rules are what changed, not the underlying resource.

To summarize, our problems are as follows.

1. Client application have to implement validatio rules, often being an exact copy of the API's validation rules.
2. Validation rules need to be more dynamic.
3. Major changes should be versioned.

How we tackle all of these problem and arrive at a better API architecture?

To avoid the first problem, client applications duplicating validation rules, the rules themselves should come from the API. One of the beauties of the REST is that everything is centered around resources. What are resources? Well, just about anything can be resource, even validation rules. By having the API exposes the validation rules as a resource, the UI application does not have to duplicate the validation rules. Instead the API can provide the UI with the validation code, the UI can then execute the code to confirm the data is valid. Having the API provide the UI with code to execute is not a new idea. In fact, it is one of key constraints in the REST archicture, [code on demond](https://en.wikipedia.org/wiki/Representational_state_transfer#Code_on_demand_(optional)). The idea being that the UI application be extended by having the API providing additional logic. This idea is not just limited to REST, there is no rule that prevents you from doing the same in GraphQL, or GRPC.   

OK, so we'll have the validation rules exposed as a resource. How? Well, most API are built with JSON, and [JSON Schema](https://json-schema.org/) can be used to described data format, plus it allows us to provide human readable documentation. That sounds familiar, describing data formarts while providing human readable messages its the key essence of data validation. JSON Schema is the perfect tool for API validation, it can be exposed as an API resources, the client application can then consume the schema script, execute it and validate the data. 

This is perfect, JSON Schema solves all of our problems, the client doesn't have to duplicate any validation rules, the JSON Schema resource can be versioned in case of major change. It also means our rules are now more dynamic and be easily changed.

OK, let's create an example to see how it all would work. 

A [while back](/Adding-Customer-Resource) I exposes a customers resource on my [Chinook](https://chinook-jsonapi.herokuapp.com/) JSON:API project. The endpoint does not have any data validation, I purposely skipped over it since I figure I would eventually write this post and would tackle data validation at the time. I am going to modify the customers resource by exposing a JSON Schema resource that can be retrieve by a client app. Here is a sample customer,

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

What are some rules we can extract from model above? Base on the Chinook database schema, I know the following.

1. The field firstName cannot be null, must be no more than 40 characters.
2. The field lastName cannot be null, must be no more than 20 characters.
3. The field company, is an optional field, when provided, it cannot be more than 80 characters.
4. The field address, is an optional field, when provided, it cannot be more than 70 characters.
5. The field city, is an optional field, when provided, it cannot be more than 40 characters.
6. The field state, is an optional field, when provided, it cannot be more than 40 characters.
7. The field country, is an optional field, when provided, it cannot be more than 40 characters.
8. The field postalCode, is an optional field, when provided, it cannot be more than 10 characters, it can have a dashes and spaces.
9. The field phone, is an optional field, when provided, it cannot be more than 24 characters, it can have a dashes, spaces, and parenthesis.
10. The field fax, is an optional field, when provided, it cannot be more than 24 characters, it can have a dashes, spaces, and parenthesis.
11. The email field is required, must be no more than 60 characters, must be a valid email.
