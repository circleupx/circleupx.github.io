---
title: POC - Building A DSL Driven Rules Engine
layout: post
tags: [ Parsing, Pidgin, DSL, Rules Engine]
readtime: true
---

In my [previous post](/2021-01-17-Parsing-in-CSharp) I went over how to parse strings in c#. The main objective of that post was to illustrate how to create expressions tree from strings. I want to build [dynamic LINQ queries](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/expression-trees/how-to-use-expression-trees-to-build-dynamic-queries) so that I can support filtering on my [Chinook](https://github.com/circleupx/Chinook) project. Another benefit of being able to parse string into dynamic code is that we can build [Domain Specific Languages](https://www.martinfowler.com/bliki/DomainSpecificLanguage.html). 

For example, I would like to create a rules engine that is data driven. By data driven, I mean storing a string that represents a rule, for example, in my Chinook project there is a [customers](https://chinook-jsonapi.herokuapp.com/customers) resource, each customer has a firstName attribute. When a client application sends an HTTP POST to the customers resource, I would like to add some validation, somethings like **"firstName length must be greater than 1"** or **age must be greater than 13**. I want to those strings and turn them into rules, then I can execute each rule through a rules engine. The benefit of this approach is that all rules are data driven, rather and code or configuration driven. Allowing you to change or add a new rule without having to modify existing code by simply storing the rules on database then loading them to memory when the app starts or using some event-driven message broker that delivers the new/changed rule to the rules engine. 

To accomplish building a DSL Driven Rules Engine I am going to need three things. First, a syntax parser, which I have, I will use [pidgin](https://github.com/benjamin-hodgson/Pidgin), a [Rules Engine](https://en.wikipedia.org/wiki/Business_rules_engine) which I build by following the [Rules Pattern](https://www.michael-whelan.net/rules-design-pattern/), and the Domain Specific Language syntax which I will also have to build. 

{: .box-note} 
Serilog's [Filter Expressions](https://github.com/serilog/serilog-filters-expressions) is a great example of a project that uses a Domain Specific Language. Check out [this](https://nblumhardt.com/2017/01/serilog-filtering-dsl/) post by [Nicholas Blumhardt](https://nblumhardt.com/) to learn more.

Time to tackle the first goal, we need parser, as I stated before, I will use Pidgin so all I need right now is to install it on my new .NET 5 project [DSLDrivenRulesEngine](https://github.com/circleupx/DSLDrivenRulesEngine). Pidgin can be install by running the following dotnet command.

```text
dotnet add package Pidgin --version 2.5.0
```

Now I need to create a rules engine. I will use the Rules Pattern to create a simple rules engine. I'll start by creating an abstraction that represents a concrete rule. Here is the interface derived from [Steve Smith](https://ardalis.com/)'s PluralSight [course on Design Patterns](https://app.pluralsight.com/library/courses/c-sharp-design-patterns-rules-pattern/table-of-contents).

```c#
public interface IRule
{
    void ExecuteRule();
    bool IsRuleSatisfied();
}
```

![The Rule Pattern UML Diagram](../assets/img/rule-pattern.png)
Now all that is left to do is to create my Domain Specific Language. I'll keep the language simple, I don't want to introduce too much complexity at the moment, so there won't be any support for rules that evaluate objects with complex properties. For example, take the following user class.

```c#
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } 
    public Role Role { get; set; }
}

public class Role
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

The rules engine I am building will not evaluate a user's role. Again, all so that our engine can be kept everything simple. To build the DSL I need to consider the type of rules I would like to support. Using the user class above I want to support the following rules.

```text
User.Id equals 11
```

or

```text
User.Name contains 'a'
```

Equals, greater than, less than, not equals etc... are all [comparison operators](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/comparison-operators). The language syntax will need to support comparison operators, but for simplicity, I will only support equals, greater than and less than. The language syntax will also need to support rules that are a composition of two rules. For example.

```text
User.Id equals 11 and User.Name length greater than zero.
```

AND, OR, NOT, NAND, NOR, XOR are all [logical operators](https://www.computerhope.com/jargon/l/logioper.htm). For simplicity, I will only support and, or, not.

The rules will be applied to C# [types](https://docs.microsoft.com/en-us/dotnet/csharp/basic-types) and their properties, so the language syntax will need to understand how to deal with fields, it will also need to support [functions](https://docs.microsoft.com/en-us/dotnet/framework/data/adonet/ef/language-reference/canonical-functions) such as "contains". There will also be constants, like 11 or 'Jupiter' so they need to be supported as well. 

So, I have comparison expressions, logical expressions, constants, functions, and fields. The language so far can be documented like this.

| Type       | Value                                         | Example                              |
|------------|-----------------------------------------------|--------------------------------------|
| Comparison | equals, greater than, less than               | Age less than 20                     |
| Logical    | not, and, or                                  | Age less than 20 and greater than 10 |
| Constant   | 1, 'cool'                                     | Name equals 'Yunier'                 |
| Function   | contains                                      | Name.Id contains 'y'                 |
| Field      | Id                                            | User.Id                              |

Time to codify the DSL, I'll start by adding a set of classes that will represent each value on our Lexer. 

```c#

```

![Lexer and AST Diagram](../assets/img/rules-engine-parsing.png)
