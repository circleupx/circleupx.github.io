---
title: Sqlite Error No Such Table
layout: post
categories: [Sqlite, Unit Test, Errors, WebApplicationFactory, EF Core, .NET Core]
image: /assets/img/sqlite/sqlite.PNG
description: "How to fix SQlite error no such table exception."
---

Are you using SQlite as an in-memory provider with EF Core in your Unit/Integration test? If you are, you may come across the following exception when creating a database schema.

![EFCore ToView Note](/assets/img/sqlite/pet-exception.PNG)

As you can see from the exception, the error is **"SQlite Error 1: 'no such table vPet'"** which is odd because vPet has been defined as a sql view on my DbContext, not a sql table.

This is my DbContext.
```c#
public class PetsDbContext : DbContext
{
    // Rest of code omitted for brevity
    public DbSet<Pet> Pets { get; set; }

    public PetsDbContext()
    {

    }

    public PetsDbContext(DbContextOptions<PetsDbContext> options) : base(options)
    {

    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Rest of code omitted for brevity
        modelBuilder.ApplyConfiguration(new PetConfiguration())
    }
}
```
here is the entity model for Pet.

```c#
public class Pet
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

and finally, here is my entity type configuration class for Pet.

```c#
public class PetModelConfiguration : IEntityTypeConfiguration<Pet>
{
    public void Configure(EntityTypeBuilder<Pet> builder)
    {
        builder.HasNoKey();

        builder.ToView("vPet");

        builder.Property(p => p.Id)
            .IsRequired()
            .HasMaxLength(50);
    }
}
```

Notice the usage of **.ToView()** on my entity type configuration class, per the [EF Core documentation](https://docs.microsoft.com/en-us/ef/core/modeling/keyless-entity-types?tabs=data-annotations#mapping-to-database-objects), the method .ToView assumes that the view vPet has been created prior to the execution of **EnsuredCreated** or **EnsureCreatedAsync**.

![EFCore ToView Note](/assets/img/sqlite/efcore-toview-note.PNG)

```c#
[Fact]
public async Task Test_UsingSqliteInMemoryProvider()
{
    var options = new DbContextOptionsBuilder<PetsDbContext>()
        .UseSqlite("DataSource=:memory:")
        .Options;

    using (var context = new PetsDbContext(options))
    {
        // This fails to create vPet view.
        await context.Database.EnsureCreatedAsync();
    }
}
```

In other words, any entity configuration that utilizes .ToView() on your DbContext will not generate a view. This is why SQlite is throwing the error "SQlite Error 1: 'no such table vPets'". To get around this problem, you can write a script that generates the missing View in SQlite, for example.

```c#

[Fact]
public async Task Test_UsingSqliteInMemoryProvider()
{
    var options = new DbContextOptionsBuilder<PetsDbContext>()
        .UseSqlite("DataSource=:memory:")
        .Options;

    using (var context = new PetsDbContext(options))
    {
        // vPet will now be created
        context.Database.ExecuteSqlRaw("CREATE VIEW vPets as p.Id, p.[Name] FROM Pet p");
        await context.Database.EnsureCreatedAsync();
    }
}
```
Now whenever EnsureCreatedAsync creates a new database schema, vPets will be included.