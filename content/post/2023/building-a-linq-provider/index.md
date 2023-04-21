---
title: Building a LINQ provider
tags: [LINQ]
author: "Yunier"
date: "2023-03-24"
description: "Exploring how to create a LINQ provider"
published: false
---

Everyone that uses .NET is familiar with Entity Framework, the ORM from Microsoft. The team behind Entity Framwork has bi-weekly community stand ups hosted on [Youtube](https://www.youtube.com/watch?v=pmnHGWYpCfg&list=PLdo4fOcmZ0oX0ObHwBrJ0vJpZ7PiYMqeA). These stands cover many different topics related to Entity Framework, from [pagination](https://www.youtube.com/watch?v=DIKH-q-gJNU), [performance](https://www.youtube.com/watch?v=VgNFFEqwZPU&list=PLdo4fOcmZ0oX0ObHwBrJ0vJpZ7PiYMqeA&index=46), [triggers](https://www.youtube.com/watch?v=Gjys0Yebobk&list=PLdo4fOcmZ0oX0ObHwBrJ0vJpZ7PiYMqeA&index=41) and so on. 

The last community stand up tiled [EF Core internals: IQueryable, LINQ and the EF Core query pipeline](https://www.youtube.com/watch?v=1Ld3dtnTrMw&list=PLdo4fOcmZ0oX0ObHwBrJ0vJpZ7PiYMqeA&index=2) explore the inner working of EF Core translate your LINQ queries to SQL, the video reminded of a talk from Dotnetos Conference 2019 by [Shay Rojansky](https://twitter.com/shayrojansky) titled [How Entity Framework translates LINQ all the way to SQL](https://www.youtube.com/watch?v=r69ZxXgOIK4) where he explains again how EF takes your LINQ queries and translates them into proper SQL. 

One thing that stood out for my during the [EF Core internals: IQueryable, LINQ and the EF Core query pipeline](https://www.youtube.com/watch?v=1Ld3dtnTrMw&list=PLdo4fOcmZ0oX0ObHwBrJ0vJpZ7PiYMqeA&index=2) stand up was when Shay started to explain the way a LINQ provider works, it is around the 33rd minute mark, which made me think, how hard is it to build my own LINQ provider? Let's explore that.

The first step in my journey will be to implement IQueryable, I can create a custom class for that as shown on the code below.

```c#
public class MyCustomQueryable<TElement> : IQueryable<TElement>
{
    public Type ElementType => throw new NotImplementedException();

    public Expression Expression => throw new NotImplementedException();

    public IQueryProvider Provider => throw new NotImplementedException();

    public IEnumerator<TElement> GetEnumerator()
    {
        throw new NotImplementedException();
    }

    IEnumerator IEnumerable.GetEnumerator()
    {
        throw new NotImplementedException();
    }
}
```

Expression represents the expression that needs to be translated by the provider, the Provider is responsible for enumarating the data being asked by MyCustomQueryable

