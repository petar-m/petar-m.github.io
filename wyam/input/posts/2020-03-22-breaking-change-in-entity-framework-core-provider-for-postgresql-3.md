Title: Breaking change in Entity Framework Core provider for PostgreSQL 3.0
Published: 22/3/2020
Tags: [PostgreSQL,.NET Core,EF]
---
<!-- # Breaking change in Entity Framework Core provider for PostgreSQL 3.0 -->

Beware when upgrading form `Npgsql.EntityFrameworkCore.PostgreSQL 2.2` and running `PostgreSQL 9.6` or earlier.

## Background  

Recently I wanted to add [ASP.NET Core Identity](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-3.1&tabs=visual-studio) to an application using PostgreSQL as a data storage. Since I prefer to manage database schema and migrations separately from the application (currently using [DbUp](https://dbup.readthedocs.io/en/latest/)) I decided to use Entity Framework Core migrations to generate the SQL script for it.

## Generating the migration SQL script  

Using the Entity Framework Core tools 3.x is straight-froward. 
(Given that you have added the necessary dependencies and defined your application's `IdentityDbContext`)  

First make sure it is installed:  

```powershell
dotnet tool install --global dotnet-ef
```

Initialize the migrations in the project where your DbContext is:

```powershell
dotnet ef migrations add init
```

Then generate the SQL script for the DbContext (called ApplicationDbContext in my case):

```powershell
dotnet ef migrations script -c ApplicationDbContext -o migration_script.sql
```

Removed the SQL related to `__EFMigrationsHistory` and my migration script was ready.
Then removed the migrations folder from my project and was ready to go.  

Ran the SQL script and... it failed!

## The issue  

I was quite surprized to see that the error was in the SQL:  

```
ERROR: syntax error at or near "GENERATED"
```

And the offending statement was:

```sql
CREATE TABLE "AspNetRoleClaims" (
    "Id" integer NOT NULL GENERATED BY DEFAULT AS IDENTITY,
...    
```

which on further research made perfect sense because this is a new syntax  `GENERATED AS IDENTITY` for creating auto-incremented column available in PostgreSQL 10 and I was running version 9.6.  

I was using `Npgsql.EntityFrameworkCore.PostgreSQL 3.1.1` and going through it I found a documented breaking change in `3.0.0`:

> *The default value generation strategy has changed from the older SERIAL columns to the newer IDENTITY columns, introduced in PostgreSQL 10.*

You can look for details here: [3.0 Release Notes](https://www.npgsql.org/efcore/release-notes/3.0.html)  

Fortunately the fix was easy - just had to specify the PostgreSQL version I was targeting:

```csharp
public class ApplicationDbContext : IdentityDbContext
{
    public ApplicationDbContext(DbContextOptions options) : base(options)
    {
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseNpgsql("DefaultConnection", o => o.SetPostgresVersion(9, 6));
    }
}
```

After re-generating the migrations the older `SERIAL` statement was used:

```sql
CREATE TABLE "AspNetRoleClaims" (
    "Id" serial NOT NULL,
...    
```

and everything went fine.  

## Conclusion  

PostgreSQL 10 introduced `GENERATED AS IDENTITY` syntax aiming to replace `SERIAL` for automatic assignment of unique value to a column. `Npgsql.EntityFrameworkCore.PostgreSQL 3.0.0` takes advantage of it and uses it as a default when generating SQL migrations. Upgrading dependencies caught me off guard this time but the breaking change was well documented and my problem easily fixed.