---
title: Worker Services configure Serilog
layout: post
tags: [Worker Services, Serilog]
readtime: true
published: true
---

A [worker service](https://docs.microsoft.com/en-us/dotnet/core/extensions/workers) is a type of [Background Service](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.backgroundservice?view=dotnet-plat-ext-5.0) that are generally use for long-running task. They can be seen as the equivalent of Windows Services in the .NET Framework, though a worker service is not limited to just windows.

If you are building a worker service, then more than likely you will need to be able to write log data, be that general information of the worker services or perhaps just errors. If you plan to use Serilog, then this post will show you how to configure Serilog on a worker project.

To configure Serilog you will need the following NuGet packages.

```text
dotnet add package Serilog
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Extensions.Hosting
dotnet add package Serilog.Settings.Configuration
```

Once the packages are installed modify the Program.cs file to [bootstrap Serilog](https://nblumhardt.com/2020/10/bootstrap-logger/) and to [confiure Serilog](https://nblumhardt.com/2019/10/serilog-in-aspnetcore-3/). The following class illustrates how I usually configure Serilog.

```c#
public class Program
{
    public static void Main(string[] args)
    {
        Log.Logger = new LoggerConfiguration()
            .WriteTo.Console()
            .CreateBootstrapLogger();

        try
        {
            CreateHostBuilder(args)
                .Build()
                .Run();
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "Worker service failed initiation. See exception for more details");
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }

    public static IHostBuilder CreateHostBuilder(string[] args)
    {
        return Host.CreateDefaultBuilder(args).ConfigureServices((hostContext, services) => 
        {
            services.AddHostedService<Worker>();

        }).ConfigureLogging((hostContext, builder) =>
        {
            builder.ConfigureSerilog(hostContext.Configuration);
        }).UseSerilog();
    }
}
```

The most important piece of code in this class is the usage of the method UserSerilog. Withouth it you will have a configured Serilog but it won't be used by the worker, so don't forgot to use it. As for the method ConfigureSerilog, that is one of my extension methods. It reads Serilog configurations from appsettings.json. Here is the class definition.

```c#
public static class LoggingBuilderExtensions
{
    public static ILoggingBuilder ConfigureSerilog(this ILoggingBuilder loggingBuilder, IConfiguration configuration)
    {
        Log.Logger = new LoggerConfiguration()
            .ReadFrom.Configuration(configuration)
            .CreateLogger();

        return loggingBuilder;
    }
}
```

and my JSON configuration is as follows. 

```json
{
  "Serilog": {
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "theme": "Serilog.Sinks.SystemConsole.Themes.AnsiConsoleTheme::Code, Serilog.Sinks.Console",
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} <s:{SourceContext}>{NewLine}{Exception}"
        }
      }
    ]
  }
}
```

If you want to use a different Sink, [file](https://github.com/serilog/serilog-sinks-file), for example, then simply install the corresponding NuGet package and update the JSON configurations.

Now that everything has been configured, I can run the worker to see the prettified console logs.

![Console Logger](/assets/img/WorkerServices/ConsoleLog.PNG)

Ain't Serilog just awesome. 

Anyways, I hope this post helped you configured Serilog.