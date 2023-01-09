---
title: Using AutoFixture
tags: [.NET]
author: "Yunier"
date: "2023-01-02"
description: "Using AutoFixture to simplify unit testing."
---

I enjoy writing unit tests and any tools that make writing tests easier are appreciated. For the last year, I have incorporated [AutoFixture](https://github.com/AutoFixture/AutoFixture/) into all of my unit tests. I have found AutoFixture to be an excellent tool, it changed the way I approach the "Arrange" phase. 

Previously, my arrange phase involved manually assigning values to properties, in a small class that is referenced by a few tests, you may tolerate manually assigning values. Once you start to deal with a class that has many properties such as nested properties as shown on the "Employee" class below, things get out of hand.

```c#
public class Employee
{
    public Employee()
    {
        Customers = new HashSet<Customer>();
        InverseReportsToNavigation = new HashSet<Employee>();
    }

    public long EmployeeId { get; set; }
    public string LastName { get; set; }
    public string FirstName { get; set; }
    public string Title { get; set; }
    public long? ReportsTo { get; set; }
    public byte[] BirthDate { get; set; }
    public byte[] HireDate { get; set; }
    public string Address { get; set; }
    public string City { get; set; }
    public string State { get; set; }
    public string Country { get; set; }
    public string PostalCode { get; set; }
    public string Phone { get; set; }
    public string Fax { get; set; }
    public string Email { get; set; }
    public virtual Employee ReportsToNavigation { get; set; }
    public virtual ICollection<Customer> Customers { get; set; }
    public virtual ICollection<Employee> InverseReportsToNavigation { get; set; }
}
```

Under the hood, AutoFixture uses a [faking library](https://github.com/fsprojects/FAKE) that generates random values based on the property type.

Here is a quick example.

```c#
public class UnitTestingExample
{
    [Fact]
    public void AutoFixtureExmaple()
    {
        var fixture = new Fixture();
        var employeeFixture = fixture.Create<Employee>();
        Console.WriteLine(employeeFixture.FirstName) // f17b08e4-efb0-40b8-9b6b-c937d8032c8c
        
    }
}
```

As shown above, AutoFixture will assign random values to each property, but in a unit test you would want to control the value, as shown in the following code snippet, you can use AutoFixture to assign a value to a property. 

```c#
[Fact]
public void UnitTestingExample()
{
    var fixture = new Fixture();
    fixture.Customize<Employee>(e =>
    
        e.With(p => p.FirstName, "John")
    );

    var employeeFixture = fixture.Create<Employee>();
    Console.WriteLine(employeeFixture.FirstName) // John
}
```

In the code above, the property FirstName is assigned to the value "John" using the customization API. Now whenever you ask AutoFixture to create an instance or many instances of the Employee class, all instances will have "John" as the value for the property FirstName. 

When working with a property, and that property is an object with its own properties you might attempt to create the following code.

```c#
[Fact]
public void UnitTestingExample()
{
    var fixture = new Fixture();
    fixture.Behaviors.Add(new OmitOnRecursionBehavior());
    fixture.Customize<Employee>(e =>
    
        e.With(p => p.ReportsToNavigation.FirstName, "Benjamin") // Throws an exception.
    )
    var employeeFixture = fixture.Create<Employee>();
    Console.WriteLine(employeeFixture.FirstName)
}
```

The code above will lead to an exception being thrown, instead, use the customize API again to assign a value to the nested property, create the fixture, then use the customize API again on the property of the parent object, and use the created fixture as the value. 

```c#
public void UnitTestingExample()
{
    var fixture = new Fixture();
    fixture.Customize<Employee>(e => e.With(p => p.FirstName, "John"));
    var employeeFixture1 = fixture.Create<Employee>()
    fixture.Customize<Employee>(e =>
    {
       return e.With(p => p.FirstName, "Benjamin")
               .With(p => p.ReportsToNavigation, employeeFixture1)
    });
    
    var employeeFixture2 = fixture.Create<Employee>();
    Console.WriteLine(employeeFixture1.FirstName); // John 
    Console.WriteLine(employeeFixture2.FirstName); // Benjamin
}
```

AutoFixture can also create collections. 

```c#
[Fact]
public void UnitTestingExample()
{
    var fixture = new Fixture();
    fixture.Behaviors.OfType<ThrowingRecursionBehavior>()
        .ToList()
        .ForEach(b => fixture.Behaviors.Remove(b));
    fixture.Behaviors.Add(new OmitOnRecursionBehavior())
    var employeeFixture = fixture.CreateMany<Employee>();
    var a = employeeFixture.Count();
    Console.WriteLine(employeeFixture.Count()); // 3
}
```

You can specify the total number of objects to be created by passing an amount as an argument to CreateMany.

```c#
[Fact]
public void UnitTestingExample()
{
    var fixture = new Fixture();
    fixture.Behaviors.OfType<ThrowingRecursionBehavior>()
        .ToList()
        .ForEach(b => fixture.Behaviors.Remove(b));
    fixture.Behaviors.Add(new OmitOnRecursionBehavior())
    var employeeFixture = fixture.CreateMany<Employee>(100);
    var a = employeeFixture.Count();
    Console.WriteLine(employeeFixture.Count()); // 100
}
```

One flaw with the code I have shown is that if you are writing many tests that share the fixture configuration then the tests become hard to maintain. As documented in [Keep your unit tests DRY with AutoFixture Customizations](https://megakemp.com/2011/12/15/keep-your-unit-tests-dry-with-autofixture-customizations/) by [Enrico Campidoglio](https://megakemp.com/about/) you can centralize same configurations into a class.


```c#
public class UnitTestingExample
{
    [Fact]
    public void Test1()
    {
        var fixture = new Fixture();
        fixture.Customize(new EmployeeCustomization());
        var employeFixture = fixture.Create<Employee>();
        Console.WriteLine(employeFixture.FirstName); // John
    }
}

public class EmployeeCustomization : ICustomization
{
    public void Customize(IFixture fixture)
    {
        fixture.Behaviors.Add(new OmitOnRecursionBehavior());
        fixture.Customize<Employee>(e => e.With(p => p.FirstName, "John"));
    }
}
```

With a centralized configuration class, test maintenance should be much easier. Another key feature of AutoFixure, it supports all the major .NET testing frameworks like [XUnit](https://xunit.net/), [NUnit](https://nunit.org/), [Moq](https://github.com/moq/moq) and so on.

