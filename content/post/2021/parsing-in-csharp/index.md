---
title: Parsing in C#
tags: [Parsing]
author: "Yunier"
date: "2021-01-17"
description: "Guide on how to parse strings on C#"
---

I am currently building a [JSON:API](https://jsonapi.org/) driven API on [.NET 5](https://dotnet.microsoft.com/download/dotnet/5.0), the project is called [Chinook](https://github.com/circleupx/Chinook) after the Sqlite Chinook project. The API is mature enough for me to introduce [filtering](https://jsonapi.org/format/#fetching-filtering) via the [Filter](https://jsonapi.org/recommendations/#filtering) query parameter used in JSON:API.

I would like to support dynamic filtering, I want to avoid creating nested if-else/switch statements that check if a given input is part of the filter criteria, and if it is then it gets appended to a filtering clause. For example, take the following API request, it uses the [OData](https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part2-url-conventions.html#_Toc31361038) filter syntax.

```text
https://api.example.com/users?filter=name eq 'James'
```

Some developers might choose to handle that request by implement filtering like this,

```c#
public class UserDataHandler
{
    public async Task<IEnumerable<Users>> Handle(GetUsersResourceCollectionCommand request, CancellationToken cancellationToken)
    {
        var uriQueryString = request.QueryString;
        var query = await _chinookDbContext.Users
            .ToListAsync(cancellationToken);

        if(uriQueryString.Contains("name"))
        {
            var parsedUri = Microsoft.AspNetCore.WebUtilities.QueryHelpers.ParseQuery(uriQueryString);
            var value = parsedUri["name"];
            query.where(u =>u.Where(u => u.FirstName.Contains(value) || u.LastName.Contains(value)))
        }
    }
}
```

Each additional filter on the API is another if-else/switch statement. We should avoid this type of code, simply because .NET gives us the power to create better solutions through [expressions trees](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/expression-trees/). Expression trees represent code in a tree-like data structure, where each node is an expression, for example, a method call or a binary operation such as x < y. You can compile and run code represented by expression trees. This enables dynamic modification of executable code, the execution of LINQ queries in various databases, and the creation of dynamic queries.

The first step in creating an expression tree is being able to parse a sequence of characters into a stream of tokens, this process is better known as **lexical analysis** or a **lexer**. Sometimes it is also refer to as a **tokenizer**. The **tokens** in this process are nothing more than a key value pair, in our example above, 'James' would be a token of type string, with a corresponding value of James.

The question now is, how to parse strings in C#, something like 2 * 3 + 8. For that, I will create a parser based on a [regex parser created](https://jack-vanlightly.com/blog/2016/2/24/a-more-efficient-regex-tokenizer) by [Jack Vanlightly](https://jack-vanlightly.com/). The source code to his parser can be found on his [GitHub](https://github.com/Vanlightly/DslParser) page.

To build this parser I will have to define the tokens it supports. For our expression, 2 * 3 + 8 I will only need four token types, an integer, summation, multiplication and a sequence terminator type. The following Enum can be used to represent those tokens.

```c#
public enum TokenType 
{
    Sum = 0,
    Number = 1,
    Multiplication = 2,
    SequenceTerminator = 4
}
```

Now all I need to do is combine this enum with the tokenizer class created by Jack Vanlightly and I end up with the following code.

```c#
class Program
{
    static void Main(string[] args)
    {
        var testExpression = "2*3+8";
        var precedenceBasedRegexTokenizer = new PrecedenceBasedRegexTokenizer();
        var tokens = precedenceBasedRegexTokenizer.Tokenize(testExpression);
        foreach (var token in tokens)
        {
            Console.WriteLine($"Token type {token.TokenType} has value {token.Value}");
        }
    }
}

public enum TokenType 
{
    Sum = 0,
    Number = 1,
    Multiplication = 2,
    SequenceTerminator = 4
}

public class TokenDefinition
{
    private Regex _regex;
    private readonly TokenType _returnsToken;
    private readonly int _precedence;

    public TokenDefinition(TokenType returnsToken, string regexPattern, int precedence)
    {
        _regex = new Regex(regexPattern, RegexOptions.IgnoreCase | RegexOptions.Compiled);
        _returnsToken = returnsToken;
        _precedence = precedence;
    }

    public IEnumerable<TokenMatch> FindMatches(string inputString)
    {
        var matches = _regex.Matches(inputString);
        for (int i = 0; i < matches.Count; i++)
        {
            yield return new TokenMatch()
            {
                StartIndex = matches[i].Index,
                EndIndex = matches[i].Index + matches[i].Length,
                TokenType = _returnsToken,
                Value = matches[i].Value,
                Precedence = _precedence
            };
        }
    }
}

public class TokenMatch
{
    public TokenType TokenType { get; set; }
    public string Value { get; set; }
    public int StartIndex { get; set; }
    public int EndIndex { get; set; }
    public int Precedence { get; set; }
}

public class Token
{
    public Token(TokenType tokenType)
    {
        TokenType = tokenType;
        Value = string.Empty;
    }

    public Token(TokenType tokenType, string value)
    {
        TokenType = tokenType;
        Value = value;
    }

    public TokenType TokenType { get; set; }
    public string Value { get; set; }

    public Token Clone()
    {
        return new Token(TokenType, Value);
    }
}

public class PrecedenceBasedRegexTokenizer
{
    private List<TokenDefinition> _tokenDefinitions;

    public PrecedenceBasedRegexTokenizer()
    {
        _tokenDefinitions = new List<TokenDefinition>
        {
            new TokenDefinition(TokenType.Multiplication, @"[*]", 1),
            new TokenDefinition(TokenType.Sum, @"[+]", 1),
            new TokenDefinition(TokenType.Number, "\\d+", 2)
        };
    }

    public IEnumerable<Token> Tokenize(string lqlText)
    {
        var tokenMatches = FindTokenMatches(lqlText);

        var groupedByIndex = tokenMatches.GroupBy(x => x.StartIndex)
            .OrderBy(x => x.Key)
            .ToList();

        TokenMatch lastMatch = null;
        for (int i = 0; i < groupedByIndex.Count; i++)
        {
            var bestMatch = groupedByIndex[i].OrderBy(x => x.Precedence).First();
            if (lastMatch != null && bestMatch.StartIndex < lastMatch.EndIndex)
                continue;

            yield return new Token(bestMatch.TokenType, bestMatch.Value);

            lastMatch = bestMatch;
        }

        yield return new Token(TokenType.SequenceTerminator);
    }

    private List<TokenMatch> FindTokenMatches(string lqlText)
    {
        var tokenMatches = new List<TokenMatch>();

        foreach (var tokenDefinition in _tokenDefinitions)
            tokenMatches.AddRange(tokenDefinition.FindMatches(lqlText).ToList());

        return tokenMatches;
    }
}
```

Running the code above yields the following result.

```text
Token type Number has value 2
Token type Multiplication has value *
Token type Number has value 3
Token type Sum has value +
Token type Number has value 8
Token type SequenceTerminator has value
```

Great! I am able to parse an input string into tokens, the time has come to build an expression tree from these token. If you break down the expression 2 * 3 + 8 into a tree it would like this.

![Expression Tree Structure](/post/2021/parsing-in-csharp/expression-tree.png)

The above structure is what we are aiming to build. Once the expression tree has been created I can compile it and execute it. If everything is done correctly, we should get 14 as our result. It is time to use perhaps one of the least used design patterns, yet one of the most powerful patterns, I'm talking about the [Visitor Design pattern](https://refactoring.guru/design-patterns/visitor). The visitor pattern or visitation pattern will be used to visit each node on our tree, the visitor will traverse the tree, in the end it will output our compile expression which can then be invoked to get our our result.

The visitation pattern is heavily used in frameworks like EF Core. [This](https://www.youtube.com/watch?v=r69ZxXgOIK4) video by [Shay Rojansky](https://twitter.com/shayrojansky) explains how EF Core uses expression tree to generate SQL statements.

The first step in implementing the visitor pattern is to define our Visitor interface and what is often called the "element" interface. The Element interface declares a method for “accepting” visitors. This method should have one parameter declared with the type of the visitor interface.

```c#
// The Visitor interface declares a set of visiting methods that can take concrete elements of an object structure as
// arguments.
public interface IExpressionTreeVisitor
{
    public IExpression LeftChildNode { get; set; }
    public IExpression RightChildNode { get; set; }
    public IExpression ParentNode { get; set; }

    void VisitLeafNode(Number integer);
    void VisitParentNode(Multiplication addition);
    void VisitParentNode(Addition multiplication);
}

// The accepting interface, declares a method for accepting visitors.
interface IExpression
{
    void Accept(IExpressionTreeVisitor visitor);
}
```

Additionally, I need to create a class that represents each the nodes on the tree.

```c#
public class Number : IExpression
{
    public double Value { get; }

    public Number(double value)
    {
        Value = value;
    }

    public void Accept(IExpressionTreeVisitor visitor)
    {
        visitor.VisitLeafNode(this);
        if (visitor.ParentNode == null)
        {
            return;
        }

        visitor.ParentNode.Accept(visitor);
    }
}

public class Addition : IExpression
{
    public Addition()
    {

    }
    public void Accept(IExpressionTreeVisitor visitor)
    {
        visitor.VisitParentNode(this);
    }
}

public class Multiplication : IExpression
{
    public Multiplication()
    {

    }
    public void Accept(IExpressionTreeVisitor visitor)
    {
        visitor.VisitParentNode(this);
    }
}
```

Finally, here is the implementation of the IExpressionTreeVisitor interface.

```c#
public class ExpressionTreeVisitor : IExpressionTreeVisitor
{
    public IExpression LeftChildNode { get; set; }
    public IExpression RightChildNode { get ; set ; }
    public IExpression ParentNode { get; set; }

    private Expression _leftChildExpressionValue;
    private Expression _rightChildExpressionValue;

    public void VisitLeafNode(Number number)
    {
        if (_leftChildExpressionValue == null)
        {
            LeftChildNode = number;
            _leftChildExpressionValue = Expression.Constant(number.Value);
        }
        else
        {
            RightChildNode = number;
            _rightChildExpressionValue = Expression.Constant(number.Value);
        }
    }

    public void VisitParentNode(Multiplication multiplication)
    {
        if (ParentNode == null)
        {
            ParentNode = multiplication;
            return;
        }

        var expression = Expression.Multiply(_leftChildExpressionValue, _rightChildExpressionValue);
        ParentNode = null;
        _leftChildExpressionValue = expression;
    }

    public void VisitParentNode(Addition addition)
    {
        if (ParentNode == null)
        {
            ParentNode = addition;
            return;
        }

        var expression = Expression.Add(_leftChildExpressionValue, _rightChildExpressionValue);
        ParentNode = null;
        _leftChildExpressionValue = expression;
    }

    public Expression<Func<double>> GetCompletedExpression()
    {
        return Expression.Lambda<Func<double>>(_leftChildExpressionValue);
    }
}
```

and here is the complete Program.cs class utilizing the expression visitor class.

```c#
static void Main(string[] args)
{
    var testExpression = "2*3+8";
    var precedenceBasedRegexTokenizer = new PrecedenceBasedRegexTokenizer();
    var tokens = precedenceBasedRegexTokenizer.Tokenize(testExpression);
    var listOfExpressions = new List<IExpression>();
    foreach (var token in tokens)
    {
        Console.WriteLine($"Token type {token.TokenType} has value {token.Value}");
        switch (token.TokenType)
        {
            case TokenType.Addition:
                var sumNode = new Addition();
                listOfExpressions.Add(sumNode);
                break;
            case TokenType.NumberLiteral:
                var numberNode = new Number(Convert.ToDouble(token.Value));
                listOfExpressions.Add(numberNode);
                break;
            case TokenType.Multiplication:
                var multiplicationNode = new Multiplication();
                listOfExpressions.Add(multiplicationNode);
                break;
            case TokenType.SequenceTerminator:
                break;
            default:
                break;
        }
    }

    var visitor = new ExpressionTreeVisitor();
    foreach (var item in listOfExpressions)
    {
        item.Accept(visitor);
    }

    var completedExpression = visitor.GetCompletedExpression();
    var result =  completedExpression.Compile();
    Console.WriteLine($"Expression {testExpression} is equal to {result()}");
}
```

If I run the program I get the following console output.

```text
Token type NumberLiteral has value 2
Token type Multiplication has value *
Token type NumberLiteral has value 3
Token type Addition has value +
Token type NumberLiteral has value 8
Token type SequenceTerminator has no value
Expression 2*3+8 is equal to 14
```

Awesome, 14 was the final result obtained after executing the expression tree. I now have a foundation for creating dynamic filtering on JSON:API. By the way, a lot of the work done here can be bypass by using some of the awesome projects found in the .NET community. When it comes to parsing you have two great options, one is [superpower](https://github.com/datalust/superpower) and another one is [pidgin](https://github.com/benjamin-hodgson/Pidgin). Personally, I use superpower, it is what [serilog](https://github.com/serilog/serilog) uses to do parsing, plus I am a huge fan of any work done by [Nicholas Blumhardt](https://github.com/nblumhardt). For my JSON:API project I will use Pidgin, I want to understand how Pidgin works, and understand all of its capabilities and how it differs from Superpower.

The complete code above can be found [here](https://github.com/circleupx/tokenizer).

**Credits:**

- [https://ruslanspivak.com/lsbasi-part1/](https://ruslanspivak.com/lsbasi-part1/)
- [https://codereview.stackexchange.com/questions/108001/implementation-of-the-visitor-pattern](https://codereview.stackexchange.com/questions/108001/implementation-of-the-visitor-pattern)
- [https://jack-vanlightly.com/blog/2016/2/24/a-more-efficient-regex-tokenizer](https://jack-vanlightly.com/blog/2016/2/24/a-more-efficient-regex-tokenizer)
- [https://www.youtube.com/watch?v=t4dYy6P3JuA](https://www.youtube.com/watch?v=t4dYy6P3JuA)
- [https://www.youtube.com/watch?v=klHyc9HQnNQ](https://www.youtube.com/watch?v=klHyc9HQnNQ)
- [https://thesharperdev.com/c-design-patterns-the-visitor-pattern/](https://thesharperdev.com/c-design-patterns-the-visitor-pattern/)
- [https://refactoring.guru/design-patterns/visitor](https://refactoring.guru/design-patterns/visitor)