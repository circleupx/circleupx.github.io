---
title: Authorization Code From Terminal
tags: [.NET, OAuth]
author: "Yunier"
date: "2022-06-05"
description: "How to extract authorization code from the browser onto a terminal."
---

I was recently presented with a unique challenge at work. I needed to create a script that clones repositories from Bitbucket. The problem is that as of June 2022, Bitbucket only supports managing repositories using OAuth via two grant types, the authorization code grant & the implicit grant. 

I won't get into the details here but the implicit grant is no longer recommended and is in fact discouraged from ever being used. Regardless of which flow I use I will end up facing the same problem, the browser. In both the implicit and authorization grant, user interaction (3-legged OAuth) is required, the end-user must provide their credentials in order to properly authenticate, in some cases this may even include multifactor authentication.

Normally this isn't much of a problem, except in the case where the end-user is running a script from a terminal like Powershell. When the script runs, a browser must be opened to some predefined URL that will prompt the user to authenticate, when this happens the identity provider, in my case Bitbucket, will redirect the user back to another predefined URL, in the redirect URL, the authorization code will be included. This authorization code must then be exchanged for an access token.

The challenged I faced was how to retrieve the authorization code from the redirect URL? The redirect happens on the web browser but the script is being run under a terminal. Luckily I have some experience working with OAuth and generally understood what needed to be done at a high level. When the redirect occurs, a listener must be ready to extract the authorization code from the redirect URL and inject it into the terminal executing the script and while I knew what needed to be done I myself have never done anything like this before, hence my statement of being presented with a unique challenge.

I want to take the time to document my solution to the problem above in today's post. Instead of Bitbucket, I'll use Google for my example. If you want to follow along, I suggest reading [Getting Google OAuth Access Token using Google APIs](https://medium.com/automationmaster/getting-google-oauth-access-token-using-google-apis-18b2ba11a11a) by [Osanda Deshan Nimalarathna](https://medium.com/@osandadeshan) to properly configure your OAuth client.

With a properly configured OAuth client, we can move on to the next step, the authorization code listener. The purpose of the listener is to grab the authorization token from the redirect URL. This is done by having a local web server that listens for the redirect URL, then when the user is redirected the listener outputs the authorization code to stdout. By writing the code to stdout, a terminal like Powershell or any Linux terminal can save the authorization code value to a variable that can later be referenced when the code is used to get an access token. 

A few implications to be aware of. First, the redirect URL must be localhost with a port of your choosing. Second, the client redirect URL must be the same localhost URL. You can choose to redirect to the root for example, with something like http:localhost:9000 or http:localhost:9000/auth/callback. The redirect URL can be anything you want, up to you, you just need to make sure the local server is running and listening to any request made to the redirect URL endpoint otherwise you will not be able to capture the authorization code.

I ultimately ended up building the listener with .NET but you can use anything you want. Some languages like python come with a built-in local server that can be executed using the following command.

```python
python3 -m http.server
```

Other options include using great packages like [http-server](https://www.npmjs.com/package/http-server) if you prefer to write the listener in Node.js. The server implementation can be done in any language that can write to stdout. As for the .NET implementation, I wasn't sure how to go about doing it, I decided to search and came across a blog post from [Dean Ward](https://twitter.com/deanward81/) titled [Authenticating to Google using PowerShell and OAuth](https://bakedbean.org.uk/posts/2020-01-powershell-oauth/) which pointed me in the right direction. Below is the code created by Dean.

```c#
class Program
{
  static readonly CancellationTokenSource _cts = new CancellationTokenSource();
  static Task Main(string[] args) =>
    WebHost.CreateDefaultBuilder()
      // prevent status messages being written to stdout
      .SuppressStatusMessages(true)
      // disable all logging to stdout
      .ConfigureLogging(config => config.ClearProviders())
      // listen on the port passed on the args
      .UseKestrel(options => options.ListenLocalhost(int.Parse(args[0])))
      .Configure(app => app.Run(async context =>
        {
          var message = "ERROR! Unable to retrieve authorization code.";
          if (context.Request.Query.TryGetValue("code", out var code))
          {
            // we received an authorization code, output it to stdout
            Console.WriteLine(code);
            message = "Done!";
          }
          await context.Response.WriteAsync(message + " Check Dev-Local-Setup to continue");
          // cancel the cancellation token so the server stops
          _cts.Cancel();
        })
      )
      .Build()
      // run asynchronously using the cancellation token
      // to signal when the process should end. This will be awaited
      // by the framework and the process will end when the cancellation
      // token is signalled.
      .RunAsync(_cts.Token);
  }
```

As you can see, the code creates a local web server that listens on localhost on a port that is passed as a parameter when the server is started. I tried to port the code to .NET 6 but it did not work. The code above was written on .NET Core 3. There is a substantial difference between .NET Core 3.1 and .NET in terms of how the server is initialized, therefore, I had to slightly rewrite Dean's code.

The code provided by Dean in .NET 6 ended up like this.

```C#
public class Program
{
    private static readonly CancellationTokenSource CancellationTokenSource = new CancellationTokenSourc();
    
    public static async Task Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);
        builder.WebHost.SuppressStatusMessages(true);
        builder.WebHost.ConfigureLogging(loggingBuilder => loggingBuilder.ClearProviders());
        builder.WebHost.UseKestrel(options =>
        {
            options.ListenLocalhost(int.Parse(args[0]));
        });

        var app = builder.Build();
        app.Map("/oauth/callback", HandleCallback);
        await app.RunAsync(CancellationTokenSource.Token);
    }

    private static void HandleCallback(IApplicationBuilder app)
    {
        app.Run(context =>
        {
            if (context.Request.Query.TryGetValue("code", out var code))
            {
                // we received an authorization code, output it to stdout
                Console.WriteLine(code);
                CancellationTokenSource.Cancel();
            }
            return Task.CompletedTask;
        });
    }
}
```

My code is very similar to Dean's code though I should point out a few things. Just like Dean's code, the setting to suppress status code messages is still set to true. This is essential because status messages are the messages you see when you run a .NET application and those messages are written to stdout. Without this setting, stdout would be polluted with data that we do not need, which means we would need additional work to properly parse the data to get the authorization code. Again, keep stdout clean of any unnecessary messages.

The next step is two clean the logging providers which leaves only the console as a provider. This is an optional step in a project this small. The next step is to configure kestrel, just like in Dean's code we take the port as an argument that was passed to the application when the command dotnet run was executed. Next, the web app is built and we map our redirect URL to a handler, the handler will attempt to get the "code" parameter from the redirect URL and write to stdout. This means the value will be available for any terminal to use, then using a cancellation source, a cancel signal is sent to shut down the listener. This step is essential, the cancellation signal tells .NET that the server can be shut done. You don't want to be in a state where the local server is running indefinitely.

One last tip, the most important, do not forget to await app.RunAsync. I was having a lot of trouble when I was testing, the local server was never available this is because I was not awaiting the task. So, don't forget to use the await keyword to await app.RunAsync, take a look at  [.NET Generic Host in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-6.0#set-up-a-host) for more details. 

Everything is now ready for us to capture the authorization code from a terminal. All that missing is the script that will execute all of this work. Dean's script works but I found out that it also required some small modifications. Below you will find my modified PowerShell script.


```Powershell
function Get-AccessToken {
  $accessToken = $null;
  $port = 8000
  $clientId = "<your client id>"
  $clientSecret = "<your client secret>"
  $redirectUrl = "http://localhost:$port/oauth/callback"
  $url = "https://accounts.google.com/o/oauth2/auth?scope=https://www.googleapis.com/auth/drive&response_type=code&access_type=offline&redirect_uri=$redirectUrl&client_id=$clientId"

  Write-Host "Launching web browser to authenticate..."
  Start $url

  # Build the .NET solution
  dotnet build -c Release 'path to project or solution'
  # Run the .NET solution
  $authorizationCode = & dotnet run -c Release --no-build -project 'path to project or solution' -- $port

  if ($authorizationCode) {
    $authorizationResponse = Invoke-RestMethod -Uri "https://www.googleapis.com/oauth2/v4/token?code=$authorizationCode&client_id=$clientId&client_secret=$clientSecret&redirect_uri=http://localhost:$port&grant_type=authorization_code" -Method Post
    $accessToken = $authorizationResponse.access_token
  }
  else {
      Write-Host "Unexpected error while retrieving access token" -ForegroundColor Red 
  }

  return $accessToken;
}
```

Something you may notice from the above Powershell is that the commands that build and run the local server are separate. This was done on purpose, you could combine both commands into a single command, dotnet run, because dotnet runs executes dotnet build by default. However, I found a flaw to this approach, when you run dotnet run, dotnet build will be executed and in that execution, logs will be written to stdout, which again is something we want to avoid, we want to keep stdout clean of any messages or logs. So my solution was to split the commands into a build and run and to use the --no--build flag on dotnet run.

Before I end this post, I do want to let you know that I did look at other options such as using a WebView window as described by Stephen Owen in [Using PowerShell and oAuth](https://www.foxdeploy.com/blog/using-powershell-and-oauth.html). I found WebView to be vastly inferior, it lacks the proper controls and tools that modern browsers providers, such as the convenience of a password manager that can auto-fill your information or the fact that you may already be authenticated and don't have to authenticate again. If you are interested in using WebView you can find a sample code over [in this repo](https://github.com/gscales/Powershell-Scripts/blob/master/Graph101/SPAuth.ps1). 