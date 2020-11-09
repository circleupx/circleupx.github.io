---
title: JSON&#58;API in .NET Core - The Home Resource. 
layout: post
categories: [.NET Core, JSON, JSON&#58;API, JsonApiFramework, APIs, REST]
image: /assets/img/json-api/json-api.PNG
description: "Part one of my blog series on building a .NET Core Web Api using JSON&#58;API"

---

Hello, this is going to be my first entry into a multi-part series of post showing how I build RESTful APIs using the [JSON:API](https://jsonapi.org/) specification.

In this post, I will start by creating a new .NET Core Web Api project, I am going to call this project **Chinook**, after the sqlite [database](https://www.sqlitetutorial.net/sqlite-sample-database/) that I will use for this project. Whenever I create a .NET Core project I like to follow the project structure outlined by Steve Smith in his [Clean Architecture](https://github.com/ardalis/CleanArchitecture) repository.

![Chinook Project Structure](/assets/img/json-api/chinook-project-structure.PNG)

Now that I have a project structure, I will a NuGet package that will help build the API. The official JSON:API website contains a [list](https://jsonapi.org/implementations/#server-libraries-net) of implementations that can be use on .NET Core. Out of that list, [JsonApiDotNetCore](https://github.com/json-api-dotnet/JsonApiDotNetCore) is by far the most popular and the one with the most community support, but for the Chinook project I will utilize [JsonApiFramework](https://github.com/scott-mcdonald/JsonApiFramework). The reason for that is that I do not like using decorators on my models or service models as JsonApiFramework calls them. 

The convention/configuration approach taken by JsonApiFramework means that your models remain clean and independent of any JSON:API implementation details, it also means that the models can be used in other higher level frameworks like EF Core with no interference. JsonApiDotNetCore does plan to [support](https://github.com/json-api-dotnet/JsonApiDotNetCore/issues/776) a similar fluent style API, but for now JsonApiFrameWork will do the job.

OK. Let's get started by installing JsonApiFramework using the following dotnet command.

```bash

dotnet add package JsonApiFramework.Server --version 2.4.0

```

Now that I have JsonApiFramework installed, I can add our first API resource, the Home resource. If you are not familiar with the concept of having a home resource, I suggest reading [API Discovery Using JSON Home](https://apievangelist.com/2017/08/03/api-discovery-using-json-home/) by API Evangelist. Before creating the Home resource, I will add a new folder, Resources, to the Chinook.Web project, in this folder I will add the HomeResource class.

![Home Resource Class](/assets/img/json-api/home-resource.PNG)

It is now time to configure our Home resource using JsonApiFramework. First, I added a new folder named ServiceModels under Chinook.Core. In this folder I added new class, HomeServiceModel. Additionally, under the ServiceModels I added a Configuration folder, and within this folder I added a HomeConfiguration class. Our Chinook.Core project so far looks like this.

![Home Service Model](/assets/img/json-api/home-service-model.PNG)

Our HomeServiceModel class is rather simple, for now it will have a message property, the property will be used to display a "Hello World" message on our home document.

```c#
namespace Chinook.Core.ServiceModels
{
    public class Home
    {
        public string Message { get; set; }
    }
}
```

As for our configuration class, I will set the API path to be an empty string as the home resource will not expose relationships to other resources. Then I set the [JSON:API type](https://jsonapi.org/format/#document-resource-object-identification) for this resource to be "home".

```c#
using JsonApiFramework.ServiceModel.Configuration;

namespace Chinook.Core.ServiceModels.Configurations
{
    class HomeConfiguration : ResourceTypeBuilder<Home>
    {
        private const string JsonApiType = "home";

        public HomeConfiguration()
        {
            Hypermedia()
                .SetApiCollectionPathSegment(string.Empty);

            ResourceIdentity()
                .SetApiType(JsonApiType);
        }
    }
}
```

Now that I have created the home configuration class I will need to configure JsonApiFramework to use it. I will start by adding a ConfigurationFactory class under the Chinook.Core project. In this class we can tell JsonApiFramework how I want our API resources to be configured. For example, I want to the JSON:API attributes to use Camel Casing, I also want JsonApiFramework to pluralize the API types, and I also want JsonApiFramework to auto discovery properties in my models and to map them, if my model exposes an Id property, then I want JsonApiFramework to automatically map that to the JSON:API resource's Id. These type of configurations can be achieved like this.

```c#
using JsonApiFramework.Conventions;
using JsonApiFramework.ServiceModel;
using JsonApiFramework.ServiceModel.Configuration;

namespace Chinook.Core.ServiceModels
{
    public static class ConfigurationFactory
    {
        public static IConventions CreateConventions()
        {
            var conventionsBuilder = new ConventionsBuilder();

            conventionsBuilder.ApiAttributeNamingConventions()
                              .AddStandardMemberNamingConvention();

            conventionsBuilder.ApiTypeNamingConventions()
                              .AddPluralNamingConvention()
                              .AddStandardMemberNamingConvention();

            conventionsBuilder.ResourceTypeConventions()
                              .AddPropertyDiscoveryConvention();

            var conventions = conventionsBuilder.Create();
            return conventions;
        }
    }
}
```

Something to note, the method AddStandardMemberNamingConvention() just refers to the fact that JSON:API recommends using [camel casing](https://jsonapi.org/recommendations/#naming). JsonApiFramework does support Pascal, Kebab or your own custom casing, though I have never explored that option.

It is now time to update our Home resource class.

```c#
namespace Chinook.Web.Resources
{
    public class HomeResource
    {
        private readonly IHttpContextAccessor httpContextAccessor;
        private readonly ILogger<HomeResource> logger;

        public HomeResource(IHttpContextAccessor httpContextAccessor, ILogger<HomeResource> logger)
        {
            this.httpContextAccessor = httpContextAccessor;
            this.logger = logger;
        }

        public Task<Document> GetHomeDocument()
        {
            var homeResource = new HomeServiceModel
            {
                Message = "Hello World"
            };

            var displayUri = httpContextAccessor.HttpContext.Request.GetDisplayUrl();
            var currentRequestUri = new Uri(displayUri);

            using var chinookDocumentContext = new ChinookDocumentContext(currentRequestUri);
            var document = chinookDocumentContext
                        .NewDocument(currentRequestUri)
                            .SetJsonApiVersion(JsonApiVersion.Version10)
                            .Links()
                                .AddSelfLink()
                            .LinksEnd()
                            .Resource(homeResource)
                            .ResourceEnd()
                        .WriteDocument();

            return Task.FromResult(document);
        }
    }
}
```

Now, the last item we need, the DocumentContext, this class is required by JsonApiFramework whenever we work with a JSON:API documents. Here is the completed class.

```c#
namespace Chinook.Core
{
    public class ChinookDocumentContext : DocumentContext
    {
        public ChinookDocumentContext(Uri currentRequestUri)
        {
            var urlBuilderConfiguration = CreateUrlBuilderConfiguration(currentRequestUri);
            UrlBuilderConfiguration = urlBuilderConfiguration;
        }

        public ChinookDocumentContext(Uri currentRequestUri, Document document) : base(document)
        {
            var urlBuilderConfiguration = CreateUrlBuilderConfiguration(currentRequestUri);
            UrlBuilderConfiguration = urlBuilderConfiguration;
        }

        protected override void OnConfiguring(IDocumentContextOptionsBuilder optionsBuilder)
        {

            var serviceModel = ConfigurationFactory.CreateServiceModel();
            optionsBuilder.UseServiceModel(serviceModel);
            optionsBuilder.UseUrlBuilderConfiguration(UrlBuilderConfiguration);
        }

        private IUrlBuilderConfiguration UrlBuilderConfiguration { get; set; }

        private static UrlBuilderConfiguration CreateUrlBuilderConfiguration(Uri currentRequestUri)
        {
            var scheme = currentRequestUri.Scheme;
            var host = currentRequestUri.Host;
            var port = currentRequestUri.Port;
            var urlBuilderConfiguration = new UrlBuilderConfiguration(scheme, host, port);
            return urlBuilderConfiguration;
        }
    }
}
```

I have all the required dependencies, I can run the web api project.

![Home Document Wrong Serializer](/assets/img/json-api/home-document-with-wrong-json-serializer.PNG)

Kinda hard to see, I am going to install JSON prettifier to my browser, if you are using a chrome based browser then I recommend installing [JSON VIEWER](https://chrome.google.com/webstore/detail/json-viewer/gbmdgpbipfallnflgajpaliibnhdgobh). Now If I reload the page.

![Using JSON Viewer](/assets/img/json-api/home-document-with-json-viewer.PNG)

Much better, JSON Viewer sure helps, but wait, something doesn't look right. The serialization does not look correct, luckily I know the problem and how it can be fixed. JsonApiFramework has a [hard decency](https://github.com/scott-mcdonald/JsonApiFramework/issues/67) on Newtonsoft.JSON and .NET 3.1 uses [System.Text.Json](https://devblogs.microsoft.com/dotnet/try-the-new-system-text-json-apis/). We can correct our problem by installing the following NuGet package.

```bash
dotnet add package Microsoft.AspNetCore.Mvc.NewtonsoftJson
```

I need to update our StartUp.cs class so that NewtonsoftJson is used to serialize/deserialize JSON.

```c#
namespace Chinook
{
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            //Omitted for brevity

            services.AddControllers()
                .AddNewtonsoftJson();
        }
    }
}
```

All set, if I run the project now I should get home document with the correct json serialization settings.
![Final Home Document](/assets/img/json-api/home-document-with-correct-json-serialization.PNG).

Perfect, our home resource is now working as expected.