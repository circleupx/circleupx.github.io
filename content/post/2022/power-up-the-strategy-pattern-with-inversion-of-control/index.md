---
title: Power Up The Strategy Pattern With Inversion Of Control
tags: [Patterns]
author: "Yunier"
date: "2022-12-04"
description: "Create highly maintainable code using the strategy pattern and inversion of control."
---

If you are a fan of the [strategy pattern](https://refactoring.guru/design-patterns/strategy), and you find yourself adding a lot of conditional logic around each strategy then you should consider replacing all branching logic using inversion of control. 

Take the following code as an example. It defines a strategy for reading different file types. For simplicity, the code writes out to the console a message, in a real-world application, the logic would be far more complex, but we are not interested in that logic, rather we are interested in how the strategy pattern works and how we can improve its usage.

```c#
public interface IStrategy
{
    void RunStrategy();
}

public class TextFileStrategy : IStrategy
{
    public void RunStrategy()
    {
        Console.WriteLine("Reading text file....");
    }
}

public class PDFFileStrategy : IStrategy
{
    public void RunStrategy()
    {
        Console.WriteLine("Reading PDF file....");
    }
}

public class PNGFileStrategy : IStrategy
{
    public void RunStrategy()
    {
        Console.WriteLine("Reading PNG File....");
    }
}
```

Typically, there is a context class, this is the container class that decides which strategy to use. For our example, I created the following context class.

```c#
public class StrategyContext
{
    public void SelectStrategy(string fileExtesion)
    {
        if (fileExtesion is ".txt")
        {
            var textFileStrategy = new TextFileStrategy();
            textFileStrategy.RunStrategy();
        }
        else if (fileExtesion is ".pdf")
        {
            var pdfFileStrategy = new PDFFileStrategy();
            pdfFileStrategy.RunStrategy();
        }
        else if(fileExtesion is ".png")
        {
            var pngFileStrategy = new PNGFileStrategy();
            pngFileStrategy.RunStrategy();
        }
    }
}
```

What you see above is often shown as an example of the strategy pattern, and it is often found in real-world apps. Technically, the code above is valid and can be easily maintained if you have a few strategies. Once you start to reach more than a few strategies then the code above can become a problem as every new strategy requires adding another conditional check. 

An approach that I consider to be an improvement is to decide the strategy via inversion of control. Take the example above, instead of if/else or switch statements, the context class takes a collection of all available strategies, then based on the file extension the corresponding strategy is selected. 

Here is how the code would look.

First, the IStrategy interface is updated by adding a new AppliesTo method. The purpose of this method is to know if the strategy can satisfy the given file type by looking at the file extension.

```c#
public interface IStrategy
{
    bool AppliesTo(string fileExtension);
    void RunStrategy();
}
```

With the updated interface, each strategy is also updated.

```c#
public class TextFileStrategy : IStrategy
{
    public bool AppliesTo(string fileExtension)
    {
        return fileExtension.Equals(".txt", StringComparison.OrdinalIgnoreCase);
    }

    public void RunStrategy()
    {
        Console.WriteLine("Reading text file....");
    }
}

public class PDFFileStrategy : IStrategy
{
    public bool AppliesTo(string fileExtension)
    {
        return fileExtension.Equals(".pdf", StringComparison.OrdinalIgnoreCase);
    }

    public void RunStrategy()
    {
        Console.WriteLine("Reading PDF file....");
    }
}

public class PNGFileStrategy : IStrategy
{
    public bool AppliesTo(string fileExtension)
    {
        return fileExtension.Equals(".png", StringComparison.OrdinalIgnoreCase);
    }

    public void RunStrategy()
    {
        Console.WriteLine("Reading PNG File....");
    }
}
```

And the updated context class.

```c#
public class StrategyContext
{
    private readonly IStrategy[] _strategies;

    public StrategyContext(IStrategy[] strategies)
    {
        _strategies = strategies;
    }

    public void SelectStrategy(string fileExtesion)
    {
        var strategy = _strategies.FirstOrDefault(s => s.AppliesTo(fileExtesion));
        strategy.RunStrategy();
    }
}
```

The class now takes a collection of all concrete implementations of IStrategy. Look how much cleaner the context class has become, all the conditional logic is gone and the best part, adding a new strategy only involves adding the new strategy class via dependency injection. With this approach, the context class never has to change with the addition of a new strategy. Pretty neat.

By the way, the technique shown above can be applied whenever you find yourself working with interfaces that share similar functionality.