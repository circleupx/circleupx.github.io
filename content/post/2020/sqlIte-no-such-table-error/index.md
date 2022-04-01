---
title: Sqlite - No such table error
tags: [Sqlite, EF Core]
author: "Yunier"
date: "2020-09-19"
description: "Guide on how to handle SQLite error on Unit/Integration test"
---

Are you using SQLite as an in-memory provider for EF Core on your Unit/Integration test? If you are, you may come across the following exception when creating the in-memory database.

![EFCore ToView Note](/post/2020/sqlIte-no-such-table-error/pet-exception.PNG)

As you can see from the exception, the error is **"SQLite Error 1: 'no such table vPet'"** which is odd because vPet is defined as a SQL view on my DbContext, not a SQL table.

Here is my PetsDbContext.
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
and my entity model for Pet.

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

Notice the usage of **.ToView()** on my entity configuration class, per the [EF Core documentation](https://docs.microsoft.com/en-us/ef/core/modeling/keyless-entity-types?tabs=data-annotations#mapping-to-database-objects), the method .ToView assumes that the database object vPet has been created outside of the execution **EnsuredCreated** or **EnsureCreatedAsync**.

![EFCore ToView Note](/post/2020/sqlIte-no-such-table-error/efcore-toview-note.PNG)

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

In other words, any entity configuration that utilizes .ToView() on your DbContext will not generate a corresponding SQL view. This is why SQLite is throwing the error "SQLite Error 1: 'no such table vPets'". To get around this problem, you can write a script that generates the missing View in SQLite, for example.

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