---
title: Fun With Flurl
tags: [Flurl]
author: "Yunier"
date: "2022-11-01"
description: "Using Flurl as an HTTP Client"
---

A few months ago I was looking for a new HTTP client to use within my applications. I first checked on [awesome dotnet](https://github.com/quozd/awesome-dotnet) under the [HTTP section](https://github.com/quozd/awesome-dotnet#http) to see what projects the .NET community is using instead of the default HTTP client. One that immediately stands out is [RestSharp](https://github.com/restsharp/RestSharp), this project has been around for a while and is overall a good choice, but I was looking for something new and fresh, that is when I came across [Flurl](https://flurl.dev/).

Flurl, according to its own website, is a modern, fluent, asynchronous, testable, portable, buzzword-laden URL builder and HTTP client library for .NET. That is quite a statement to make, I wanted to see if Flurl held up to that statement by creating a few use cases to see how Flurl works and how it could be used.

To demonstrate, I will create a new console application on .NET 6, the app will use Flurl to make an API request, get the response, and serialize it to a JSON object. I also want to demonstrate how Flurl makes testing super easy using its fake HTTP mode.

First, creating the .NET 6 console app involves executing the following command in a terminal.

```shell
dotnet new console
```

Now that I have my project, I need to add Flurl using the following commands.

```shell
dotnet add package Flurl.Http --version 3.2.4
```

Now I need an API that I can invoke with Flurl. This is when I like to use [HTTPBin](https://httpbin.org/). HTTPBin is a simple request/response service, it allows you to test different aspects of the HTTP spec like headers, codes, statuses, and so on.

For my first HTTP request, I will use the Headers API, this API takes the headers from your request and returns them as the payload on the response as seen below.

```json
{
  "headers": {
    "Accept": "application/json",
    "Accept-Encoding": "gzip, deflate, br",
    "Accept-Language": "en-US,en",
    "Host": "httpbin.org",
    "Referer": "https://httpbin.org/",
    "Sec-Fetch-Dest": "empty",
    "Sec-Fetch-Mode": "cors",
    "Sec-Fetch-Site": "same-origin",
    "Sec-Gpc": "1",
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.0.0 Safari/537.36",
    "X-Amzn-Trace-Id": "Root=1-636b1492-6c9692c6282b170967942576"
  }
}
```

Now that I know what the JSON payload will be I can use a tool like [json2csharp](https://json2csharp.com/) to create a model that will bind to the API response. Quick note, if using Visual Studio, for a while now there has been an option under paste that creates a model from a JSON or XML payload. It is under Edit > Special Paste > JSON for JSON or Edit > Special Paste > XML for XML.

Using json2charp, the JSON payload above is converted into the following model.

```c#
public class Headers
{
    public string Accept { get; set; }

    [JsonProperty("Accept-Encoding")]
    public string AcceptEncoding { get; set; }

    [JsonProperty("Accept-Language")]
    public string AcceptLanguage { get; set; }
    public string Host { get; set; }

    [JsonProperty("Sec-Fetch-Dest")]
    public string SecFetchDest { get; set; }

    [JsonProperty("Sec-Fetch-Mode")]
    public string SecFetchMode { get; set; }

    [JsonProperty("Sec-Fetch-Site")]
    public string SecFetchSite { get; set; }

    [JsonProperty("Sec-Fetch-User")]
    public string SecFetchUser { get; set; }

    [JsonProperty("Sec-Gpc")]
    public string SecGpc { get; set; }

    [JsonProperty("Upgrade-Insecure-Requests")]
    public string UpgradeInsecureRequests { get; set; }

    [JsonProperty("User-Agent")]
    public string UserAgent { get; set; }

    [JsonProperty("X-Amzn-Trace-Id")]
    public string XAmznTraceId { get; set; }
}

public class Root
{
    public Headers headers { get; set; }
}
```

Where Root is the top-level representation of the JSON document that is returned by HTTPBin.

Now that I have my response model, I can make the API request using the following code. Three lines of code are all that are needed to make an HTTP request to the endpoint https://httpbin.org/headers using Flurl.

## Example

```csharp
var result = await "https://httpbin.org"
    .AppendPathSegment("headers")
    .GetJsonAsync<Root>();
```

Simple and super easy. The fluent style interface exposed by Flurl also provides methods to work with POST, PUT, DELETE, adding headers, and using authentication.

## Testing

As for testing, it involves putting Flurl into test mode, this can be done using [HttpTest](https://github.com/tmenier/Flurl/blob/dev/src/Flurl.Http/Testing/HttpTest.cs) class.

```c#
public class FlurlTests
{
    [Fact]
    public async Task AssertHTTPGetCallsHttpBin()
    {
        // Act
        var httpTest = new HttpTest()
        
        // Arrange
        var sut = await "https://httpbin.org"
            .AppendPathSegment("headers")
            .GetJsonAsync<Root>()
        
        // Assert
        httpTest.ShouldHaveCalled("https://httpbin.org/headers")
    }
}
```

With Flurl in test mode, all requests and configurations can be faked and controlled giving you the ability to test complex scenarios. If you are using dependency injection and Flurl's [IFlurlClientFactory](https://github.com/tmenier/Flurl/blob/dev/src/Flurl.Http/Configuration/IFlurlClientFactory.cs) you are going to need to inject [PerBaseUrlFlurlClientFactory](https://github.com/tmenier/Flurl/blob/dev/src/Flurl.Http/Configuration/PerBaseUrlFlurlClientFactory.cs) or provide your own implementation of IFlurlClientFactory.

## Error Handling

Error handling in Flurl is a bit different, the default behavior is to throw an exception on any calls that do not result in a status code that is in the 200 range. I like this behavior because it works great if you use an [exception handling middleware](/post/2020/json-api-exception-handling-middleware/) that can take an exception thrown by Flurl and convert the exception into a [problem details](/post/2021/problem-details-for-http-apis/) response.
