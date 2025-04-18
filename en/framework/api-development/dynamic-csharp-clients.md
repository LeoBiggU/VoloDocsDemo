# Dynamic C# API Client Proxies

ABP can dynamically create C# API client proxies to call your remote HTTP services (REST APIs). In this way, you don't need to deal with `HttpClient` and other low level details to call remote services and get results.

Dynamic C# proxies automatically handle the following stuff for you;

* Maps C# **method calls** to remote server **HTTP calls** by considering the HTTP method, route, query string parameters, request payload and other details.
* **Authenticates** the HTTP Client by adding access token to the HTTP header.
* **Serializes** to and deserialize from JSON.
* Handles HTTP API **versioning**.
* Add **correlation id**, current **tenant** id and the current **culture** to the request.
* Properly **handles the error messages** sent by the server and throws proper exceptions.

This system can be used by any type of .NET client to consume your HTTP APIs.

## Static vs Dynamic Client Proxies

ABP provides **two types** of client proxy generation system. This document explains the **dynamic client proxies**, which generates client-side proxies on runtime. You can also see the [Static C# API Client Proxies](./static-csharp-clients.md) documentation to learn how to generate proxies on development time.

Development-time (static) client proxy generation has a **performance advantage** since it doesn't need to obtain the HTTP API definition on runtime. However, you should **re-generate** the client proxy code whenever you change your API endpoint definition. On the other hand, dynamic client proxies are generated on runtime and provides an **easier development experience**.

## Service Interface

Your service/controller should implement an interface that is shared between the server and the client. So, first define a service interface in a shared library project, typically in the `Application.Contracts` project if you've created your solution using the startup templates.

Example:

````csharp
public interface IBookAppService : IApplicationService
{
    Task<List<BookDto>> GetListAsync();
}
````

> Your interface should implement the `IRemoteService` interface to be automatically discovered. Since the `IApplicationService` inherits the `IRemoteService` interface, the `IBookAppService` above satisfies this condition.

Implement this class in your service application. You can use [auto API controller system](./auto-controllers.md) to expose the service as a REST API endpoint.

## Client Proxy Generation

> The startup templates already comes pre-configured for the client proxy generation, in the `HttpApi.Client` project.

If you're not using a startup template, then execute the following command in the folder that contains the .csproj file of your client project:

````
abp add-package Volo.Abp.Http.Client
````

> If you haven't done it yet, you first need to install the [ABP CLI](../../cli/index.md). For other installation options, see [the package description page](https://abp.io/package-detail/Volo.Abp.Http.Client).

Now, it's ready to create the client proxies. Example:

````csharp
[DependsOn(
    typeof(AbpHttpClientModule), //used to create client proxies
    typeof(BookStoreApplicationContractsModule) //contains the application service interfaces
    )]
public class MyClientAppModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        //Create dynamic client proxies
        context.Services.AddHttpClientProxies(
            typeof(BookStoreApplicationContractsModule).Assembly
        );
    }
}
````

`AddHttpClientProxies` method gets an assembly, finds all service interfaces in the given assembly, creates and registers proxy classes.

### Endpoint Configuration

`RemoteServices` section in the `appsettings.json` file is used to get remote service address by default. The simplest configuration is shown below:

```json
{
  "RemoteServices": {
    "Default": {
      "BaseUrl": "http://localhost:53929/"
    } 
  } 
}
```

See the "AbpRemoteServiceOptions" section below for more detailed configuration.

## Usage

It's straightforward to use. Just inject the service interface in the client application code:

````csharp
public class MyService : ITransientDependency
{
    private readonly IBookAppService _bookService;

    public MyService(IBookAppService bookService)
    {
        _bookService = bookService;
    }

    public async Task DoIt()
    {
        var books = await _bookService.GetListAsync();
        foreach (var book in books)
        {
            Console.WriteLine($"[BOOK {book.Id}] Name={book.Name}");
        }
    }
}
````

This sample injects the `IBookAppService` service interface defined above. The dynamic client proxy implementation makes an HTTP call whenever a service method is called by the client.

### IHttpClientProxy Interface

While you can inject `IBookAppService` like above to use the client proxy, you could inject `IHttpClientProxy<IBookAppService>` for a more explicit usage. In this case you will use the `Service` property of the `IHttpClientProxy<T>` interface.

## Configuration

### AbpRemoteServiceOptions

`AbpRemoteServiceOptions` is automatically set from the `appsettings.json` by default. Alternatively, you can configure it in the `ConfigureServices` method of your [module](../architecture/modularity/basics.md) to set or override it. Example:

````csharp
public override void ConfigureServices(ServiceConfigurationContext context)
{
    context.Services.Configure<AbpRemoteServiceOptions>(options =>
    {
        options.RemoteServices.Default =
            new RemoteServiceConfiguration("http://localhost:53929/");
    });
    
    //...
}
````

### Multiple Remote Service Endpoints

The examples above have configured the "Default" remote service endpoint. You may have different endpoints for different services (as like in a microservice approach where each microservice has different endpoints). In this case, you can add other endpoints to your configuration file:

````json
{
  "RemoteServices": {
    "Default": {
      "BaseUrl": "http://localhost:53929/"
    },
    "BookStore": {
      "BaseUrl": "http://localhost:48392/"
    } 
  } 
}
````

`AddHttpClientProxies` method can get an additional parameter for the remote service name. Example:

````csharp
context.Services.AddHttpClientProxies(
    typeof(BookStoreApplicationContractsModule).Assembly,
    remoteServiceConfigurationName: "BookStore"
);
````

`remoteServiceConfigurationName` parameter matches the service endpoint configured via `AbpRemoteServiceOptions`. If the `BookStore` endpoint is not defined then it fallbacks to the `Default` endpoint.

#### Remote Service Configuration Provider

You may need to get the remote service configuration for a specific remote service in some cases. For this, you can use the `IRemoteServiceConfigurationProvider` interface.

**Example: Get the remote service configuration for the "BookStore" remote service**

````csharp
public class MyService : ITransientDependency
{
    private readonly IRemoteServiceConfigurationProvider _remoteServiceConfigurationProvider;

    public MyService(IRemoteServiceConfigurationProvider remoteServiceConfigurationProvider)
    {
        _remoteServiceConfigurationProvider = remoteServiceConfigurationProvider;
    }

    public async Task GetRemoteServiceConfiguration()
    {
        var configuration = await _remoteServiceConfigurationProvider.GetConfigurationOrDefaultAsync("BookStore");
        Console.WriteLine(configuration.BaseUrl);
    }
}
````

### As Default Services

When you create a service proxy for `IBookAppService`, you can directly inject the `IBookAppService` to use the proxy client (as shown in the usage section). You can pass `asDefaultServices: false` to the `AddHttpClientProxies` method to disable this feature.

````csharp
context.Services.AddHttpClientProxies(
    typeof(BookStoreApplicationContractsModule).Assembly,
    asDefaultServices: false
);
````

Using `asDefaultServices: false` may only be needed if your application has already an implementation of the service and you do not want to override/replace the other implementation by your client proxy.

> If you disable `asDefaultServices`, you can only use `IHttpClientProxy<T>` interface to use the client proxies. See the *IHttpClientProxy Interface* section above.

### Retry/Failure Logic & Polly Integration

If you want to add retry logic for the failing remote HTTP calls for the client proxies, you can configure the `AbpHttpClientBuilderOptions` in the `PreConfigureServices` method of your module class.

**Example: Use the [Polly](https://github.com/App-vNext/Polly) library to re-try 3 times on a failure**

````csharp
public override void PreConfigureServices(ServiceConfigurationContext context)
{
    PreConfigure<AbpHttpClientBuilderOptions>(options =>
    {
        options.ProxyClientBuildActions.Add((remoteServiceName, clientBuilder) =>
        {
            clientBuilder.AddTransientHttpErrorPolicy(policyBuilder =>
                policyBuilder.WaitAndRetryAsync(
                    3,
                    i => TimeSpan.FromSeconds(Math.Pow(2, i))
                )
            );
        });
    });
}
````

This example uses the [Microsoft.Extensions.Http.Polly](https://www.nuget.org/packages/Microsoft.Extensions.Http.Polly) package. You also need to import the `Polly` namespace (`using Polly;`) to be able to use the `WaitAndRetryAsync` method.



## See Also

* [Static C# Client Proxies](./static-csharp-clients.md)
