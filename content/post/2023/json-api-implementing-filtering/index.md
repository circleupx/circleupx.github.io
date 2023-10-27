---
title: JSON:API Implementing Filtering
tags: [JSON:API]
author: "Yunier"
date: "2023-10-15"
description: "Implementing a Filtering Strategy in JSON:API"
series: [JSON:API In .NET]
---

### Introduction

It has been over [a year since I last wrote](/post/2022/json-api-pagination-links/) about JSON:API, since then the [team](https://jsonapi.org/about/#editors) behind JSON:API has published version 1.1 of the JSON:API specification. I would like to continue my journey of documenting JOSN:API in .NET by introducing a really cool feature to my [Chinook JSON:API](https://github.com/circleupx/Chinook) project, filtering.

The first thing to know about filtering in JSON:API is that the spec itself [is agnostic](https://jsonapi.org/format/#fetching-filtering) to any filtering strategies. Meaning it is up to you to define how filtering should be handled by your API. In my opinion, this has always been a drawback of the JSON:API spec, I believe in that it would have been a better choice for the spec if it had decided on a filtering strategy, but that is discussion for another day. While the spec does not favor any filtering strategy it does have some [recommendations](https://jsonapi.org/recommendations/#filtering).

The spec recommends using the LHS bracket syntax to denote which resource the filtering should be applied to. For example, imagine you are dealing with a post resource and each post resource can expose a relationship to an author resource collection, that is to say, each post has a 1 to many relationship with with the authors resource.

As a client of the API if you may want to find out which post has an author with a firstName of "Dan", typically what you see in most APIs is that you would first need to filter the authors resource to only those that have "Dan" as a first name, then once you have those resources, you can filters the posts resource to only post that have a matching author's resource. This type of filtering is not ideal even though it is how many REST APIs out in the wild are implemented, this is the well-know problem of over-fetching and under-fetching, a selling point of GraphQL.

We can do better, by properly defining the relationships between our resources a client application should be able to make the following request to handle the scenario I just described.

```shell
GET /posts?filter[post]=published eq true&filter[author]=firstName eq 'Dan'&include=author HTTP/1.1
```

Note the usage of LSH brackets and ODATA syntax, we'll talk about that later on this post. If the request above is valid, then it should in theory yield the following JSON:API response.

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

In a single request, the client has requested that the API should get all posts where the published field is true and to include the all the related authors, the request also states that out of that list of posts, the API should only return posts where the author is name Dan. Using a nested filtering allows a client of the API to overcome the over-fetching and under-fetching problems that many REST APIs have.

So, how can we implement this in feature in .NET? Let's take a look.

The first step in implementing filtering will be to take the filter query parameter from the incoming HTTP request URL and tokenize them. You have the option to implement a custom tokenizer, see my [Parsing in C#](/content/post/2021/parsing-in-csharp/) blog post, or use one of the many awesome parsing libraries that exit in .NET. Personally, I have always relied on [SuperPower](https://github.com/datalust/superpower) as it easily allows you define a tokenizer and parsers. Once we have a tokenizer and a parser, the next step is to build an [Abstract Syntax Tree](https://www.youtube.com/shorts/mi6DoxNEN6w) the generate runtime expression that can be then be given to an ORM system like EF Core or Dapper.

By the way, if you are interesting in learning more about parsers then you can enroll in [Building a Parser from scratch](http://dmitrysoshnikov.com/courses/parser-from-scratch/) by [Dmitry Soshnikov](http://dmitrysoshnikov.com/).

Let's review the Chinook project I have been building for the last two years, as I mentioned before, it is a level 3 REST API that implements the JSON:API specification. The API exposes a number of resources, one of them, the invoices resource, has a one to one relationship to the customer resource, a single customer resource is represented with the following JSON payload.

```JSON
{
    "jsonapi": {
        "version": "1.0"
    },
    "links": {
        "self": "https://localhost:5001/invoices/98/customer",
        "up": "https://localhost:5001/invoices/98"
    },
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
        },
        "links": {
            "self": "https://localhost:5001/customers/1"
        }
    }
}
```

### The Problem

Back to the Chinook project, imagine the following scenario, a client of the Chinook API would like to query the API to find all invoices where the billing country is Germany **and** the customer, which is a related resource of invoices has a first name equal to **Leonie**, as it stands, a client of the Chinook API today could do it with the following actions.

1) Navigating to the invoice resource collection, pulling all records in-memory, about 412 in total.
2) Loop through the invoice collection to find any invoice where the billing country is Germany.
3) For any invoice where the billing country is Germany, navigate to the related customer.
3) Check if the customer's first name is Leonie.

What I have just described is totally inefficient, as I mention before this is a well-known problem, [Over Fetching And Under Fetching](https://nordicapis.com/what-are-over-fetching-and-under-fetching/) and it often sighted as the main reason to use GraphQL.

In other to provide better usability and developer experience I will introduce resource filtering to the Chinook project, the invoice resource collection will now allow a client to specify a filtering criteria. This filtering criteria can work against the invoice resource collection as well as any related resources but for this purposes of this demo I will only add filtering support for the related customer resource.

Adding filtering to an API means that you will need to come up with a filtering language, for the purposes of this blog post I am going to stick to OData, why? Simple, it is well known standard that many developers are already familiar with and comes with well defined operators, you can of course come up with your own if desired.

#### OData Operators

Let's take a look at the operators offered by OData as defined in [Built-in Filter Operations](https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html#sec_BuiltinFilterOperations)

The filter operators offered by OData are as follows.

| Comparison Operators |                       |                                       |
|----------------------|-----------------------|---------------------------------------|
| eq                   | Equal                 | Address/City eq 'Redmond'             |
| ne                   | Not equal             | Address/City ne 'London'              |
| gt                   | Greater than          | Price gt 20                           |
| ge                   | Greater than or equal | Price ge 10                           |
| lt                   | Less than             | Price lt 20                           |
| le                   | Less than or equal    | Price le 100                          |
| has                  | Has flags             | Style has Sales.Color'Yellow'         |
| in                   | Is a member of        | Address/City in ('Redmond', 'London') |
| Logical Operators    |                       |                                       |
| and                  | Logical and           | Price le 200 and Price gt 3.5         |
| or                   | Logical or            | Price le 3.5 or Price gt 200          |
| not                  | Logical negation      | not endswith(Description,'milk')      |
| Arithmetic Operators |                       |                                       |
| add                  | Addition              | Price add 5 gt 10                     |
| sub                  | Subtraction           | Price sub 5 gt 10                     |
| mul                  | Multiplication        | Price mul 2 gt 2000                   |
| div                  | Division              | Price div 2 gt 4                      |
| divby                | Decimal Division      | Price divby 2 gt 3.5                  |
| mod                  | Modulo                | Price mod 2 eq 0                      |
| Grouping Operators   |                       |                                       |
| ( )                  | Precedence grouping   | (Price sub 5) gt 10                   |


OData also offers [Built-in Query Functions](https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html#sec_BuiltinQueryFunctions) but those functions are beyond the scope of this blog post.

In order for the a client of the Chinook API to get find an invoice where the billing country is Germany and the related customer's first name is Leonie the client would have to use the following query string.

```bash
GET /invoices?filter[invoices]=billingCountry eq 'Germany'&filter[customers]=firstName eq 'Leonie'&include=invoices HTTP/1.1
```

Let's break down the request above, **/invoices** is the resource collection we are dealing with, then we have the query parameters, the first one, the JSON:API keyword filter, is used by the client to inform the server that the resource should be filtered, then we have **[invoices]**, this is used to inform the server that the filtering will be done against the resource invoices, remember JSON:API uses compound documents, so you can have a JSON:API document with multiple resources, we then use **=billingCountry eq 'Germany'** to inform the server that the filter is against the property billingCountry on the source invoices, with OData syntax, **eq** as mention above being used as the equals operators, then since invoice billingCountry is a string type we use single quotes to specify the value. The second filter, is against the customers resource, against the firstName property, using the eq operator from OData to inform the server to filter the resource to only customer that have a first name of Leonie.

A quick aside, JSON is case sensitive, you can encounter APIs in different case formats, i.e. snake vs camel, therefore, when doing filtering, take into consideration the casing being used on the field you plan to filter one.

Now that we know what the HTTP request will look like let's switch to the Chinook API and add the code needed to support filtering.


First thing I am going to do is to install Superpower on the Chinook Core Project by running the following command.

```shell
dotnet add package Superpower --version 3.0.0
```

Now that Superpower is installed I will modify the existing UriKeyWords class that was added in [JSON:API - Pagination Links](/post/2022/json-api-pagination-links/).

```C#
public static class UriKeyWords
{
    public static string PageNumber = $"page[{number}]";
    public static string PageSize = $"page[{size}]";
    public const string size = nameof(size);
    public const string number = nameof(number);
}
```

Here is what the class looks like after adding the OData operators.

```C#
public static class UriKeyWords
{
    public static string PageNumber = $"page[{number}]";
    public static string PageSize = $"page[{size}]";
    public const string size = nameof(size);
    public const string number = nameof(number);

    public const string And = "and";
    public const string Or = "or";
    public const string Eq = "eq";
    public const string Nq = "nq";
}
```

Note the addition of some of the OData operators we previously defined, I kept the list small since we don't need to support all OData operators at the moment. Next, for each operator type that you plan to support you will need to add a corresponding token, represent as an enum value. For example, for the "eq" operator types which falls under the equality operator you would have the following enum.

```c#
[Flags]
public enum ODataTokens
{
  [Token(Category = "OData Equality Operator", Example = "Eq, Nq, Gt Lt", Description = "Equality operators supported by API.")]
  EqualityOperator = 0,
}
```

Adding the rest of the support operators yields the following ODataTokens class.

```C#
[Flags]
public enum ODataTokens
{
    [Token(Category = "Empty value", Example = "Empty string or null", Description = "Represents no value.")]
    None = 0,
   
    [Token(Category = "String value", Example = "'Hello World'", Description = "Parsed string value.")]
    StringValue = 1,
   
    [Token(Category = "Boolean Expression", Example = "True")]
    BooleanValue = 2,
   
    [Token(Category = "Object Field", Example = "payment.CreditCardNumber", Description = "An object property")]
    ObjectField = 4,
   
    [Token(Category = "Logical Operator", Example = "And, Or")]
    LogicalOperator = 8,
   
    [Token(Category = "Equality Operator", Example = "Eq, Nq, Gt Lt", Description = "All equality operators supported")]
    EqualityOperator = 16,
   
    [Token(Category = "Number value", Example = "1", Description = "Parsed number value.")]
    IntegerValue = 32
}
```

Up next, we need to build our actual OData parsers with Superpower, let's gets started with the most basic, an OData string. As shown before, an OData request may look like the following HTTP GET request.

```shell
GET /posts?filter[post]=published eq true&filter[author]=firstName eq 'Dan'&include=author HTTP/1.1
```

#### OData Parser

Note the usage of **'** to denote when a string starts and ends, this is what the we need to parse and Superpower allows us to build a parser with LINQ, the following LINQ expression can be used by Superpower to parse an OData string.

```C#
public static class ODataParser
{
    private static char SingleQuote => '\'';
    private static readonly char[] InvalidStringCharacters = {'*', '=', '+', '$', '#', '~', '`', '\'', ' '};

    public static TextParser<string> ODataString =>
    from start in Character.EqualTo(SingleQuote)
    from chars in Content
    from end in Character.EqualTo(SingleQuote)
    select chars;

    public static TextParser<char> Characters =>
    from c in Character.ExceptIn(InvalidStringCharacters)
    select c;

    public static TextParser<string> Content =>
    from content in Characters.Many()
    select new string(content);
}
```

Here we start seeing the beauty of parser combinators like Superpower, you can compose parsers out of other parsers. In the code above, the ODataString parser is composed by the Content parser and the Characters parser.

Now that we have our tokens and parses, it is time to create our [Tokenizer](https://www.youtube.com/watch?v=4m7ubrdbWQU). For the tokenizer we can use the built in Tokenizer provided by Superpower, along with their helpers functions, like Match, Ignore, Build and so on. Below is how the Tokenizer looks so far, note the usage of the key words defined in ODataParser and the Enum ODataTokens.

#### Tokenizer

```C#
public static class Tokenizers
{
    public static TokenizerBuilder<ODataTokens> GetODataTokenizer()
    {
        var tokenizerBuilder = new TokenizerBuilder<ODataTokens>()
            .Ignore(Span.WhiteSpace)
            .Match(ODataParser.ODataEqualOperator, ODataTokens.ComparisonOperator)
            .Match(ODataParser.ODataNotEqualOperator, ODataTokens.ComparisonOperator)
            .Match(ODataParser.ODataGreaterThanOperator, ODataTokens.ComparisonOperator)
            .Match(ODataParser.ODataLessThanOperator, ODataTokens.ComparisonOperator)
            .Match(ODataParser.ODataGreaterThanOrEqualOperator, ODataTokens.ComparisonOperator)
            .Match(ODataParser.ODataLessThanOrEqualOperator, ODataTokens.ComparisonOperator)
            .Match(ODataParser.ContainsOperator, ODataTokens.ComparisonOperator)
            .Match(ODataParser.ODataString, ODataTokens.StringValue)
            .Match(ODataParser.ODataLogicalAnd, ODataTokens.LogicalOperator)
            .Match(ODataParser.ODataLogicalOr, ODataTokens.LogicalOperator)
            .Match(ODataParser.ODataFalse, ODataTokens.BooleanValue)
            .Match(ODataParser.ODataTrue, ODataTokens.BooleanValue)
            .Match(Numerics.IntegerInt32, ODataTokens.IntegerValue);
        return tokenizerBuilder;
    }
}
```

Something worth mentioning here, Superpower has a good support for error handling, when an error is encountered Superpower will report that error and you can set how to handle it, I won't cover error handling of the tokenizer on this blog post as I feel it is beyond the scope of this post, but essentially what you would do is take the error reported by Superpower and convert it into an [Errors Document](https://jsonapi.org/format/#errors).

Next, I will add a Tokenizer Builder, the builder will take expose method that take in a generic type, TObject, inspect the properties in this generic type and tokenize them along with the OData tokenizer above. Here is the tokenizer builder.

```C#
internal class TokenizerBuilder
{
    private static readonly BindingFlags Flags = BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance;

    /// <summary>Tokenize an array of properties using reflection. Ignores casing</summary>
    /// <typeparam name="TObject">Object from which the properties will be extracted.</typeparam>
    /// <returns>Tokenize list of <see cref="TObject"/> properties </returns>
    internal static Tokenizer<ODataTokens> TokenizeObjectProperties<TObject>()
    {
        var tokenizer = Tokenizers.GetODataTokenizer();
        var mappedPropertiesToToken = MapPropertiesToToken<TObject>(tokenizer);
        return mappedPropertiesToToken.Build();
    }

    private static TokenizerBuilder<ODataTokens> MapPropertiesToToken<TObject>(TokenizerBuilder<ODataTokens> tokenizer)
    {
        IEnumerable<string> properties = typeof(TObject)
            .GetProperties(Flags)
                .Select(propertyInfo => propertyInfo.Name)
                    .Distinct();

        foreach (var propertyName in properties)
        {
            tokenizer.Match(Span.EqualToIgnoreCase(propertyName), ODataTokens.ObjectField);
        }

        return tokenizer;
    }
}
```

With the code above, when I call the method TokenizeObjectProperties and use Invoices as the generic type, all the properties that currently exist in the Invoices class will get tokenized.

Taking an incoming query string, tokenize it, parse it, building an AST, then finally getting an expression to pass to an ORM like EF Core or Dapper takes a look of work, whenever I have face this before I have relied on the builder pattern, and since what I am building is rather complex, the builder pattern allows me delegate different parts of the process to small parts of the builder.

#### Building a DSL

The one thing I want to ensure is that on top of the builder pattern there needs to be a [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) to ensure proper usage of the builder pattern. For example, I want to expose the following DSL.

```c#
public class DSL
{
   public static string BuildDSL()
   {
      var displayUrl = _httpContextAccessor.HttpContext.GetRequestUri();
      var resource = ResourceQueryBuilder
      .NewResourceQuerySpecification(displayUrl)
        .StartFilter()
          .AddFilter<Invoice>()
        .EndFilter()
      .BuildSpecification();
   }
}
```

In the DSL above, ordering of operations can be controlled, I want to shield the any consumers from incorrectly tokenizing, parsing, building the AST and finally the expression, a DSL works create for this as the fluent style API controls the way the final result can be assembled. [How to create a fluent interface in C#
](https://scottlilly.com/how-to-create-a-fluent-interface-in-c/) by [Scott Lilly](https://twitter.com/scottlilly) can probably explain the benefits of this type of code better than I can, if you can, I do recommend reading it.

The DSL I built has support for pagination, ordering, but for now we are only interesting in filtering. As you can see there is a AddFilter method that accepts a generic type, the Invoice class in this case, this filtering method is responsible for a good chuck of the work I have described so far. Let's take a deeper look at this method.

```c#
public sealed class ResourceQueryBuilder
{
  public IAddResourceFiltering AddFilter<TResource>()
  {
     var tokenizer = TokenizerBuilder.TokenizeObjectProperties<TResource>();
     var parsedQueryStringDictionary = _queryParameterService.ParseFilterQueryString();
     var resourceType = typeof(TResource);
  }
}
```

The first part of the AddFilter method in the ResourceQueryBuilder class is to call the tokenizer as shown in the code above to tokenize the properties on the resource, the next step is to call ParseFilterQueryString in the QueryParameterService class, this class was introduce in [JSON:API - Pagination Links](/post/2022/json-api-pagination-links/) as UriQueryParametersReader I just consolidated the reader and writer into a service. The new method, ParseFilterQueryString, was added in to the class due to a limitation in [JsonApiFramework](https://github.com/scott-mcdonald/JsonApiFramework), which powers the Chinook project was never built to handle multiple query filters.

```c#
public class QueryParameterSerive
{
    public Dictionary<string, string> ParseFilterQueryString()
    {
        var filterDictionary = new Dictionary<string, string>();
        if (string.IsNullOrWhiteSpace(_requestUri.Query))
        {
            return filterDictionary;
        }

        var requestUriQuery = _requestUri.Query;
        var startIndex = requestUriQuery[(requestUriQuery.IndexOf(UriKeyWords.QuerySeparator) + 1)..];
        var startIndexCopy = startIndex;

        foreach (var filter in startIndexCopy.Split(UriKeyWords.Ampersand))
        {
            var array = filter.Split(UriKeyWords.Equal);
            var filterKey = Uri.UnescapeDataString(array[0]);
            var filterValue = Uri.UnescapeDataString(array[1]);
            var filterKeySplit = filterKey.Split(UriKeyWords.LeftBracket, UriKeyWords.RightBracket);

            if (filterKeySplit.Length == 3 && filterKeySplit[2] == string.Empty)
            {
                var filterQueryString = filterKeySplit[0];
                var filterQueryStringValue = filterKeySplit[1];
                if (string.Compare(filterQueryString, UriKeyWords.Filter, StringComparison.OrdinalIgnoreCase) == 0)
                {
                    filterDictionary[filterQueryStringValue] = filterValue;
                }
            }
        }

        return filterDictionary;
    }
}
```

With the code above the API can now handle multiple query parameters being used in the URL query string. Back to the ResourceQueryBuilder class, the next step is to get the resource type that was used to call AddFilter. This is essential as we need to know which service model we need to use when building our expressions.


```c#
public sealed class ResourceQueryBuilder
{
  public IAddResourceFiltering AddFilter<TResource>()
  {
      var resource = parsedQueryStringDictionary.FirstOrDefault(x => x.Key.Camelize() == resourceType.Name.Camelize().Pluralize());
      if (resource.Key is null)
      {
        keyValuePairs.Add(resourceType.Name, PredicateBuilder.New<TResource>());
        return this;
      }
  }
}
```

The next part of the AddFilter method is looking at the parsed query strings values return from ParseFilterQueryString to see if the current resource matches up with a filter used in the URL query string. If no matched then we add a default expression to a global dictionary that is keeping track of all the filters being applied.

#### PredicateBuilder

Notice the usage of the PredicateBuilder class, this is a helper class that allows us to work with expressions, the new method creates a starting expression that evaluates to true, essentially it creates the following code.

```c#
public Expression<Func<Customer, bool>> FilterExpression { get; } = entity => true;
```

As mentioned in [JSON:API - Pagination Links](/post/2022/json-api-pagination-links/), the code above is great because the Linq Provide will see this code and do nothing, in other words, this is a good default value to have when the client doesn't specify any filters.

Speaking of the PredicateBuilder, this is a [well-known class](https://www.albahari.com/nutshell/predicatebuilder.aspx) that has been used over the year by many developers working with C# expression, most notably, this is what powers the [LinqKit Project](https://github.com/scottksmith95/LINQKit).

The final part of the AddFilter method to loop through each node that was tokenized, then to use the visitor patter to visit each node grabbing the value and composing an expression, here is the code.

```c#
public sealed class ResourceQueryBuilder
{
    public IAddResourceFiltering AddFilter<TResource>()
    {
        var result = tokenizer.TryTokenize(resource.Value);
        var tokenList = result.Value;

        var visitor = new ExpressionTreeVisitor<TResource>();
        foreach (var token in tokenList)
        {
            switch (token.Kind)
            {
                case ODataTokens.None:
                    break;
                case ODataTokens.StringValue:
                    VisitConstantNode(token, visitor);
                    break;
                case ODataTokens.BooleanValue:
                    VisitConstantNode(token, visitor);
                    break;
                case ODataTokens.ObjectField:
                    VisitObjectField(token, visitor, resourceType);
                    break;
                case ODataTokens.LogicalOperator:
                    VisitLogicalNode(token, visitor);
                    break;
                case ODataTokens.ComparisonOperator:
                    VisitComparisonNode(token, visitor);
                    break;
                case ODataTokens.IntegerValue:
                    VisitConstantNode(token, visitor);
                    break;
            }
        }

        var filter = visitor.GetFilterExpression();
        keyValuePairs.Add(resourceType.Name, filter);
        return this;
    }
}
```

In the code above we call the TryTokenize method in the tokenizer to tokenize the query parameter string, if successful, we loop through the list then begin to visit each known using the [Visitor Pattern](https://dofactory.com/net/visitor-design-pattern). After each node is visited an [expression factory](https://github.com/circleupx/Chinook/blob/master/src/Chinook.Core/Factories/ExpressionFactory.cs) is responsible for putting together a complete C# expression which is returned by the method [GetFilterExpression](https://github.com/circleupx/Chinook/blob/master/src/Chinook.Core/Visitors/ExpressionTreeVisitor.cs#L14C56-L14C56).


Now that we have an expression that can be given to EF Core all that I need to do is modify the [InvoiceResource](https://github.com/circleupx/Chinook/blob/master/src/Chinook.Web/Resources/InvoiceResource.cs) class and the [GetInvoiceResourceCollectionHandler](https://github.com/circleupx/Chinook/blob/master/src/Chinook.Infrastructure/Handlers/GetInvoiceResourceCollectionHandler.cs) to accept the output of the [ResourceQueryBuilder](https://github.com/circleupx/Chinook/blob/master/src/Chinook.Core/Builders/ResourceQueryBuilder.cs) which is a [specification](https://enterprisecraftsmanship.com/posts/specification-pattern-c-implementation/).


Here is the structure of the specification that is generated by the ResourceQueryBuilder class.

```c#
public class ResoureQuerySpecification
{
    private Dictionary<string, dynamic> FilterDictionary { get; } = new Dictionary<string, dynamic>();
    private List<dynamic> Includes { get; } = new List<dynamic>();
    private dynamic OrderBy { get; set; }
    private dynamic OrderByDescending { get; set; }
    private dynamic GroupBy { get; set; }

    public dynamic GetFilter<T>(string key)
    {
        var hasValue = FilterDictionary.TryGetValue(key, out dynamic filter);
        if (hasValue)
        {
            return filter;
        }
        else
        {
            return PredicateBuilder.New<T>();
        }
    }

    public void AddFilter(string key, dynamic value)
    {
        FilterDictionary.TryAdd(key, value);
    }
}
```

And here is the code required to add filtering support to the Invoice resource.

```c#
var displayUrl = _httpContextAccessor.HttpContext.GetRequestUri();
var resource = ResourceQueryBuilder
.NewResourceQuerySpecification(displayUrl)
    .StartFilter()
        .AddFilter<Invoice>()
        .AddFilter<Customer>()
    .EndFilter()
.BuildSpecification();
```

As I said, a DSL is super useful here, ideally, this is the only code developer of the API would use to add filtering, and in the future pagination, includes and ordering.

Back to the modified GetInvoiceResourceCollectionHandler, the code below is all that is needed to retrieve the filters from the ResoureQuerySpecification class.

```c#
public class GetInvoiceResourceCollectionHandler : IRequestHandler<GetInvoiceResourceCollectionCommand, IEnumerable<Invoice>>
{
    private readonly ChinookDbContext _chinookDbContext;

    public GetInvoiceResourceCollectionHandler(ChinookDbContext chinookDbContext)
    {
        _chinookDbContext = chinookDbContext;
    }

    public async Task<IEnumerable<Invoice>> Handle(GetInvoiceResourceCollectionCommand request, CancellationToken cancellationToken)
    {
        Expression<Func<Invoice, bool>> invoiceFilter = request.querySpecification.GetFilter<Invoice>(nameof(Invoice));
        Expression<Func<Customer, bool>> customerFilter = request.querySpecification.GetFilter<Customer>(nameof(Customer));

        var invoiceQuery = _chinookDbContext.Invoices.Where(invoiceFilter);
        var customerQuery = _chinookDbContext.Customers.Where(customerFilter);

        var query =
            from fi in invoiceQuery
            join fc in customerQuery on fi.CustomerId equals fc.CustomerId
            select fi;

        var result = await query
            .TagWithSource()
            .ToListAsync();

        return result;
    }
}
```

Let's break down the code, the [GetInvoiceResourceCollectionCommand](https://github.com/circleupx/Chinook/blob/master/src/Chinook.Infrastructure/Commands/GetInvoiceResourceCollectionCommand.cs) exposes now the specification we need to retrieve the filter expression, here the [GetFilter](https://github.com/circleupx/Chinook/blob/master/src/Chinook.Core/ResoureQuerySpecification.cs#L14) is used to retrieve the generated filter expression, as mentioned before, if the filter doesn't exist we default to the expression created by the PredicateBuilder class. Once we have the expression, a query is created for each resource we want to support in this case there is query for the Invoice and Customer resource. The two queries are then combined in a joining query, the joining query is executed, here EF Core will take the expression and translate it to the proper SQL code.

### Results

Let's run a few examples. If I navigate to the invoices resource collection without any query parameters I get back 412 records in a single call, remember we haven't added pagination.

```shell
GET /invoices HTTP/1.1
```

The database was queried using the following SQL query.

```sql
SELECT "i"."InvoiceId", "i"."BillingAddress", "i"."BillingCity", "i"."BillingCountry", "i"."BillingPostalCode", "i"."BillingState", "i"."CustomerId", "i"."InvoiceDate", "i"."Total"
FROM "invoices" AS "i"
INNER JOIN "customers" AS "c" ON "i"."CustomerId" = "c"."CustomerId"
```

So far so good, let's run another test.

```shell
GET /invoices?filter[invoices]=billingCountry eq 'Brazil' HTTP/1.1
```

The API returns now only 35 records and not the usual 412, so some type of filtering appears to be happening. Let's confirm by looking at the SQL generated.


```SQL
SELECT "i"."InvoiceId", "i"."BillingAddress", "i"."BillingCity", "i"."BillingCountry", "i"."BillingPostalCode", "i"."BillingState", "i"."CustomerId", "i"."InvoiceDate", "i"."Total"
FROM "invoices" AS "i"
INNER JOIN "customers" AS "c" ON "i"."CustomerId" = "c"."CustomerId"
WHERE "i"."BillingCountry" = 'Brazil'
```

That looks right.

One more test, filtering a related resource which is what started this blog post.

```shell
GET /invoices?filter[invoices]=billingCountry eq 'Brazil'&filter[customers]=firstName eq 'Eduardo'
```

Returns only 7 records, so now we have further filtering, let's look at the generated SQL.

```SQL
SELECT "i"."InvoiceId", "i"."BillingAddress", "i"."BillingCity", "i"."BillingCountry", "i"."BillingPostalCode", "i"."BillingState", "i"."CustomerId", "i"."InvoiceDate", "i"."Total"
FROM "invoices" AS "i"
INNER JOIN (
    SELECT "c"."CustomerId", "c"."Address", "c"."City", "c"."Company", "c"."Country", "c"."Email", "c"."Fax", "c"."FirstName", "c"."LastName", "c"."Phone", "c"."PostalCode", "c"."State", "c"."SupportRepId"
    FROM "customers" AS "c"
    WHERE "c"."FirstName" = 'Eduardo'
) AS "t" ON "i"."CustomerId" = "t"."CustomerId"
WHERE "i"."BillingCountry" = 'Brazil'
```

Oh yeah, that looks right, this is perfect, filtering appears to be working as expected and now our clients can query not just top level resource but also related resource in a single HTTP request.

Goal achived.


### Resources

Here are a list of resource related to everything that I just talked about, these resource will come in handy if run into any issues.

1) [Entity Framework Core 5 – Pitfalls To Avoid and Ideas to Try](https://blog.jetbrains.com/dotnet/2021/02/24/entity-framework-core-5-pitfalls-to-avoid-and-ideas-to-try/)
2) [Dynamically Build LINQ Expressions](https://blog.jeremylikness.com/blog/dynamically-build-linq-expressions/)
3) [Expression Trees](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/expression-trees/)
4) [Giving Clarity to LINQ Queries by Extending Expressions](https://www.red-gate.com/simple-talk/development/dotnet-development/giving-clarity-to-linq-queries-by-extending-expressions/)
5) [How Do I Create an Expression<Func<>> with Type Parameters from a Type Variable](https://stackoverflow.com/questions/25793736/how-do-i-create-an-expressionfunc-with-type-parameters-from-a-type-variable)