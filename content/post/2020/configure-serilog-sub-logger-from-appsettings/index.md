---
title: Configure Serilog sub-loggers using XML app settings
tags: [Serilog, .NET]
author: "Yunier"
date: "2020-08-31"
description: "Guide on how to configure Serilog through XML settings"
---

Serilog has a neat feature that allows you to configure sub-loggers. With this feature you can essentially have log specific instances running on your application. 

I recently had to configure a .NET Framework application to use two different sub-loggers and while I was able find many examples online on how to configure sub-loggers through AppSettings.json, I did not find any examples on how to configure them through AppSettings.config/App.config so I wanted to document that process on this post.

First, we need to grab a few NuGet packages.

We need Serilog.

```bash
dotnet add package Serilog
```

We have to read application settings from AppSettings.config/App.config, for that, we use the following Serilog package.

```bash
dotnet add package Serilog.Settings.AppSettings
```

Our sample application will be configured to log to files, so we need to grab file sink package.

```bash
dotnet add package Serilog.Sinks.File
```

and the last NuGet package.

```bash
dotnet add package Serilog.Filters.Expressions
```

This NuGet package will allow us to write logs to different sub-loggers based on an expression defined on our configuration. Our log will also be enriched logs with these expressions. 

Now that we have all the required NuGet package we can move on to adding our log configuration. For our sample application, we are going to configure two sub-loggers, sub-logger red and sub-logger blue.

```xml
<appSettings>
    <!-- Configure the red logger -->
    <add key="red:serilog:minimum-level" value="Verbose" />
    <add key="red:serilog:using:File" value="Serilog.Sinks.File" />
    <add key="red:serilog:using:FilterExpressions" value="Serilog.Filters.Expressions" />
    <add key="red:serilog:write-to:File.path" value="C:\Log\red-.log" />
    <add key="red:serilog:write-to:File.rollingInterval" value="Day" />
    <add key="red:serilog:write-to:File.formatter" value="Serilog.Formatting.Compact CompactJsonFormatter, Serilog.Formatting.Compact" />
    <add key="red:serilog:write-to:File.fileSizeLimitBytes" value="100" />
    <add key="red:serilog:write-to:File.retainedFileCountLimit" value="10" />
    <add key="red:serilog:write-to:File.restrictToMinimumLevel" value="Verbose" />
    <add key="red:serilog:filter:ByIncludingOnly.expression" value="isRed = true" />
    
    <!-- Configure the blue logger -->
    <add key="blue:serilog:minimum-level" value="Verbose" />
    <add key="blue:serilog:using:File" value="Serilog.Sinks.File" />
    <add key="blue:serilog:using:FilterExpressions" value="Serilog.Filters.Expressions" />
    <add key="blue:serilog:write-to:File.path" value="C:\Log\blue-.log" />
    <add key="blue:serilog:write-to:File.formatter" value="Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact" />
    <add key="blue:serilog:write-to:File.rollingInterval" value="Day" />
    <add key="blue:serilog:write-to:File.fileSizeLimitBytes" value="100" />
    <add key="blue:serilog:write-to:File.retainedFileCountLimit" value="10" />
    <add key="blue:serilog:write-to:File.restrictToMinimumLevel" value="Verbose" />
    <add key="blue:serilog:filter:ByIcludingOnly.expression" value="isBlue = true" />
</appSettings>
```
As you can see, the trick here is to put the name of the sub-logger in front of :serilog. Next, we need to have Serilog read the configuration we defined above. 

```c#
Log.Logger = new LoggerConfiguration()
                .WriteTo.Logger(ReadRedLoggerConfigurations)
                .WriteTo.Logger(ReadBlueLoggerConfigurations)
                .CreateLogger();
```
ReadRedLoggerConfigurations and ReadBlueLoggerConfigurations are two static methods used to configure each sub-logger. 

```c#
private static void ReadRedLoggerConfigurations(LoggerConfiguration loggerConfiguration)
{
    const string redLogger = "red";
    loggerConfiguration.ReadFrom.AppSettings(redLogger);
}

private static void ReadBlueLoggerConfigurations(LoggerConfiguration loggerConfiguration)
{
    const string blueLogger = "blue";
    loggerConfiguration.ReadFrom.AppSettings(blueLogger);
}
```
Here is the completed console application.

```c#
namespace SubLogger
{
    class Program
    {
        static void Main(string[] args)
        {
            try
            {
                Log.Logger = new LoggerConfiguration()
                .WriteTo.Logger(ReadRedLoggerConfigurations)
                .WriteTo.Logger(ReadBlueLoggerConfigurations)
                .CreateLogger();

                //Log to sub-loggers
                var log = Log.Logger;
                log.ForContext(new PropertyEnricher("isBlue", true)).Information("Logging to blue sub-logger.");
                log.ForContext(new PropertyEnricher("isRed", true)).Information("Logging to red sub-logger.");
            }
            catch (Exception exception)
            {

                Log.Fatal(exception, "Build Error.");
            }
            finally
            {
                Log.CloseAndFlush();
            } 
        }

        private static void ReadRedLoggerConfigurations(LoggerConfiguration loggerConfiguration)
        {
            const string redLogger = "red";
            loggerConfiguration.ReadFrom.AppSettings(redLogger);
        }

        private static void ReadBlueLoggerConfigurations(LoggerConfiguration loggerConfiguration)
        {
            const string blueLogger = "blue";
            loggerConfiguration.ReadFrom.AppSettings(blueLogger);
        }
    }
}
```

Now we can test our console application to confirm that the correct configurations are being applied to each sub -logger. If you run the application, then two log files should be generated on C:\Log

![Log Files](/post/2020/configure-serilog-sub-logger-from-appsettings/logfilesincdrive.PNG)

and if we open the red log file you should see the following log lines.
```json
{
    "@t":"2020-09-01T00:52:06.1676551Z",
    "@mt":"Logging to red sub-logger.",
    "isRed":true
}
```
Congratulations, you have successfully configured two different Serilog sub-loggers through XML configuration.
