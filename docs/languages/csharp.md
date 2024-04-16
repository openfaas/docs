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

To manage dependencies, edit the Function.csproj file.

### Swagger

To add Swagger to an endpoint, edit Function.cs and add the following:

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

Add new references to the Function.csproj file:

```xml
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="8.0.3" />
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.4.0" />
  </ItemGroup>
```

You can then view the swagger UI by visiting `http://127.0.0.1:8080/function/my-function/swagger`, or the swagger JSON by clicking on the link displayed.
