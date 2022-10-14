# How to Block IP Addresses in ASP.NET Core Web API+

In this article, we are going to learn how to block IP addresses in ASP.NET Core Web API. 

## Blocking vs Allowing IP Addresses

Blocking IP addresses is a useful feature if we want to ensure **known malicious users don’t gain access to our application**. This approach **allows access by default** and only defines the list of IP addresses to block from our application.

Similarly, we can decide to allow only a certain list of IP addresses to access our application, taking the approach of **blocking access to everyone** and only allowing **known, trusted entities to access our application.** 

Both methods are valid ways to restrict access to our application based on IP address, and logically in code, there is little difference in how we implement it. However, we are only going to focus on blocking IP addresses in this article.

Let’s look at some of the ways we can do that.

## Different Ways to Block IP Addresses in ASP.NET Core

ASP.NET Core provides us with 2 different ways to block IP addresses from accessing our applications.

Firstly, we can choose to **implement application-wide blocking for every request that our application receives, through the ASP.NET Core Middleware**.

> If you need to refresh your knowledge about middleware, check out our article on [ASP.NET Core Middleware](https://code-maze.com/working-with-asp-net-core-middleware/).

Our second option is to use [**action filters**](https://code-maze.com/action-filters-aspnetcore/), allowing us to be more selective about which controllers or even specific endpoints we wish to apply blocking to.

We will look at how we can implement IP blocking using both approaches, so let’s start by creating a project.

## Create an ASP.NET Core Project

Let’s start by creating an ASP.NET Core Web API project by using the Visual Studio template or running the `dotnet new webapi` command in a terminal window.

To demonstrate IP blocking at the endpoint level we need multiple endpoints. So let’s create a new controller with a couple of different endpoints.

## Create an IP Blocking Controller

Let’s add `IpBlockController` with 2 endpoints:

```csharp
[Route("api/[controller]")]
[ApiController]
public class IpBlockController : ControllerBase
{
    [HttpGet("unblocked")]
    public string Unblocked()
    {
        return "Unblocked access";
    }

    [HttpGet("blocked")]
    public string Blocked()
    {
        return "Blocked access";
    }
}
```

We will revisit these endpoints when it comes to testing our IP address blocking.

Next, let’s create our IP blocking service.

## Implement an IP Blocking Service

Let’s create a common service that can be used in both middleware and action filters:

```csharp
public interface IIpBlockingService
{
    bool IsBlocked(IPAddress ipAddress);
}
```

Next, we create an implementation of the interface:

```csharp
public class IpBlockingService: IIpBlockingService
{
    private readonly List<string> _blockedIps;

    public IpBlockingService(IConfiguration configuration)
    {
        var blockedIps = configuration.GetValue<string>("BlockedIPs");
        _blockedIps = blockedIps.Split(',').ToList();
    }

    public bool IsBlocked(IPAddress ipAddress) => _blockedIps.Contains(ipAddress.ToString());
}
```

First, we inject the `IConfiguration` interface, and use it to get the BlockedIPs value from `appsettings.json`, which is a comma-separated list of IP addresses.

Finally, we call the `IsBlocked()` method to check if `our _blockedIps` list contains the `ipAddress` parameter.

As we are using a value from `appsettings.json`, let’s add that next:

```csharp
"BlockedIPs": ""
```

We’ll leave the value blank for now as we will receive an IP address in the testing section that can be used to block.

The final thing to do is register our service as a transient service in the `Program` class:

```csharp
builder.Services.AddTransient<IIpBlockingService, IpBlockingService>();
```

With the service created, let’s look at blocking access to our application by using the ASP.NET Core Middleware.

## Write an IP Blocking Middleware 

To start, we’re going to create our middleware class:

```csharp
public class IpBlockMiddelware
{
    private readonly RequestDelegate _next;
    private readonly IIpBlockingService _blockingService;

    public IpBlockMiddelware(RequestDelegate next, IIpBlockingService blockingService)
    {
        _next = next;
        _blockingService = blockingService;
    }

    public async Task Invoke(HttpContext context)
    {
        var remoteIp = context.Connection.RemoteIpAddress;

        var isBlocked = _blockingService.IsBlocked(remoteIp!);

        if (isBlocked)
        {
            context.Response.StatusCode = (int)HttpStatusCode.Forbidden;
            return;
        }

        await _next.Invoke(context);
    }
}
```







