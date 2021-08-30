---
title: The order of Interfaces impacts performace
tags: [Interfaces, Performance, .NET, BenchmarkDotNet, Performance]
readtime: true
published: true
---

I was looking through some of my bookmarked Github issues when I rediscovered issue [#32488](https://github.com/dotnet/runtime/pull/32488), in that issue a [comment](https://github.com/dotnet/runtime/pull/32488#discussion_r380818002) was made that caught my attention. The comment stated that in .NET the order of interfaces impacts performance. This is because in the [.NET CLR](https://docs.microsoft.com/en-us/dotnet/standard/clr) all class definitions have a collection of methods and interface definitions. Casting is a linear search that walks the interface definition. If you are constantly casting to an Interface located at the end then the CLR must do a longer walk.

To see how much of it can impact performance I will use [BenchmarkDotNet](https://benchmarkdotnet.org/). In case you didn't know, BenchmarkDotnet is an open-source project that helps you track benchmarks and track the performance of your code. It is an extremely useful project, it is used by Entity Framework, ASP.NET Core itself, Newtonsoft.Json, Autofac, MediatR, SignalR, Serilog, and so on. 

{: .notice--info}
If you want to learn how to use BenchmarkDotNet then I suggest checking out Tim Corey's [intro video](https://www.youtube.com/watch?v=mmza9x3QxYE). 

To start the benchmarking experiment I will add a bunch of empty interfaces inside a new console application. The interfaces are as follows.

```c#
public interface IDog { }
public interface ICat { }
public interface IHorse { }
public interface IDuck { }
public interface ICow { }
public interface ISpider { }
public interface ITiger { }
public interface ILion { }
public interface IHuman { }
public interface IMonkey { }
public interface IDeer { }
public interface IHog { }
public interface IChicken { }
public interface IDonkey { }

```

Next, I am going to add two new classes, one that will execute BechmarkDotNet and the other one will be our base object, I'll call it CanWalk, given that all the interfaces define above are for an animal that can walk.

```c#
public class InterfaceBenchmarks 
{
    [Benchmark]
    public void CastToCatFirstInterface()
    {
        var walkable = new CanWalk();

        for (int i = 0; i < 1000000; i++)
        {
            _ = (ICat) walkable;
        }
    }

    [Benchmark]
    public void CastToDonkeyLastInterface()
    {
        var walkable = new CanWalk();
        for (int i = 0; i < 1000000; i++)
        {
            _ = (IDonkey) walkable;
        }
    }
}

public class CanWalk : ICat, IHorse, IMonkey, ICow, ITiger, ILion, IHuman, IDuck, ISpider, IDeer, IHog, IChicken, IDonkey
{

}
```

Next, the main method of the console application needs to be updated by calling the BenchmarkDotnetRunner.

```c#
class Program
{
    static void Main(string[] args)
    {
        BenchmarkRunner.Run<InterfaceBenchmarks>();
    }
}
```

OK, it is time to execute BenchmarkDotNet, and our results are...

![BenchmarkDonetResults](/assets/img/interfaces-performance.PNG)

Wait, what? That is not exactly what I was expecting. Well, it is what I was expecting, it confirms the comment made on the issue [#32488](https://github.com/dotnet/runtime/pull/32488), I'm just not sure if it is something worth optimizing for given how small the performance difference is between the first and second test. I reran the test and I ended up with the same result. So, while the order of interfaces matters when casting, I would argue that you should not worry about it too much. Having the interface at the start or end will probably have little impact on the overall performance of your application. 

That is all for now, thanks for reading, until next time.