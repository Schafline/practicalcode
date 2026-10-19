---
title: "Migrating ASP.NET Core from SQLite to PostgreSQL"
description: "A step-by-step guide to migrating an ASP.NET Core application from SQLite to PostgreSQL using Entity Framework Core."
pubDate: "Oct 17 2026"
heroImage: "../../assets/sqlite-to-postgresql.png"
---

In an earlier post, I explained how to deploy an ASP.NET Core Razor Pages application using PostgreSQL. For many projects, this is likely to be the preferred approach from the outset. However, there are also plenty of situations where a lighter weight database such as SQLite is a better fit during the early stages of development.

SQLite requires almost no setup, has no separate database server to manage and works exceptionally well for prototypes, learning projects and smaller applications. As a result, many developers choose SQLite during development before moving to a more feature-rich database platform later in the application's lifecycle.

At some point, however, project requirements may begin to outgrow SQLite. PostgreSQL offers stronger concurrency support, advanced database features, better backup and recovery options and a wide range of managed hosting solutions. These characteristics make it a popular choice for production workloads.

In this article, we'll walk through the process of migrating an existing ASP.NET Core application from SQLite to PostgreSQL, including updating Entity Framework Core configuration, recreating migrations and preparing the application for deployment.

---

## 1 Installing the PostgreSQL Provider

Before updating the application code, we need to install the Entity Framework Core provider for PostgreSQL. This package contains the Npgsql database provider and enables EF Core to generate PostgreSQL specific migrations.

From your project folder, run:

``` bash
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

Once the package is installed, EF Core can target PostgreSQL when generating migrations and the application can be configured to use the `UseNpgsql()` provider at runtime.

---

## 2 Creating a Design-Time DbContext Factory

Before we can migrate to PostgreSQL, Entity Framework Core needs a reliable way to create a `DbContext` during design-time operations such as:

``` bash
dotnet ef migrations add
dotnet ef database update
```

When these commands run, the application itself is not executing, so EF Core cannot rely on the normal runtime dependency injection configuration found in `Program.cs`.

To solve this problem, we create a class that implements `IDesignTimeDbContextFactory<TContext>`. This factory provides EF Core with a predictable way to construct a `DbContext` whenever it needs to generate or apply migrations.

If your project doesn't already contain a design-time factory, create one for your application's `DbContext`. A common naming convention is to append `Factory` to the name of the context class. 
Throughout this article, I'll continue to use `ShopContext` as the name of the Entity Framework Core `DbContext`. `ShopContext` is the `DbContext` used by the sample application from my eBook, Building Your First Shopping Basket, on which this extract is based.

Create a new file and add the following code, replacing `ShopContext` with the name of your own `DbContext` class throughout, including the file name:

**ShopContextFactory.cs**

``` csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;

// Replace with your own namespace
namespace Storefront.Data;

// Rename this class and replace ShopContext with your own DbContext throughout this file
public class ShopContextFactory
: IDesignTimeDbContextFactory<ShopContext>
{
  public ShopContext
  CreateDbContext(string[] args)
  {
    var config = new ConfigurationBuilder()
      .AddJsonFile("appsettings.json")
      .AddJsonFile("appsettings.Development.json",
       optional: true)
      .AddEnvironmentVariables()
      .Build();

    var env =
      Environment.GetEnvironmentVariable(
      "ASPNETCORE_ENVIRONMENT");
    // Replace ShopContext with your own DbContext
    var optionsBuilder =
      new DbContextOptionsBuilder<ShopContext>();

    if (env == "Production")
    {// Use your production PostgreSQL connection string
      var connectionString =
        Environment.GetEnvironmentVariable(
        "POSTGRES_CONNECTION");

      optionsBuilder.UseNpgsql(connectionString);
    }
    else
    {
      // Replace ShopDbConnection with your own connection string name
      var sqlite =
        config.GetConnectionString(
       "ShopDbConnection");

      optionsBuilder.UseSqlite(sqlite);
    }
    return new ShopContext
    (optionsBuilder.Options);
  }
}
```

The example above assumes the production connection string is supplied via an environment variable named `POSTGRES_CONNECTION`. This is used purely as an example and can be replaced with any configuration source that suits your application. If your PostgreSQL connection details are stored in `appsettings.json`, user secrets or another environment variable, update the factory accordingly.

The code also ensures that EF Core generates PostgreSQL compatible migrations
when you temporarily switch your local environment to Production.
Without this change, EF Core would continue producing SQLite‑specific
SQL, which is incompatible with PostgreSQL.

---

## 3 Regenerating the Migrations for PostgreSQL

As we're switching database providers, we'll remove the existing SQLite migrations and generate a new set of migrations targeting PostgreSQL.

Follow these steps to produce a clean PostgreSQL migration:

**Delete the existing *Migrations* folder**

These files contain SQLite‑specific SQL and cannot be reused

**Set the Environment to Production**

To ensure that your design-time factory selects PostgreSQL rather than SQLite, temporarily switch the environment to Production.

**Windows (PowerShell)**

``` bash
    $env:ASPNETCORE_ENVIRONMENT="Production"
```

**Linux/macOS**

``` bash
    export ASPNETCORE_ENVIRONMENT="Production"
```

**Provide a Temporary PostgreSQL Connection String**

The design-time factory expects a PostgreSQL connection string when running in Production mode.

**Windows (PowerShell)**

``` bash
    $env:POSTGRES_CONNECTION="Host=localhost;Port=5432;Database=fake;Username=fake;Password=fake"
```

**Linux/macOS**

``` bash
    export POSTGRES_CONNECTION="Host=localhost;Port=5432;Database=fake;Username=fake;Password=fake"
```

The value does not need to point to a real database. Entity Framework Core simply requires a valid connection string so that it can construct the `DbContext` while generating the migration.

**Generate the Migration**

Run the following command from your project folder (InitialCreate is just a suggested name here):

``` bash
    dotnet ef migrations add InitialCreate
```

EF Core will now generate PostgreSQL-compatible migration code instead of SQLite-specific migration code. Any future migrations should now be created against PostgreSQL rather than SQLite.

>⚠️ Keep the terminal window open until the migration has been generated. The environment variables created above are temporary and only exist for the current terminal session.

---

## 4 Update Program.cs to Select the Correct Provider

The design-time factory only affects Entity Framework Core tooling. The running application still needs to know which database provider to use when it starts.

To achieve this, update the `DbContext` registration in `Program.cs` so that SQLite is used during development and PostgreSQL is used in production.


``` csharp
var environment = builder.Environment;
if (environment.IsDevelopment())
{
  builder.Services.AddDbContext<ShopContext>(
    options =>
    { // Replace ShopDbConnection with your own connection string name
      options.UseSqlite(
        builder.Configuration
          .GetConnectionString(
            "ShopDbConnection"));
    });
}
else
{
  var connectionString =
    Environment.GetEnvironmentVariable("POSTGRES_CONNECTION");
  // Replace ShopContext with your own DbContext
  builder.Services.AddDbContext<ShopContext>(
    options =>
      options.UseNpgsql(
        connectionString,
        npgsqlOptions =>
        {
          npgsqlOptions
            .EnableRetryOnFailure();
        }));
}
```

In this example, the PostgreSQL connection string is supplied through a POSTGRES_CONNECTION environment variable containing a standard Npgsql connection string.

Some hosting providers, including Railway, expose PostgreSQL connection details in URI format (for example, postgres://user:password@host:port/database) rather than as key-value pairs. In these situations, you'll need to convert the URI into a standard Npgsql connection string before calling `UseNpgsql()`. I cover this process in my article [Deploying ASP.NET Core Razor Pages Applications to Railway](https://www.practicalcode.dev/blog/deploying-aspnet-core-to-railway/).

## 5 Next Steps

At this point, your application is configured to use SQLite during development and PostgreSQL in a Production environment. Entity Framework Core has been updated to use the PostgreSQL provider and a new PostgreSQL-compatible migration has been generated.

The application is now ready to be deployed to any hosting environment that supports PostgreSQL.

If you're planning to apply migrations automatically during application startup, you'll also need to add the appropriate migration code to Program.cs. I cover this approach in my article [Deploying ASP.NET Core Razor Pages Applications to Railway](https://www.practicalcode.dev/blog/deploying-aspnet-core-to-railway/), where pending migrations are applied automatically whenever the application starts in a Production environment.

One final point is worth noting. The PostgreSQL migration generated earlier was created while the application was running in a temporary Production configuration. Once you close the terminal window, those temporary environment variables will be lost and your local development environment will automatically revert to using SQLite.

If you subsequently make changes to your data model while developing against SQLite, you'll need to remove the PostgreSQL migrations and generate a new set of SQLite migrations for local development.

When you're ready to deploy those changes to PostgreSQL, repeat the migration process described in this article: delete the SQLite migrations, switch temporarily to the PostgreSQL configuration and generate a new PostgreSQL migration before deployment.

While this introduces a little extra work, it allows you to retain the simplicity of SQLite during development while taking advantage of PostgreSQL in production.
