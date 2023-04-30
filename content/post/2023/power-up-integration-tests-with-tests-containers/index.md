---
title: "Power Up Integration Tests with Test Containers"
tags: [WebApplicationFactory, Docker]
author: "Yunier"
date: "2023-04-23"
description: "Taking a look at testcontainers for dotnet"
---

### Introduction

In my blog post [Integration Testing Using WebApplicationFactory](/post/2021/integration-testing-using-webapplicationfactory/) I spoke about the benefits of testing a .NET Core Web API using [WebApplicationFactory](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.testing.webapplicationfactory-1?view=aspnetcore-7.0). The idea is that WebApplicationFactory creates a local HTTP server in-memory, meaning that when using WebApplicationFactory you are not mocking the HTTP request made to your API, you are actually using the API as if it were hosted in a live environment. 

The benefit here is that your test code seats in the middle of the Web API and the client code calling the API, meaning you can now test how the API behaves under certain requests from the client. One drawback of using WebApplicationFactory would be having to mock API dependencies, for example, the database. A common option for .NET developers using a relational database like SQL Server is to use SQLite in the integration tests, however, even that solution suffers from other drawbacks, our friend Jimmy Bogard goes into more detail in his blog [Avoid In-Memory Databases for Tests](https://jimmybogard.com/avoid-in-memory-databases-for-tests/). What if instead of faking the database we actually used a real live database in our integration tests? There is a way, how? Well, with Docker.

### TestContainers
Docker is an amazing tool, it has facilitated the rapid growth of many modern-day apps due to its ability to quickly provision up an application along with any dependents in a reliable, repeatable way. In our case, we can use Docker and a library known as [testcontainers](https://github.com/testcontainers/testcontainers-dotnet) to create and run Docker containers within our integration tests. The [Testcontainers](https://dotnet.testcontainers.org/api/create_docker_image/) for .NET acts as a wrapper around the existing .NET Docker APIs providing a lightweight implementation to support tests environments. Essentially, when we run the integration tests defined in our custom WebApplicationFactory we can spin up an external dependency like a SQL Server instance that can then be used by the API. 

Here is how it would work. 

I am going to take another project of mine, the Chinook Web API that implements the JSON:API specification, which already has integration tests that use WebApplicationFactory, I am going to change the project to now include a reference to the library testcontainers-dotnet. I'll switch to the directory of the project, Chinook.Web.UnitTest and run the following command from the terminal.

```shell
dotnet add package Testcontainers --version 3.0.0
```

With testcontainers-dotnet now installed in my test project I am going to write up a Docker file that creates a one-node SQLServer instance. The file will live in the same directory as the Test Project. Here is the file content.

### MSSQL Dockerfile

```Docker
FROM mcr.microsoft.com/mssql/server:2019-latest

ENV ACCEPT_EULA=Y
ENV MSSQL_SA_PASSWORD=Test@12345
ENV MSSQL_PID=Express
ENV MSSQL_TCP_PORT=1433

EXPOSE 1433 # Default port of SQL Server

CMD ["/opt/mssql/bin/sqlservr"] #Start SQL Server
```

Note that the above Dockerfile only sets the password, the user will be the default SQL Server user SA. The next step is to bootstrap the Docker build and run processes into the integration tests, this is where I originally went down the wrong path. What I did was to add the code that builds and runs docker into my custom WebApplicationFactory since that class is responsible for bootstrapping my service, I ended up with this code.

### Wrong WebApplicationFactory

```C#
 public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(AddServices);
    }

    private void AddServices(IServiceCollection serviceCollection)
    {
        try
        {
            var projectDirectory = CommonDirectoryPath.GetProjectDirectory();
            var containerImage = new ImageFromDockerfileBuilder()
                .WithDockerfileDirectory(projectDirectory, string.Empty)
                .WithDockerfile("Dockerfile")
                .WithName("mssql-application-factory")
                .Build();

            containerImage
                .CreateAsync()
                .ConfigureAwait(false);

            const int mssqlBindPort = 1433;
            var sqlServerContainer = new ContainerBuilder()
                .WithImage("mssql-application-factory")
                .WithPortBinding(mssqlBindPort, mssqlBindPort)
                .Build();

            sqlServerContainer
                .StartAsync()
                .ConfigureAwait(false);

            var dbContextService = serviceCollection.SingleOrDefault(d => d.ServiceType == typeof(DbContextOptions<ChinookDbContext>));
            if (dbContextService != null)
            {
                // remove the DbContext that is registered on StartUp.cs
                serviceCollection.Remove(dbContextService);
            }

            // register the new DbContext, .NET Core dependency injection framework will now use the in-memory SQLite instance instead of whatever configuration was used to register the DbContext on the StartUp class.
            var sqliteInMemoryConnectionString = "Server=localhost;User Id=SA;Password=Test@12345;";
            serviceCollection.AddDbContext<ChinookDbContext>(contextOptions => contextOptions.UseSqlServer(sqliteInMemoryConnectionString));
            var builtServiceProvider = serviceCollection.BuildServiceProvider();
            using var scopedProvider = builtServiceProvider.CreateScope();
            var scopedServiceProvider = scopedProvider.ServiceProvider;
            // private field omitted for brevity
            
            var chinookDbContext = scopedServiceProvider.GetRequiredService<ChinookDbContext>();
            // these two lines are important, they ensure the in-memory database is created now.
            chinookDbContext.Database.OpenConnection();
            chinookDbContext.Database.EnsureCreated();
            // database is now ready to be seeded through the DbContext. The data will be available in each of your integration test due to the scope of the DbContext.
        }
        catch (Exception)
        {
            throw;
        }
    }
}
```

The code above is flawed and never worked, can you spot my mistake? The build interface exposed by the TestContainers library is all asynchronous while the AddServices is not, and while yes, I can change the interface of my AddServices from void to async Task and the project will build and run, but by changing the interface, the method will not be called by ConfigureServices, which makes that solution not viable. 

I needed another way to bootstrap my containers, and that's when I realized I'm using XUnit as my testing framework, I can use the [IAsyncLifetime](https://github.com/xunit/xunit/blob/master/src/xunit.core/IAsyncLifetime.cs) interface to bootstrap the Docker container, using IAsyncLifetime, you can define code that XUNit will execute as soon as your test class is created, I do want to make whatever I define reusable, therefore, instead of having my test class implement IAsyncLifetime I will create a SqlServerContainerImage class that implements the IAsyncLifetime, then whenever a test class needs a SQL Server it can inherit from SqlServerContainerImage.

### SqlServerContainerImage Class

```C#
 public class SqlServerContainerImage : IAsyncLifetime
{
    private readonly IFutureDockerImage _sqlServerContainerImage;
    private readonly IContainer _sqlServerContainer;

    public SqlServerContainerImage()
    {
        var projectDirectory = CommonDirectoryPath.GetProjectDirectory();
        _sqlServerContainerImage = new ImageFromDockerfileBuilder()
            .WithDockerfileDirectory(projectDirectory, string.Empty)
            .WithDockerfile("Dockerfile")
            .WithName("mssql-application-factory")
            .Build();
        const int mssqlBindPort = 1433;
        _sqlServerContainer = new ContainerBuilder()
            .WithImage("mssql-application-factory")
            .WithPortBinding(mssqlBindPort, mssqlBindPort)
            .WithWaitStrategy(Wait.ForUnixContainer().UntilPortIsAvailable(mssqlBindPort)) // Don't forget this line, it ensure the container is running before the test is executed.
            .Build();
    }
    /// <summary>
    /// Clean up process, delete images and containers
    /// </summary>
    /// <returns></returns>
    public async Task DisposeAsync()
    {
        await _sqlServerContainerImage
            .DeleteAsync()
            .ConfigureAwait(false);
        await _sqlServerContainer
            .DisposeAsync()
            .ConfigureAwait(false);
    }
    /// <summary>
    /// Bootstrap process, create images and containers.
    /// </summary>
    /// <returns></returns>
    public async Task InitializeAsync()
    {
        await _sqlServerContainerImage
            .CreateAsync()
            .ConfigureAwait(false);
        await _sqlServerContainer
            .StartAsync()
            .ConfigureAwait(false);
    }
}
```

Now I will need to modify my HomeControllerIntegrationTest originally created in [Integration Testing Using WebApplicationFactory](/post/2021/integration-testing-using-webapplicationfactory/). Here is the updated class.


```C#
public class HomeControllerIntegrationTest : SqlServerContainerImage,  IClassFixture<CustomWebApplicationFactory>
{
    private readonly CustomWebApplicationFactory _customWebApplicationFactory;

    public HomeControllerIntegrationTest(CustomWebApplicationFactory customWebApplicationFactory)
    {
        _customWebApplicationFactory = customWebApplicationFactory;
    }

    [Fact]
    public async Task GetHomeResource_HttpResponse_ShouldReturn200OK()
    {
        // Arrange
        using var httpClient = _customWebApplicationFactory.CreateClient();
        var requestUri = httpClient.BaseAddress.AbsoluteUri;
        // Act
        var sut = await httpClient.GetAsync(requestUri);
        // Assert 
        var responseCode = sut.StatusCode;
        responseCode.Should().Be(HttpStatusCode.OK);
    } 
}
```

As you can see the test class now inherits from SqlServerContainerImage, and here is the final class definition for my custom WebApplicationFactory.

### Correct SqlServerContainerImage

```C#
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(AddServices);
    }

    private void AddServices(IServiceCollection serviceCollection)
    {
        try
        {
            var dbContextService = serviceCollection.SingleOrDefault(d => d.ServiceType == typeof(DbContextOptions<ChinookDbContext>));
            if (dbContextService != null)
            {
                // remove the DbContext that is registered on StartUp.cs
                serviceCollection.Remove(dbContextService);
            }
            // register the new DbContext, .NET Core dependency injection framework will now use the in-memory SQLite instance instead of whatever configuration was used to register the DbContext on the StartUp class.
            var sqliteInMemoryConnectionString = "Server=localhost;User Id=SA;Password=Test@12345;";
            serviceCollection.AddDbContext<ChinookDbContext>(contextOptions => contextOptions.UseSqlServer(sqliteInMemoryConnectionString));
            var builtServiceProvider = serviceCollection.BuildServiceProvider();
            using var scopedProvider = builtServiceProvider.CreateScope();
            var scopedServiceProvider = scopedProvider.ServiceProvider;
            // private field omitted for brevity
            var chinookDbContext = scopedServiceProvider.GetRequiredService<ChinookDbContext>();
            // these two lines are important, they ensure the in-memory database is created now.
            chinookDbContext.Database.OpenConnection();
            chinookDbContext.Database.EnsureCreated();
            // database is now ready to be seeded through the DbContext. The data will be available in each of your integration test due to the scope of the DbContext.
        }
        catch (Exception)
        {
            throw;
        }
    }
}
```

Time to run a few tests to make sure everything is configured correctly, I'm going to add a breakpoint in the DisposeAsync method that is located in SqlServerContainerImage to prevent the container from being destroyed, I want to connect to the container to confirm that everything was configured correctly. I will know that everything is configured correctly if I see the database has all the tables defined by the DbContext. 

I'll run the test GetHomeResource_HttpResponse_ShouldReturn200OK() and hit the break now to prevent the container and image from being destroyed.

Now I need to connect to the test container, first I'll need to know the name, to do so, run the following command to get all running containers.

```shell
 docker ps -a
```

In my case, it outputs the following.

```
CONTAINER ID   IMAGE                              COMMAND                  CREATED         STATUS         PORTS                     NAMES
f0ba216b827b   mssql-application-factory:latest   "/opt/mssql/bin/perm…"   2 minutes ago   Up 2 minutes   0.0.0.0:1433->1433/tcp    objective_chandrasekhar
2fb31c1cbaac   testcontainers/ryuk:0.3.4          "/app"                   2 minutes ago   Up 2 minutes   0.0.0.0:32768->8080/tcp   testcontainers-ryuk-3427352d-4a88-4b53-9a17-7c0008fa04fe
```

The name of my test container is objective_chandrasekhar. Great, now run the following command.

```shell
docker exec -it objective_chandrasekhar "bash"
```

### Connect to SQL Sever

You should now be in the bash terminal of the container, run the following command in bash to get into sqlcmd.

```shell
/opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P "Test@12345"
```

Note that the user is SA because I never defined a user in my Dockerfile, only a password which you can see it is part of the command.

Here is what my shell looks like so far.

```Shell
$ docker exec -it objective_chandrasekhar "bash"
mssql@f0ba216b827b:/$ /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P "Test@12345"
1>
```

In the last line 1>, if you see that it means that you have successfully connected to the database and have sqlcmd running. Time to see if the databases were created, I can do that by running the following command.

```Shell
SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE='BASE TABLE'
```

You should now see a new line with 2>. Now run the following command.

```Shell
GO
```

The command executes the first command which should output all the base tables that currently exist on the database. 

```Shell
TABLE_CATALOG   TABLE_SCHEMA      TABLE_NAME        TABLE_TYPE
--------------- ----------------- ----------------- ----------
master          dbo               artists           BASE TABLE
master          dbo               employees         BASE TABLE
master          dbo               genres            BASE TABLE
master          dbo               media_types       BASE TABLE
master          dbo               playlists         BASE TABLE
master          dbo               customers         BASE TABLE
master          dbo               tracks            BASE TABLE
master          dbo               invoices          BASE TABLE
master          dbo               playlist_track    BASE TABLE
master          dbo               invoice_items     BASE TABLE
```

Great, I see all the entities defined in my [Chinook DbContext](https://github.com/circleupx/Chinook/tree/master/src/Chinook.Core/Models). That means that everything is configured correctly and it also means this approach to using Docker to create external dependencies will work, the test passed and the container and the image were successfully deleted from Docker.

### Conclusion

Awesome, using test containers is going to be a game changer for integrations tests I encourage you to learn more over at the [official site](https://dotnet.testcontainers.org/) and to check out their [best practices](https://dotnet.testcontainers.org/api/best_practices/) and examples their examples.

Until next time, cheerio!