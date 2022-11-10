---
title: Preventing Invalid Assembly Dependencies
tags: [.NET]
author: "Yunier"
date: "2022-03-12"
description: "Avoid having invalid assembly references with a Unit Test."
---

.NET makes it super simple to update the dependencies of a project. If you are following a solution structure like [Clean Architecture](https://github.com/ardalis/CleanArchitecture#design-decisions-and-dependencies) where the [Web project](https://github.com/ardalis/CleanArchitecture#the-web-project) should not be referenced by the [Core project](https://github.com/ardalis/CleanArchitecture#the-core-project) or you have created your own solution structure that requires certain projects do not reference another project then you might need a way to avoid having developers incorrectly adding dependencies.

![clean architecture project dependencies](/post/2022/preventing-invalid-assembly-dependencies/clean-architecture-projet-dependencies.png)

> The diagram above gives a high-level view of all project dependencies in a Clean Architecture solution. Built with [Excalidraw](https://excalidraw.com/).

You can prevent the issue described above if you have a strong continuous integration pipeline by having a unit test that asserts that a given project is not referenced by another project, in our example above, you may want to assert that the web project does not reference the core project. The [Chinook JSON:API](https://chinook-jsonapi.herokuapp.com/) project that I have been working on follows the Clean Architecture project structure. To demonstrate, I will create a unit test that asserts that the core project in the Chinook solution never references the infrastructure project.

Before I start writing the Unit Test, I would like to share the csproj file for the [Chinook.Core](https://github.com/circleupx/Chinook/tree/master/src/Chinook.Core) project. As you can see from the configuration below, there is no reference to any other assemblies, our unit test should pass when executed. Also as a reminder, when the solution was first made, a project named [Chinook.Core.Test](https://github.com/circleupx/Chinook/tree/master/test/Chinook.Core.Test) was created to host all unit tests having to do with the core project. This project already has [XUnit](https://xunit.net/) and [FluentAssertions](https://fluentassertions.com/) installed. You will need to use FluentAssertions or a library that offers similar capabilities.

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="JsonApiFramework.Server" Version="2.8.0" />
    <PackageReference Include="MediatR" Version="9.0.0" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc.NewtonsoftJson" Version="6.0.2" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="6.0.2">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="6.0.2" />
    <PackageReference Include="Serilog" Version="2.10.0" />
    <PackageReference Include="Serilog.AspNetCore" Version="4.1.0" />
    <PackageReference Include="SimplePatch" Version="3.0.1" />
  </ItemGroup>

</Project>
```

The unit test itself is rather simple, I just need to grab the assembly of each project and use the [NotReference](https://fluentassertions.com/assemblies/) function in FluentAssertions to assert that there is no reference between each assembly.

```c#
public class ProjectDependencies
{
  [Fact]
  public void CoreProject_Dependencies_ShouldNotContainAReferenceToInfrastructureProject()
  {
    var coreProjectAssembly = typeof(Chinook.Core.ChinookJsonApiDocumentContext).Assembly;
    var infrastructureProjectAssembly = typeof(Chinook.Infrastructure.Database.ChinookDbContext).Assembly;

    coreProjectAssembly.Should().NotReference(infrastructureProjectAssembly);
  }
}
```

As expected, the test passed when I executed dotnet test on the [Chinook.Core.UnitTest.csproj](https://github.com/circleupx/Chinook/blob/master/test/Chinook.Core.Test/Chinook.Core.UnitTest.csproj) file. With this test being executed on a continuous integration pipeline on every build I can be sure that nobody will accidentally adds the Infrastructure project as a dependency of the Core project.

You can use this technique in your own solutions to do the same. I've found this solution to be better than documenting the project dependencies on a README file or any other type of documentation because there is no guarantee that a developer will ever read or see that documentation.
