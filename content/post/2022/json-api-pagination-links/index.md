---
title: JSON:API - Pagination Links
tags: [JSON:API, REST]
author: "Yunier"
date: "2022-01-25"
description: "Exposing pagination links on JSON:API documents"
series: ['JSON:API in .NET']
---

It has been a while since I blogged about [JSON:API](https://jsonapi.org/). In my last post on JSON:API I covered how to create [new resources](/post/2021/json-api-creating-new-resources). In today's post, I want to go over how I expose pagination links. [Pagination links](https://jsonapi.org/examples/#pagination) allow a client to page through a collection of resources. A shift of control from the client back to the server.

Here is an example of a possible JSON:API response that includes pagination links.

```json
{
  "data": [
    {
      // omitted for brevity
    }
  ],
  "links": {
    "up": "http://example.com/",
    "self": "http://example.com/articles?page[number]=3&page[size]=1",
    "first": "http://example.com/articles?page[number]=1&page[size]=1",
    "prev": "http://example.com/articles?page[number]=2&page[size]=1",
    "next": "http://example.com/articles?page[number]=4&page[size]=1",
    "last": "http://example.com/articles?page[number]=13&page[size]=1"
  }
}
```

As you can see, along with the [data](https://jsonapi.org/format/#document-top-level) object, the API responses included a [Links](https://jsonapi.org/format/#document-links) object, within the Links object, you can find links for up, self, first, prev, next, and last. These are all relationship name as defined in [Link Relations](https://www.iana.org/assignments/link-relations/link-relations.xhtml) by the Internet Assigned Numbers Authority (IANA).

The "up" link refers to a parent document in a hierarchy of documents. The "self" link is an identifier for the current document. The "first" link refers to the furthest preceding resource in a series of resources. The "prev" link indicates that the link's context is a part of a series and that the previous document in the series is the link target. The "next" link indicates that the link's context is part of a series and the next document in the series is the link's target. The "last" link refers to the furthest following resource in a series of resources.

The absence or presence of the pagination link is significant, if the "next" link exists, then there are more pages for the client to paginate through. If the "next" link does not exist, then the client has reached the last page. If the "prev" link exists, then the client is not on the first page. If neither a "next" or "prev" link exists, there is only one page.

I want to update the [Chinook](https://chinook-jsonapi.herokuapp.com/) project by exposing pagination links on the [customers](https://chinook-jsonapi.herokuapp.com/customers) resource. For that I will need to add a code to support reading and writing Links, calculating the total number of pages to determine if there is more than one page.

I'll start by adding the following PagedList class, this class will help me determine how many pages are available and if a previous and next page exists.

```C#
public class PagedList
{
    public PagedList(int recordCount, int pageNumber, int pageSize)
    {
        PageSize = pageSize;
        PageNumber = pageNumber;
        RecordCount = recordCount;
    }

    private int PageSize { get; }
    private int PageNumber { get; }
    private int RecordCount { get; }
    private int NumberOfPages => (int)Math.Ceiling(RecordCount / (double)PageSize);
    private bool HasAPreviousPage => PageNumber > 1;
    private bool HasANextPage => PageNumber < NumberOfPages;

    public bool HasNextPage()
    {
        return HasANextPage;
    }

    public bool HasPreviousPage()
    {
        return HasAPreviousPage;
    }

    public int GetPageSize()
    {
        return PageSize;
    }

    public int GetNextPageNumber()
    {
        return PageNumber + 1;
    }

    public int GetPreviousPageNumber()
    {
        return PageNumber - 1;
    }

    public int GetFirstPageNumber()
    {
        return 1;
    }

    public int GetLastPageNumber()
    {
        return NumberOfPages;
    }
}
```

Next, I will add code to handle creating the pagination links, it will also need to enforce pagination rules. For example, I prefer setting a limit on the number of records that can be retrieved per page. This is to prevent a single client from crashing the entire API. In my experience, I have found 100 to be the sweet spot.

We can drive the rules through configurations defined on our AppSettings.json file.

```json
{
  "PageConfiguration": {
    "DefaultNumber" : 1,
    "DefaultSize" : 10,
    "DefaultMax" : 100
  }
}
```

To load these settings, I will create a PageConfigurationSettings class, see below, this class will then be registered using the [default DI](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-6.0) framework from .NET so that these settings can be consumed by any class in the API.

```C#
public class PageConfigurationSettings
{
  public PageConfiguration PageConfiguration { get; set; }
}

public class PageConfiguration
{
  public int DefaultNumber { get; set; }
  public int DefaultSize { get; set; }
  public int DefaultSize { get; set; }
}
```

To properly manage the creation of pagination links I created three classes, one to read the query parameters from the incoming request and one to update those parameters and the last class will handle creating the Links by relying on the classes I just mentioned.

```C#
public class PaginationLinkWriter
{
    private readonly PagedList _pagedList;
    private readonly UriQueryParametersWriter _uriQueryParametersWriter;

    public PaginationLinkWriter(UriQueryParametersReader uriQueryParametersReader, UriQueryParametersWriter uriQueryParametersWriter, int totalCount)
    {
        var pageNumber = uriQueryParametersReader.GetPageNumberFromRequestUri();
        var pageSize = uriQueryParametersReader.GetSizeNumberFromRequestUri();
        
        _pagedList = new PagedList(totalCount, pageNumber, pageSize);
        _uriQueryParametersWriter = uriQueryParametersWriter;
    }

    public Link GetNextPageLink()
    {
        if (_pagedList.HasNextPage())
        {
            var nextPageNumber = _pagedList.GetNextPageNumber();
            var nextPageSize = _pagedList.GetPageSize();
            var updatedParameters = GetPageQueryParameters(nextPageNumber, nextPageSize);
            var nextPageUri = _uriQueryParametersWriter.ReplaceQueryParameters(updatedParameters);
            var nextPageLink = LinkBuilder.CreateResourceLink(nextPageUri);
            return nextPageLink;
        }
        else
        {
            return Link.Empty;
        }
    }

    public Link GetPreviousPageLink()
    {
        if (_pagedList.HasPreviousPage())
        {
            var nextPageNumber = _pagedList.GetPreviousPageNumber();
            var nextPageSize = _pagedList.GetPageSize();
            var updatedParameters = GetPageQueryParameters(nextPageNumber, nextPageSize);
            var nextPageUri = _uriQueryParametersWriter.ReplaceQueryParameters(updatedParameters);
            var nextPageLink = LinkBuilder.CreateResourceLink(nextPageUri);
            return nextPageLink;
        }
        else
        {
            return Link.Empty;
        }
    }

    public Link GetLastPageLink()
    {
        var nextPageNumber = _pagedList.GetLastPageNumber();
        var nextPageSize = _pagedList.GetPageSize();
        var updatedParameters = GetPageQueryParameters(nextPageNumber, nextPageSize);
        var nextPageUri = _uriQueryParametersWriter.ReplaceQueryParameters(updatedParameters);
        var nextPageLink = LinkBuilder.CreateResourceLink(nextPageUri);
        return nextPageLink;
    }

    public Link GetFirstPageLink()
    {
        var nextPageNumber = _pagedList.GetFirstPageNumber();
        var nextPageSize = _pagedList.GetPageSize();
        var updatedParameters = GetPageQueryParameters(nextPageNumber, nextPageSize);
        var nextPageUri = _uriQueryParametersWriter.ReplaceQueryParameters(updatedParameters);
        var nextPageLink = LinkBuilder.CreateResourceLink(nextPageUri);
        return nextPageLink;
    }

    private IDictionary<string, string> GetPageQueryParameters(int pageNumber, int pageSize)
    {
        var updatedParameters = new Dictionary<string, string>
        {
            {UriKeyWords.PageNumber, pageNumber.ToString()},
            {UriKeyWords.PageSize, pageSize.ToString()}
        };

        return updatedParameters;
    }
}

public class UriQueryParametersReader
{
    private Uri CurrentRequestUri;
    private QueryParameters CurrentRequestUriQueryParameters;
    private PageConfigurationSettings PageConfigurationSettings;

    public UriQueryParametersReader(Uri requestUri, PageConfigurationSettings pageConfigurationSettings)
    {
        CurrentRequestUri = requestUri;
        CurrentRequestUriQueryParameters = QueryParameters.Create(requestUri);
        PageConfigurationSettings = pageConfigurationSettings;
    }

    public int GetPageNumberFromRequestUri()
    {
        var pageParameters = CurrentRequestUriQueryParameters.Page;
        var hasPageParameterInRequestUri = pageParameters.TryGetValue(UriKeyWords.number, out var pageParamertersInUri);
        
        if(hasPageParameterInRequestUri)
        {
            var pageNumber = pageParamertersInUri.First();
            return int.Parse(pageNumber);
        }
        else
        {
            return PageConfigurationSettings.PageConfiguration.DefaultNumber;
        }
    }

    public int GetSizeNumberFromRequestUri()
    {
        var pageParameters = CurrentRequestUriQueryParameters.Page;
        var hasPageParameterInRequestUri = pageParameters.TryGetValue(UriKeyWords.size, out var pageParamertersInUri);
        
        if(hasPageParameterInRequestUri)
        {
            var pageSize = pageParamertersInUri.First();
            return int.Parse(pageSize);
        }
        else
        {
            return PageConfigurationSettings.PageConfiguration.DefaultSize;
        }
    }
}

public class UriQueryParametersWriter
{
    private Uri RequestUri;

    public UriQueryParametersWriter(Uri requestUri)
    {
        RequestUri = requestUri;
    }

    public Uri ReplaceQueryParameters(IDictionary<string, string> parameters)
    {
        var parsedQueryParameters = QueryHelpers.ParseQuery(RequestUri.Query);
        foreach (var parameterToReplace in parameters)
        {
            parsedQueryParameters.Remove(parameterToReplace.Key);
            parsedQueryParameters.Add(parameterToReplace.Key, parameterToReplace.Value);
        }

        return RequestUri.AddQueryStringsToUri(parsedQueryParameters);
    }

    public Uri StripParametersFromUri(IEnumerable<string> parameters)
    {
        var parsedQueryParameters = QueryHelpers.ParseQuery(RequestUri.Query);
        foreach (var parameterToRemove in parameters)
        {
            parsedQueryParameters.Remove(parameterToRemove);
        }
        return RequestUri.AddQueryStringsToUri(parsedQueryParameters);
    }

    public Uri StripParametersFromUri(string parameter)
    {
        var parsedQueryParameters = QueryHelpers.ParseQuery(RequestUri.Query);
        parsedQueryParameters.Remove(parameter);
        return RequestUri.AddQueryStringsToUri(parsedQueryParameters);
    }
}
```

I also created the following helper classes to make everything easier.

```C#
public static class UriKeyWords
{
    public static string PageNumber = $"page[{number}]";
    public static string PageSize = $"page[{size}]";
    public const string size = nameof(size);
    public const string number = nameof(number);
}

public static class LinksKeyWords
{
    public const string next = nameof(next);
    public const string prev = nameof(prev);
    public const string last = nameof(last);
    public const string first = nameof(first);
    public const string describedBy = nameof(describedBy);
}

public class CustomerQuerySpecification : IEntityQuerySpecification<Customer>
{
    public Expression<Func<Customer, bool>> FilterExpression { get; } = entity => true; // result in no SQL generated

    public int Take { get; }

    public int Skip { get; }

    public CustomerQuerySpecification(UriQueryParametersReader uriQueryParametersReader)
    {
        var pageNumber = uriQueryParametersReader.GetPageNumberFromRequestUri();
        var pageSize = uriQueryParametersReader.GetSizeNumberFromRequestUri();

        Take = pageSize;
        Skip = (pageNumber - 1) * pageSize;
    }
}
```

I have everything I need to expose pagination links on the customer resource. Time to update the customer resource class by updating the GetCustomerResourceCollection method in the CustomerResource class.

```C#
public class CustomerResource : ICustomerResource
{
  private readonly ILogger<CustomerResource> _logger;
  private readonly IMediator _mediator;
  private readonly UriQueryParametersReader _uriQueryParametersReader;
  private readonly UriQueryParametersWriter _uriQueryParametersWriter;
  private readonly Uri _currentRequestUri
  public CustomerResource(
      ILogger<CustomerResource> logger,
      IMediator mediator,
      IHttpContextAccessor httpContextAccessor,
      UriQueryParametersReader uriQueryParametersReader,
      UriQueryParametersWriter uriQueryParametersWritery)
  {
      _logger = logger;
      _mediator = mediator;
      _uriQueryParametersReader = uriQueryParametersReader;
      _uriQueryParametersWriter = uriQueryParametersWritery;
      _currentRequestUri = httpContextAccessor.HttpContext.GetCurrentRequestUri(); ;
  
  public async Task<Document> GetCustomerResourceCollection()
  {
      var customerResourceResult = await _mediator.Send(new GetCustomerResourceCollectionCommand())
      
      // Build hypermedia links
      var linktToCustomerJsonSchema = SchemaLinksBuilder.BuildLinkToCustomerSchema(_currentRequestUri);
      var linkBuilder = new PaginationLinkWriter(_uriQueryParametersReader, _uriQueryParametersWriter, customerResourceResult.Count)
      using var chinookDocumentContext = new ChinookJsonApiDocumentContext(_currentRequestUri);
      var document = chinookDocumentContext
          .NewDocument(_currentRequestUri)
          .SetJsonApiVersion(JsonApiVersion.Version10)
              .Links()
                  .AddSelfLink()
                  .AddUpLink()
                  .AddLink(LinksKeyWords.next, linkBuilder.GetNextPageLink())
                  .AddLink(LinksKeyWords.last, linkBuilder.GetLastPageLink())
                  .AddLink(LinksKeyWords.first, linkBuilder.GetFirstPageLink())
                  .AddLink(LinksKeyWords.prev, linkBuilder.GetPreviousPageLink())
                  .AddLink(LinksKeyWords.describedBy, linktToCustomerJsonSchema)
              .LinksEnd()
              .ResourceCollection(customerResourceResult.Value)
                  .Relationships()
                      .AddRelationship(InvoiceResourceKeyWords.ToManyRelationShipKey, new[] { Keywords.Related })
                  .RelationshipsEnd()
                  .Links()
                      .AddSelfLink()
                  .LinksEnd()
              .ResourceCollectionEnd()
          .WriteDocument()
      _logger.LogInformation("Request for {URL} generated JSON:API document {doc}", _currentRequestUri, document);
      return document;
  }
}
```

Using the fluent style API exposed by JsonAPIFramework, I can define additional links using the AddLink() method. Now, updating the CustomerResource class is not enough. I need to make use of the pagination parameters. Mainly passing the parameters down all the way down to the data layer so that Entity Framework can build the correct SQL query. I update the class GetCustomerResourceCollectionHandler to accept the CustomerQuerySpecification class. I also used CountAsync to get the total number of records based on the current request. This is a really important step, if the count is not correct, the pagination links will not be built correctly. The count also needs to take into account possible filtering. Since our project does not support filtering at the moment, the filter property in CustomerQuerySpecification is simply set to true. Here is another nifty thing to note, EF will accept the following code and not generate any additional SQL statements.  

```C#
public Expression<Func<Customer, bool>> FilterExpression { get; } = entity => true;
```

The fact that EF doesn't generate any additional SQL statements allows is great. I often used this to my advantage whenever I create custom filter expressions using extension methods.

Anyways, back to the code, here is the updated GetCustomerResourceCollectionHandler.

```C#
public class GetCustomerResourceCollectionHandler : IRequestHandler<GetCustomerResourceCollectionCommand, EntityCollectionResult<Customer>>
{
  private readonly ChinookDbContext _chinookDbContext;
  private readonly CustomerQuerySpecification _customerQuerySpecification
  public GetCustomerResourceCollectionHandler(ChinookDbContext chinookDbContext, CustomerQuerySpecification customerQuerySpecification)
  {
      _chinookDbContext = chinookDbContext;
      _customerQuerySpecification = customerQuerySpecification;
  
  public async Task<EntityCollectionResult<Customer>> Handle(GetCustomerResourceCollectionCommand request, CancellationToken cancellationToken)
  {
      var count = await _chinookDbContext.Customers.CountAsync(_customerQuerySpecification.FilterExpression);
      var value = await _chinookDbContext.Customers
          .TagWithSource()
          .Skip(_customerQuerySpecification.Skip)
          .Take(_customerQuerySpecification.Take)
          .ToListAsync(cancellationToken)
      return new EntityCollectionResult<Customer>(count, value);
  }
}
```

Let's test our change. As of today, February 13, 2022, there are only 59 customers on the Chinook SQL lite database. Therefore, when a client navigates to the customers resource without specifying any paging parameters, the total number of pages should be 6, since we have 59 customers, the client will be on the first page and the default page size is 10. I'm going to act as the client and navigate to the customer resource.

I'll send the following HTTP request.

```text
GET /customers HTTP/1.1
```

The request yields the following response.

```json
{
  "jsonapi": {
    "version": "1.0"
  },
  "links": {
    "self": "https://localhost:5001/customers",
    "up": "https://localhost:5001",
    "next": "https://localhost:5001/customers?page%5Bnumber%5D=2&page%5Bsize%5D=10",
    "last": "https://localhost:5001/customers?page%5Bnumber%5D=6&page%5Bsize%5D=10",
    "first": "https://localhost:5001/customers?page%5Bnumber%5D=1&page%5Bsize%5D=10",
    "prev": {
      "href": null
    },
    "describedBy": "https://localhost:5001/customers/schemas"
  },
  "data": [
    // omitted for brevity
  ]
}
```

Note that the links are URL encoded as they should be, but we can see from the response that the prev link is null, which indicates we are on the first page. We see that the last link decoded is customers?page[number]=6&page[size]=10. This is correct, the last page is 6 because there are only six pages. As I client, I now want to navigate to the last page.

I'll send the following HTTP request.

```text
GET /customers?page[number]=6&page[size]=10 HTTP/1.1
```

The request yields the following API response.

```json
{
  "jsonapi": {
    "version": "1.0"
  },
  "links": {
    "self": "https://localhost:5001/customers?page%5Bnumber%5D=6&page%5Bsize%5D=10",
    "up": "https://localhost:5001",
    "next": {
      "href": null
    },
    "last": "https://localhost:5001/customers?page%5Bnumber%5D=6&page%5Bsize%5D=10",
    "first": "https://localhost:5001/customers?page%5Bnumber%5D=1&page%5Bsize%5D=10",
    "prev": "https://localhost:5001/customers?page%5Bnumber%5D=5&page%5Bsize%5D=10",
    "describedBy": "https://localhost:5001/customers/schemas"
  },
  "data": [
    // omitted for brevity
  ]
}
```

We can see from the response that the next link is now null, which is correct, indicating that we are on the last page. Good so far, now let's change the request. As a client I want to get 100 records, not 10 upon my first HTTP request, meaning page one. Such a request should result in a response with a prev and next link as null.

Sending the following HTTP request.

```text
GET /customers?page[number]=6&page[size]=100 HTTP/1.1
```

The request yields the following API response.

```json
{
  "jsonapi": {
    "version": "1.0"
  },
  "links": {
    "self": "https://localhost:5001/customers?page%5Bnumber%5D=1&page%5Bsize%5D=100",
    "up": "https://localhost:5001",
    "next": {
      "href": null
    },
    "last": "https://localhost:5001/customers?page%5Bnumber%5D=1&page%5Bsize%5D=100",
    "first": "https://localhost:5001/customers?page%5Bnumber%5D=1&page%5Bsize%5D=100",
    "prev": {
      "href": null
    },
    "describedBy": "https://localhost:5001/customers/schemas"
  },
  "data": [
    // omitted for brevity
  ]
}
```

Our API response above looks correct. The customer resource now supports pagination links. All that is left to do is apply the same code change to all the resource collections. I will do that offline and update the API at a later time. As always you can verify this API changes yourself by visiting the [Chinook API](https://chinook-jsonapi.herokuapp.com/) directly, I recommend having some type of JSON viewer enable like [JSON Viewer](https://chrome.google.com/webstore/detail/json-viewer/gbmdgpbipfallnflgajpaliibnhdgobh).

Till next time. Cheerio.