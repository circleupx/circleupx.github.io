---
title: Rule Engines In .NET
tags: [Rules Engine]
author: "Yunier"
date: "2023-05-21"
description: "Exploring different ways to build a rules engine."
---

## Introduction

I am working on a project that requires the usage of rules engines to enforce business rules, I am unsure if I should roll out my own custom implementation, probably a bad idea, or if I should use an existing project.

To help me make a decision I will need to look at the current options for rules engines available in .NET, I need to understand their capabilities and limitations. In .NET the most well-known rules engine is probably [NRules](https://nrules.net/), the project has been around for some years and has good documentation. I also know that Microsoft created its own rules engine, [RulesEngine](https://microsoft.github.io/RulesEngine/) back in 2019. Then per [awesome-dotnet](https://github.com/quozd/awesome-dotnet) I could use [Plastic](https://github.com/sang-hyeon/Plastic) or [Peasy.NET](https://github.com/peasy/Peasy.NET) but I opted out on looking at those projects, for my use case I think NRules or RulesEngine will do.

To determine the final solution I will create a proof of concept project that uses each project, plus my own implementation.

The first step here is to come up with a business and some business rules. My fictional business will be a pizza store, called Glorious Pizza, as the owner of Glorious Pizza I want to enforce the following business rule.

>1. On Saturdays, all pizzas should have a 10 percent discount.


## NRules
The first project I am going to test will be NRules. I have created an API project called GloriousPizza, the API will expose an endpoint to create an order. The create order request only contains a single field, total.

```JSON
{
  "total": 9.99
}
```

When the request is received by the API, NRules will be invoked and the rules I have defined above will be executed against the incoming request. Now I need to translate my rules into NRules using the [NRules DSL](https://github.com/NRules/NRules/wiki/Fluent-Rules-DSL). The rule can be translated using the following DiscountRule class.

```C#
public class DiscountRule: Rule
{
    public override void Define()
    {
        Order order = default;
        When()
            .Match(() => order) // Bind this rule to an NRule fact of type Orders
            .Match<Order>(x=>x.CreatedDate.DayOfWeek == DayOfWeek.Sunday);
           
        Then()
            .Do(ctx => ApplyDiscount(order));
    }
    private static void ApplyDiscount(Order order)
    {
        var discount = (order.Total / 100) * 10;
        order.Total = order.Total - discount;
    }
}
```

The class implements a Rule from NRules, Rule is the base class exposed by NRule it offers helper functions such as When And Then, When is used to wire up a rule to a class. In the case above the When method is used to tell NRules to first bind any matching facts, your domain class or data model, in my case that being Orders. It is very important to bind a rule to a fact, if not done correctly then when the rule evaluation kicks in then the order variable will be null, meaning that a null exception would be thrown within the ApplyDiscount. Additionally, the parameter passed in the method ApplyDiscount must have a matching name, if instead, I were to rename the parameter orders to plural, then a null reference exception would occur. The second match invocation tells NRules to only apply this rule to orders that have a created date of Sunday. 

I can test the rule by modifying the API resource controller to include the following code.

```C#
[Route("api/[controller]")]
[ApiController]
public class OrdersController : ControllerBase
{
    [HttpPost]
    public IActionResult CreateResource(Order order)
    {
        var repository = new RuleRepository();
        repository.Load(x => x.From(typeof(Program).Assembly));

        //Compile rules
        var factory = repository.Compile();
        
        //Create a working session
        var session = factory.CreateSession();
        
        session.Insert(order);
        // Execute all rules in session.
        session.Fire();
        return Created("", order);
    }
}
```

The first thing this code is doing is creating an empty Rules repository, by default the repository created by NRules acts against an in-memory database that holds all the rules. After the repository is instantiated, we load all rules in the current assembly and load them into the repository. NRule is able to add any rule that implements the Rule class. After the rules have been added you will need to compile them, all rules defined in NRules are nothing more than expression trees, expression trees are a representation of a code, and like all frameworks that deal with expression trees compilation is required to convert the expression trees into executable delegates.

After compiling all the rules, you will need to create a session,  sessions are used to ensure that rules are only applied to facts that are added to the session, and as you can tell from the code, an order fact is added to the session. Once you have finished adding all facts, you are free to fire the rules engine and execute all rules. 

Let's test it by sending the following HTTP request 

```Shell
curl -X 'POST' \
  'https://localhost:7139/api/Orders' \
  -H 'accept: */*' \
  -H 'Content-Type: application/json' \
  -d '{
  "total": 10,
  "createdDate": "2023-05-21T21:11:29.616Z"
}'
```

As a client of the API, I am creating an order with a grand total of $10, if this order is created on a Sunday, then I should get a 10 percent discount, $1, which means my grand total should actually be 9, well May 21st, 2023 happens to be a Sunday thus Sending the HTTP request to the API on Sunday yields the following response. 

```JSON
{
  "total": 9,
  "createdDate": "2023-05-21T21:11:29.616Z"
}
```

My total is now 9 instead of 10. Perfect, this is great, NRules was easy to configure and easy to work with, there is one flaw with this approach, the rules and the rules engine exist together in the same code, meaning that if I wanted to change the rule to give discounts on Saturdays instead of Sunday, I would need to modify the code and redeploy the API.

NRules does offer the [ability for you to create your own custom rules repository](https://github.com/NRules/NRules/wiki/Rule-Builder), allowing you to load rules from outside the app, the trick here is that you would be responsible for translating those rules into expressions tree which involves parsing, tokenizing the rules and then finally generating the expression tree. Not an easy task, [I've done something similar](/post/2021/parsing-in-csharp/) a few times with URL parameters, taking the incoming request, parsing it, tokenizing it then creating an expression tree to then give to an ORM framework like EF Core to do dynamic queries and even after doing it a few times I still make mistakes.

## RulesEngine

Let's now take a look at Microsoft's rules engine, [RulesEngine](https://github.com/microsoft/RulesEngine), here is the official overview of the project directly from their Wiki.

> Rules Engine is a library/NuGet package for abstracting business logic/rules/policies out of a system. It provides a simple way of giving you the ability to put your rules in a store outside  > the core logic of the system, thus ensuring that any change in rules doesn't affect the core system.

Great, that is the exact limitation I just identified with NRules. To get started with RulesEngine install the library using Nuget.

```shell
dotnet add package RulesEngine --version 4.0.0
```

With the RulesEngine package installed, I am going to migrate the rule I defined in NRules over to RulesEngine, here is what the DiscountRule class looks like after the migration.


```C#
public class DiscountRule
{
    public static List<Rule> GetSundayDiscountRules()
    {
        var rules = new List<Rule>();

        Rule sundayDiscountRule = new Rule
        {
            RuleName = "Discount Rule",
            SuccessEvent = "Discount given on a Sunday",
            ErrorMessage = "Discounts are only available on Sundays",
            Expression = "CreatedDate.DayOfWeek == Sunday",
            RuleExpressionType = RuleExpressionType.LambdaExpression
        };
        
        rules.Add(sundayDiscountRule);
        return rules;
    }
}
```

The first thing to do is to create a rules container, this list will contain all the rules associated with giving out discounts to represent this rule a Rule is instantiated, a name, and message on success and error is given, then the expression and Lambda expression type, which as of May 2023 the only options is LambdaExpression. Next, the controller will be migrated as shown below.


```C#
[Route("api/[controller]")]
[ApiController]
public class OrdersController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> CreateResourceAsync(Order order)
    {
        var discountWorkflows = new List<Workflow>();
        Workflow discountWorkFlow = new Workflow();

        discountWorkFlow.WorkflowName = "Sunday Discounts";
        discountWorkFlow.Rules = DiscountRule.GetSundayDiscountRules();
        discountWorkflows.Add(discountWorkFlow);
        
        var bre = new RulesEngine.RulesEngine(discountWorkflows.ToArray());
        var rulesResult =  await bre.ExecuteAllRulesAsync(discountWorkFlow.WorkflowName, order);
        return Created("", order);
    }
}
```

The first step is to create a container to hold all the workflows, workflows in RulesEngine are how rules are stored, it is essentially a JSON representation of the rule, for example, the Sunday discount rule we have been working with would be serialized and deserialized into the following JSON document.

```JSON
{
    "WorkflowName": "Sunday Discounts",
    "WorkflowsToInject": null,
    "GlobalParams": null,
    "Rules": [
        {
            "RuleName": "Discount Rule",
            "Properties": null,
            "Operator": null,
            "ErrorMessage": "Discounts are only available on Sundays",
            "Enabled": true,
            "RuleExpressionType": "LambdaExpression",
            "WorkflowsToInject": null,
            "Rules": null,
            "LocalParams": null,
            "Expression": "CreatedDate.DayOfWeek == Sunday",
            "Actions": null,
            "SuccessEvent": "Discount is given on a Sunday."
        }
    ]
}
```

If I were using an external storage system like a database, this is how the data would be saved to the database for this particular rule, since this example doesn't deal with external storage, the rule is represented in the code shown above. So, we have a workflow, we give it a name, then the rules from the class DiscountRule are added to it, and the workflow is then passed to the engine for evaluation. 

Notice that, unlike NRules, RulesEngine only deals with rule validation, there is no logic here that says what to do after a rule is successfully executed. I will need to add additional logic to handle updating the order total. To get the result of a rule you will need to listen to the OnSuccess event which is fire for all successful events, meaning that in your event listener, you will need additional logic to filter on a specific event, currently the only option is on the name defined in the Rule itself, in my case my OnSuccess event name is Discount given on a Sunday.

```C#
[HttpPost]
public async Task<IActionResult> CreateResourceAsync(Order order)
{
    var discountWorkflows = new List<Workflow>()
    Workflow discountWorkFlow = new Workflow();

    discountWorkFlow.WorkflowName = "Sunday Discounts";
    discountWorkFlow.Rules = DiscountRule.GetSundayDiscountRules();
    discountWorkflows.Add(discountWorkFlow)
    
    var bre = new RulesEngine.RulesEngine(discountWorkflows.ToArray());
    var rulesResult = bre.ExecuteAllRulesAsync(discountWorkFlow.WorkflowName, order).Result
    
    rulesResult.OnSuccess((eventName) =>
    {
        if (eventName == "Discount given on a Sunday")
        {
            var discount = (order.Total / 100) * 10;
            order.Total = order.Total - discount;
        };
    })
    
    return Created("", order);
}
```

Now that I am listening to the success event I can update the order total and apply my discount. Let's see if the code works, I'll run the API, and as done before I will send the following HTTP request.


```Shell
curl -X 'POST' \
  'https://localhost:7139/api/Orders' \
  -H 'accept: */*' \
  -H 'Content-Type: application/json' \
  -d '{
  "total": 0,
  "createdDate": "2023-05-21T22:47:20.415Z"
}'
```

The API returned the following response.

```JSON
{
  "total": 10,
  "createdDate": "2023-05-21T23:05:58.914Z"
}
```

I got 10 instead of 9, upon checking and debugging the code the rule was never executed, tried it again a few times with the same result. As far I could tell I had the right code and rule structure, but nothing worked, I confirmed I had the right structure by changing the rule from **"CreatedDate.DayOfWeek == Sunday"** to **"Total == 10"**, I figure that maybe RulesEngine can handle nested structure or requires additional configuration to support nested structure. Sending the same HTTP request with the new rule worked, making me confident that I had written the right code, therefore, the problem was with the expression **"CreatedDate.DayOfWeek == Sunday"**. 

<div style="width:100%;height:0;padding-bottom:75%;position:relative;"><iframe src="https://giphy.com/embed/3oz8xP6SaSkSU9dhcI" width="100%" height="100%" style="position:absolute" frameBorder="0"></iframe></div>

That's when it dawned on me, RulesEngine works with an expression tree so the expression **"CreatedDate.DayOfWeek == Sunday"** needs to be in an expression format, but what is the correct format to represent the expression? Luckily I picked up a trick from [Shay Rojansky](https://twitter.com/shayrojansky) a while back on how to do this, when in doubt about what an expression should look like use the .NET compiler, how? Simply create the expression yourself as shown in the following code.

```C#
Expression<Func<Order, bool>> x = d => d.CreatedDate.DayOfWeek == DayOfWeek.Sunday;
```

If I run the expression on the debugger and look at the DebugView I get the following code.

```C#
.Lambda #Lambda1<System.Func`2[Shared.Order,System.Boolean]>(Shared.Order $d) {
    (System.Int32)($d.CreatedDate).DayOfWeek == 0
}
```

Right, that's when I realized I am an idiot, my expression should have been **"CreatedDate.DayOfWeek == DayOfWeek.Sunday"** or **"CreatedDate.DayOfWeek == 0"** not **"CreatedDate.DayOfWeek == Sunday"**. With the correct expression now in place I get the following response.

```JSON
{
  "total": 9,
  "createdDate": "2023-05-21T23:31:46.928Z"
}
```

Great, I got the expected result. So what have we learned? Well, RulesEngine solves the main problem identified with NRules, but there are three flaws in its approach, as I just show it is very easy for someone managing the rules to put the wrong expression on a rule. This flaw leads to the second flaw, in my experience, the people that manage business rules aren't developers, they are tech-savvy, yes, but not expression trees savvy, I mean, someone managing rules with RulesEngine would need to know how to create expression tree and have an understanding in what the following means and does.

```JSON
{
    "Expression": "input1.country == \"india\" AND input1.loyaltyFactor <= 2 AND input1.totalPurchasesToDate >= 5000 AND input2.totalOrders > 2 AND input3.noOfVisitsPerMonth > 2"
}
```

I wish RulesEngine has chosen a higher level syntax grammar something close enough to the following 

```txt
When an order is placed on Sunday, give a 10 percent discount.
```

High-level grammar would allow almost anyone to be able to work add, and update rules. The third flaw I see has to do with how the OnSuccess and OnFail events are raised, I think it would have been better if instead of having an OnSuccess or an OnFail, RulesEngine allowed you to register an event type and then on OnSuccess or Failure the event type gets invoked, think of Mediatr and its notification implementation, something to that style would definitively enhance RulesEngine.

## Custom Rules Engine

Caution, this is my own custom-hand-rolled rules engine, **should you use it? No**, you should go with NRules or RulesEngine or something else in another language, after all, why limit yourself to just dotnet? I've included my own version here for educational purposes.

I have decided to create my custom rules engine based on [How to Design Software — Rules Engines](https://medium.com/swlh/how-to-design-software-rules-engines-adbb098b2d73) by [Joseph Gefroh](https://medium.com/@jgefroh). Though I would have liked to have supported having a high-level language and having the rules decoupled from the engine, the task proved to be great for this blog post. Instead, I will attempt to tackle those features in a future post.

Now let's get into the code, here is my very simple rules engine.

```C#
public class Rules <T>
{
    private readonly T _type;

    public Rules(T type)
    {
        _type = type;
    }

    internal void Validate(params (dynamic Rule, string Parameter)[] validations)
    {
        foreach ((dynamic rule, string parameter) in validations)
        {
            if (rule.Condition)
            {
                rule.Effect(_type);
            }
        }
    }
}
```

My rules engine has a single method that accepts a collection of tuples, the first parameter being the rule and the second parameter is the property this rule applies to. Here is how to use it.


```C#
public class CustomOrderRules : Rules<Order>
{
    public CustomOrderRules(Order order) : base(order)
    {
        Validate(
            (Rule: OrderCreatedOnASunday(order, applyDiscountAction), Parameter: nameof(order.CreatedDate))
            
        );
    }
 
    private static dynamic OrderCreatedOnASunday(Order order) => new
    {
        Condition = order.CreatedDate.DayOfWeek == DayOfWeek.Wednesday,
    };
}
```

The above code defines a validation rule, OrderCreatedOnASunday, for now, it only has the condition that determines if this rule should be applied, it is the same condition we have been using so far, the day of the week has to be Sunday. Next, I will need to define my Effect, that is the function that will be invoked as a result of this rule. The question is now how to best implement effects, for simplicity, I'll use a callback function as shown below.

```C#
private void ApplyDiscount(Order order)
{
    var discount = (order.Total / 100) * 10;
    order.Total = order.Total - discount;
}
```

Next, I will update the OrderCreatedOnASunday method to accept an Action as a parameter.

```C#
private static dynamic OrderCreatedOnASunday(Order order, Action<Order> action) => new
{
    Condition = order.CreatedDate.DayOfWeek == DayOfWeek.Wednesday,
    Effect = action,
};
```

Finally, the constructor needs to handle the Action assignment.

```C#
public CustomOrderRules(Order order) : base(order)
{
    Action<Order> applyDiscountAction = ApplyDiscount;

    Validate(
        (Rule: OrderCreatedOnASunday(order, applyDiscountAction), Parameter: nameof(order.CreatedDate))
        
    );
}
```

Here is the complete code.

```C#
public class CustomOrderRules : Rules<Order>
{
    public CustomOrderRules(Order order) : base(order)
    {
        Action<Order> applyDiscountAction = ApplyDiscount;
        Validate(
            (Rule: OrderCreatedOnASunday(order, applyDiscountAction), Parameter: nameof(order.CreatedDate))
            
        );
    }

    private static dynamic OrderCreatedOnASunday(Order order, Action<Order> action) => new
    {
        Condition = order.CreatedDate.DayOfWeek == DayOfWeek.Wednesday,
        Effect = action,
    };

    private void ApplyDiscount(Order order)
    {
        var discount = (order.Total / 100) * 10;
        order.Total = order.Total - discount;
    }
}
```

Great, time to test my custom rules engine, I've modified the controller to now use the CustomOrderRules class. Sending the same HTTP request as before.

```Shell
curl -X 'POST' \
  'https://localhost:7139/api/Orders' \
  -H 'accept: */*' \
  -H 'Content-Type: application/json' \
  -d '{
  "total": 10,
  "createdDate": "2023-05-21T01:39:33.272Z"
}'
```

Yields the following response.

```JSON
{
  "total": 9,
  "createdDate": "2023-05-21T01:39:33.272Z"
}
```

Perfect, that is the expected result. The custom rules engine works, and even though it suffers from the same limitations as NRules and Rules Engine, I like its simplicity and flexibility. Adding a new rule is as simple as defining a new Effect, for example.

```C#
private void OrderIsMoreThan100OfferDiscount(Order order)
{
    var discount = (order.Total / 100) * 10;
    order.Total = order.Total - discount;
}
```

Then the new rule will use that effect.

```C#
private static dynamic OrderIsMoreThan100(Order order, Action<Order> action) => new
{
    Condition = order.Total > 100,
    Effect = action,
};
```

And finally using the rule.

```C#
public CustomOrderRules(Order order) : base(order)
{
    Action<Order> applyDiscountAction = ApplyDiscount;
    Action<Order> orderIsMoreThan100Action = OrderIsMoreThan100OfferDiscount;

    Validate(
        (Rule: OrderCreatedOnASunday(order, applyDiscountAction), Parameter: nameof(order.CreatedDate)),
        (Rule: OrderCreatedOnASunday(order, orderIsMoreThan100Action), Parameter: nameof(order.Total))
    );
}
```

Though it suffers from the same issues as NRule and Rules Engine, I do hope to come back to this project to make the following improvements.

1) Decouple the rules from the engine. Facilitating storing the rules in an external system.
2) Supporting high-level grammar, something along the lines of 'When X is true then do Y'. 
3) Rules should support types, using dynamic as a POC works, but in reality, I want to offer type safety.
4) The engine should offer a fluent style API, similar to NRules.
5) The engine should support rule composition, for example, in FluentValidation you can create a validation rule that is composed of two other validation rules. I think such flexibility offers an amazing developer experience. 

Anyways, while I like NRules more, I have decided to go with Microsoft's Rule Engine simply for the fact that the rules can live independently. Thus allowing an external app like an Admin tool to manage them.

