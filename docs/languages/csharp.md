## C# (CSharp)

The `dotnet8-csharp` template is a C# template built with WebApplication and the minimal API style. It is multi-arch, and can be deployed to an amd64 or arm64 platform.

For a function based upon an existing project or Dockerfile, see: [Build ASP.NET Core APIs with Kubernetes and OpenFaaS](https://www.openfaas.com/blog/asp-net-core/)

### Create a new function

```bash
faas-cli template store pull dotnet8-csharp

export PREFIX=ttl.sh

faas-cli new \
    --lang dotnet8-csharp \
    my-function
```

### Add a dependency

To manage dependencies, edit the `Function.csproj` file and add an `<ItemGroup>` with `<PackageReference>`

```xml
<ItemGroup>
    <PackageReference Include="Npgsql" Version="8.0.2" />
</ItemGroup>
```

Or, run the `dotnet` command, i.e.

```bash
dotnet add my-function package Npgsql --version 8.0.2
```

## Add static files

You may want to copy static files into the final image such as a SQLite database, JSON data, HTML files or text templates for emails.

If a folder named static is found in the root of your function's source code, **it will be copied** into the final image published for your function.

To serve the contents of the static folder you can setup the file server in `Handler.cs`.

```c#
    public static void MapEndpoints(WebApplication app) {
        app.UseStaticFiles();
    }
```

### Swagger

To add Swagger to an endpoint, edit Handler.cs and add the following:

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;

namespace function;

public static class Handler
{
    public static void MapServices(IServiceCollection services)
    {
        services.AddEndpointsApiExplorer();
        services.AddSwaggerGen();

        // register additional services
    }

    public static void Map(WebApplication app)
    {
        app.UseSwagger();
        app.UseSwaggerUI();

        app.MapGet("/customers", () =>
        {
            return "A list of customers";
        })
        .WithOpenApi();
    }
}
```

Add the required package references to the Function.csproj file:

=== "dotnet CLI"

    ```bash
    dotnet add my-function \
        package Microsoft.AspNetCore.OpenApi --version 8.0.3
    dotnet add my-function \
        package Swashbuckle.AspNetCore --version 6.4.0
    ```

=== "xml"

    ```xml
    <ItemGroup>
        <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="8.0.3" />
        <PackageReference Include="Swashbuckle.AspNetCore" Version="6.4.0" />
    </ItemGroup>
    ```

You can then view the swagger UI by visiting `http://127.0.0.1:8080/function/my-function/swagger`, or the swagger JSON by clicking on the link displayed.

### Query a PostgreSQL database

Add a NuGet reference to the [Npgsql](https://www.nuget.org/packages/Npgsql) package to the `Function.csproj` file.

Either edit the `Function.csproj` file directly or use the dotnet cli.

=== "dotnet CLI"

    ```bash
    dotnet add my-function package Npgsql --version 8.0.2
    ```

=== "xml"

    ```xml
    <ItemGroup>
        <PackageReference Include="Npgsql" Version="8.0.2" />
    </ItemGroup>
    ```

Edit `Handler.cs` to query the postgreSQL database:

```c#
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Npgsql;

namespace function;

public static class Handler
{
    // MapEndpoints is used to register WebApplication
    // HTTP handlers for various paths and HTTP methods.
    public static void MapEndpoints(WebApplication app)
    {
        var connectionString = File.ReadAllText("/var/openfaas/secrets/pg-connection");
        var dataSource = NpgsqlDataSource.Create(connectionString);

        app.MapGet("/customers", async () =>
        {   
            var customers = new List<Customer>();
            await using (var cmd = dataSource.CreateCommand("SELECT id, name, email FROM customer"))
            await using (var reader = await cmd.ExecuteReaderAsync())
            {
                while (await reader.ReadAsync())
                {
                    customers.Add(new Employee{
                        Id = (int)reader["id"],
                        Name = reader.GetString(1),
                        Email = reader.GetString(2)
                    });
                }
            }
            return Results.Ok(customers);
        });
    }

    // MapServices can be used to configure additional
    // WebApplication services
    public static void MapServices(IServiceCollection services)
    {
    }
}

public class Customer {
    public int Id { get; set; }
    public string? Name { get; set; }
    public string? Email { get; set; }
}
```

The functions reads the PostgreSQL connection string from an [OpenFaaS secret](/reference/secrets/). Make sure the secret is created and [referenced](/reference/yaml/#function-secure-secrets) in the `stack.yml` file before deploying the function.

The connection string for Postgres should be formatted like this:

```
Host=postgresql;Username=postgres;Password=mysecretpassword;Database=postgres
```