---
title: A practical Web API validation strategy
tags: [REST, FluentValidation, ProblemDetails]
author: "Yunier"
date: "2021-10-13"
description: "Guide on how to create practical API validation in .NET"
---

In my [last post](/post//2021/a-better-web-api-validation-strategy/index/) I wrote about how you can leverage [JSON Schema](https://json-schema.org/) to do Web API validation. The main benefit is that the API can expose the schema as an API resource, clients of the API can consume the schema and execute it on their end against any data. The benefit of doing API validation like this is that the client does not need to duplicate any validation logic, they only need to execute the schema. In this post, I would like to explore API validation in .NET, using the library [FluentValidation](https://fluentvalidation.net/) and exposing validation errors using [Problem Details](https://datatracker.ietf.org/doc/html/rfc7807).

I'll start by creating a new Web API project using the following dotnet command.

```console
dotnet new webapi
```

Next, I'll add the package FluentValidation as a dependency.

```console
dotnet add package FluentValidation.AspNetCore
```

The .NET webapi template, the one used when I executed dotnet new webapi, comes with a weathers controller that exposes the following model.

```c#
public class WeatherForecast
{
    public DateTime Date { get; set; }

    public int TemperatureC { get; set; }

    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);

    public string Summary { get; set; }
}
```

I'm going to enhance the Web API project I just created by introducing a new HTTP POST endpoint in the API. This new endpoint will allow a client app to create a new WeatherForecast resource. The WeatherForecast model will also be enhanced by having fluent validation enforce the following rules.

1. The field, Date, is required, must be UTC.
2. The field, Summary, is required and must not be empty.
3. The field, TemperatureC is required and must be a valid range

I'll need to create a class that implements [AbstractValidator](https://github.com/FluentValidation/FluentValidation/blob/main/src/FluentValidation/AbstractValidator.cs). I will name the class, WeatherForecastValidator, as seen below, it will be used to define all the validation rules that pertain to the WeatherForecast model.

```c#
public class WeatherForecastValidator : AbstractValidator<WeatherForecast>
{
    public WeatherForecastValidator()
    {
        RuleFor(x => x.Date)
            .NotNull()
            .WithErrorCode("missingDate")
            .WithMessage("Date is required field")
            .Must(x => x.Kind == DateTimeKind.Utc)
            .WithErrorCode("dateIsNotUtc")
            .WithMessage("Date is not in UTC format.");

        RuleFor(x => x.Summary)
            .NotNull()
            .WithErrorCode("missingSummary")
            .WithMessage("Summary is a required field")
            .NotEmpty()
            .WithErrorCode("invalidSummary")
            .WithMessage("Summary cannot be empty");

        RuleFor(x => x.TemperatureC)
            .InclusiveBetween(-18, 40)
            .WithErrorCode("invalidTemperatureValue")
            .WithMessage("TemperatureC must be between -18 and 40");
    }
}
```

Now that the validation rules are in place, I need to register the validator on the .NET pipelines so that any validation errors are properly propagated through the .NET application. To register a validator class, just use the AddFluentValidation extension method off of the AddControllers method in the Startup class. You can add them one by one or have FluentValidation scan a given assembly to automatically import validators.

```c#
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers()
            .AddFluentValidation(fv => fv.RegisterValidatorsFromAssemblyContaining<Program>());
    }
}
```

The last thing I now need to do is update the WeatherForecast controller with the new HTTP POST endpoint.

```c#
[ApiController]
[Route("[controller]")]
public class WeatherForecastController : ControllerBase
{
    private static readonly string[] Summaries = new[]
    {
        "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
    };

    private readonly ILogger<WeatherForecastController> _logger;

    public WeatherForecastController(ILogger<WeatherForecastController> logger)
    {
        _logger = logger;
    }

    [HttpGet]
    public IEnumerable<WeatherForecast> Get()
    {
        var rng = new Random();
        return Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Date = DateTime.Now.AddDays(index),
            TemperatureC = rng.Next(-20, 55),
            Summary = Summaries[rng.Next(Summaries.Length)]
        })
        .ToArray();
    }

    [HttpPost]
    public IActionResult Post([FromBody] WeatherForecast weather)
    {
        if(!ModelState.IsValid)
        {
            var validationProblemDetails = new ValidationProblemDetails(ModelState);
            return BadRequest(validationProblemDetails);
        }

        return Created(weather);
    }
}
```

Perfect, I can test the new endpoint by sending an HTTP POST to the WeatherForecast endpoint with the following JSON in the HTTP request body.

```JSON
{
  "date": "2021-11-07T01:29:13.695Z",
  "temperatureC": 0,
  "summary": ""
}
```

As you can tell from the payload, the field summary is empty, this should trigger a validation error that should end up in me getting a ProblemDetals response object. When I submit the HTTP request, I received the following response from the API.

```JSON
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "traceId": "00-6f0c73e8915f564488e1b4ab538c3ec0-ae270bf71b7ee44b-00",
  "errors": {
    "Summary": [
      "Summary cannot be empty"
    ]
  }
}
```

Nice, as you can see the error messages defined in the WeatherForecastValidator are included on the response object. With this approach, the API can send a friendly error message to the client whenever a validation error occurs. FluentValidation even helps you with [Localizations](https://docs.fluentvalidation.net/en/latest/localization.html).


Couple of things to note. The validations I've written here are arbitrary and outright stupid, for example, celsius can include decimal values like -17.8, which would be 0 Fahrenheit. Please ignore the validity of the validation rules I have written here, this post is meant to demonstrate how to provide practical API validation in .NET by using tools and patterns like FluentValidation and ProblemDetails.

If you are in .NET Core 3 or earlier, ModelState is not serialized using camel casing. You can see that in the example above, where the word "Summary", in the error array has an upper case s. This was fixed in [7439](https://github.com/dotnet/aspnetcore/issues/7439) for newer versions of .NET. In 3 or earlier you can fix that by using the following code.

```c#
services.AddControllers()
    .AddNewtonsoftJson(mvcNewtonsoftJsonOptions => mvcNewtonsoftJsonOptions.UseCamelCasing(processDictionaryKeys: true));
```

If you prefer to not have a check on the state of ModelState on every action that requires validation. You can configure a global response by configuring the [ApiBehaviorOptions](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.apibehavioroptions?view=aspnetcore-6.0) in .NET on the StartUp.cs class.

```c#
public void ConfigureServices(IServiceCollection services)
{

    services.AddControllers()
        .AddFluentValidation(fv => fv.RegisterValidatorsFromAssemblyContaining<Program>());
    
    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new OpenApiInfo { Title = "webapi", Version = "v1" });
    });

    services.Configure<ApiBehaviorOptions>(options =>
    {
        options.InvalidModelStateResponseFactory = context =>
        {
            var problemDetails = new ValidationProblemDetails(context.ModelState)
            {
                Instance = context.HttpContext.Request.Path,
                Status = 400,
                Type = $"https://httpstatuses.com/400",
                Detail = "Validation Error"
            };
            return new BadRequestObjectResult(problemDetails)
            {
                ContentTypes =
                {
                    "application/problem+json"
                }
            };
        };
    });
}
```