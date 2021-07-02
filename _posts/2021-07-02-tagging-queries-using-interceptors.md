---
title: Tagging queries with interceptors.
layout: post
tags: ["EF Core", "Query Tags"]
readtime: true
published: false
---

A while back I [talked about tagging EF Core queries](https://www.yunier.dev/2020-12-09-Tagging-EF-Core-Queries/). A feature that was added back in .NET Core 2.2, it allows you add additional information metadata to an EF Core query by tagging the query. 

```c#
public async Task<IEnumerable<Customer>> Handle(GetCustomerResourceCollectionCommand request, CancellationToken cancellationToken)
{
    var tagMessage = $"Calling from {nameof(GetCustomerResourceCollectionHandler)}";
    return await _chinookDbContext.Customers.TagWith(tagMessage).ToListAsync(cancellationToken);
}
```

As seen from the example above, whenever the customer list is retrieved via EF Core, the query generated is tagged with a message. The sql statement below is the query produced by EF Core.

```sql
-- Calling from GetCustomerResourceCollectionHandler

SELECT "c"."CustomerId", "c"."Address", "c"."City", "c"."Company", "c"."Country", "c"."Email", "c"."Fax", "c"."FirstName", "c"."LastName", "c"."Phone", "c"."PostalCode", "c"."State", "c"."SupportRepId"
FROM "customers" AS "c"
```

Notice how our tag became part of the SQL statement that was executed. This type of code is useful for troubleshoot/debugging purposes. In large applications it can become hard to determine which code executed the query, it would be nice to know the exact line of code that produce a sql statement that was used against the database. That can be done using the following code.

```c#
public static class IQueryableTaggingExtensions
{
    public static IQueryable<T> TagWithSource<T>(this IQueryable<T> queryable,
        [CallerLineNumber] int lineNumber = 0,
        [CallerFilePath] string filePath = "",
        [CallerMemberName] string memberName = "")
    {
        return queryable.TagWith($"{memberName}  - {filePath}:{lineNumber}");
    }

    public static IQueryable<T> TagWithSource<T>(this IQueryable<T> queryable,
        string tag,
        [CallerLineNumber] int lineNumber = 0,
        [CallerFilePath] string filePath = "",
        [CallerMemberName] string memberName = "")
    {
        return queryable.TagWith($"{tag}{Environment.NewLine}{memberName}  - {filePath}:{lineNumber}");
    }
}
```

This is great. The only drawback is that developers would have to add this code whenever they use EF Core, it be great if instead, every EF Core query was automatically tagged. This is where an interceptor comes in. 

[EF Core interceptors](https://docs.microsoft.com/en-us/ef/core/logging-events-diagnostics/interceptors) enable interception, modification, and/or suppression of EF Core operations. This includes low-level database operations such as executing a command, as well as higher-level operations, such as calls to SaveChanges.

