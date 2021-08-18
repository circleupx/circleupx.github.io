---
title: JSON&#58;API in .NET - Exception handling middleware
tags: [JSON&#58;API, REST]
readtime: true
---

On my second post on [JSON:API](https://jsonapi.org/) in .NET Core I wanted to create an exception handling [middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-3.1). This middleware would be responsible for catching all exceptions and for generating a JSON:API [Errors Documents](https://jsonapi.org/format/#document-top-level). 

I'll start by adding a middleware folder on the Chinook.Web project, for now it will only have the exception handling middleware, but, eventually it will have additional middleware.

![Middleware Folder](/assets/img/json-api/middleware-folder.PNG)

Folder has been added, now I will add the middleware class to the project in here.

```c#
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
         _logger = logger;
    }
    
    public Task Invoke(HttpContext httpContext)
    {
        return _next(httpContext);
    }
}
```

The code above is the default middleware class generate by Visual Studio, pretty simple, nothing complex, on another blog post I will come back to this middleware to add more complex error handling, for now it returns an HTTP 500 status code for all errors with a JSON:API Error document as the response. 

Our first modification to the code will be to wrap the code in the Invoke method around a try/catch. Then to create a HandleException method to put all logic that deals with error handling. I need to build the Errors Document and I also need to transform .NET Core Exception objects into a JSON:API Errors Object. Additionally, I want to include inner exceptions on the Errors document. Ben Brandt, has an [exception extension class](https://gist.github.com/benbrandt22/8676438) that I use a lot, though I slightly modified it for my use case, still it is useful because we can extract each child exception and log them while they are being added to the Errors document. 

![Exception Extension](/assets/img/json-api/exception-extension.PNG)

Back in the exception handling middleware class, I'll use the exception extension class to get all exceptions and to transformer them into a JSON:API Error Objects. The code ends up looking like this, 

```c#
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task Invoke(HttpContext httpContext)
    {
        try
        {
            await _next(httpContext);
        }
        catch (Exception exception)
        {
            await HandleException(httpContext, exception);
        }
    }

    private async Task<Task> HandleException(HttpContext httpContext, Exception exceptionContext)
    {
        // Change http status code to 500.
        httpContext.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
    
        var errorsDocument = new ErrorsDocument();
        var exceptionCollection = exceptionContext.GetAllExceptions(); // Get parent exception and all child exceptions
        foreach (var exception in exceptionCollection)
        {
            // For pointers, see https://tools.ietf.org/html/rfc6901
            var pointer = JsonConvert.SerializeObject(new { pointer = exception.Source });
            var exceptionSource = JObject.Parse(pointer);
            var eventTitle = exception.GetType().Name;
            var eventId = RandomNumberGenerator.GetInt32(10000); // For demo purposes only. Your event ids should be tied to specific errors.
            var @event = new EventId(eventId, eventTitle);
            var linkDictionary = new Dictionary<string, Link>
            {
                {
                    Keywords.About, new Link(exception.HelpLink) // Link to error documentation, this is a hypermedia driven api after all.
                }
            };
            var targetSite = exception.TargetSite.Name;
            var meta = new JObject { ["targetSite"] = targetSite };

            var errorException = new ErrorException(
                Error.CreateId(@event.Id.ToString()),
                HttpStatusCode.InternalServerError,
                Error.CreateNewId(),
                @event.Name,
                exception.Message,
                exceptionSource,
                new Links(linkDictionary),
                meta);

            errorsDocument.AddError(errorException);
        }

        var jsonResponse = await errorsDocument.ToJsonAsync();
        return httpContext.Response.WriteAsync(jsonResponse); // Remember to always write to the response body asynchronously.
    }
}
```
We can test it by generating an exception, I can do that by modifying the GetHomeDocument method in the HomeResource class to throw a not implemented exception.

```c#
public Task<Document> GetHomeDocument()
{
    throw new NotImplementedException();
}
```

All I need to do now is to register the exception handling middleware in the StartUp.cs class. The Configure method should now look like this.

```c#
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseHttpsRedirection();
    app.UseRouting();
    app.UseAuthorization();
    app.UseExceptionHandlingMiddleware(); // Register ExceptionHandling Middleware
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

If I run the Chinook.Web project, I get the following JSON:API Errors Document.

```json
{
  "errors": [
    {
      "id": "3646",
      "status": "InternalServerError",
      "code": "94ee7495-d507-4819-ad8c-4ebaab08c8e4",
      "title": "NotImplementedException",
      "detail": "The method or operation is not implemented.",
      "source": {
        "pointer": "Chinook.Web"
      },
      "links": {
        "about": {
          "href": null
        }
      },
      "meta": {
        "targetSite": "GetHomeDocument"
      }
    }
  ]
}
```

Now the API has a global exception handler that will transform all errors into JSON:API Errors document.

{: .notice--warning}
The code here is meant to illustrate how to generate JSON:API errors documents using [JsonApiFramework](https://github.com/scott-mcdonald/JsonApiFramework). I would not use this code in a production environment as it is still missing important implementation such as having a proper event id for each error rather than just generating random number. Additionally, as I mentioned before, this doesn't handle specific errors like validation, 404s, invalid query parameters etc. I will cover those later on this series.