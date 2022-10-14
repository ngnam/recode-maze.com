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

First, we inject `RequestDelegate`, which is a function that can process an HTTP request. It represents the next delegate in the pipeline that will handle the HTTP request.

Also, we inject the `IIpBlockingService` interface.

Next, `Invoke()` takes the current HttpContext as a parameter, which allows us to retrieve the current request’s IP address through `context.Connection.RemoteIpAddress`.

Then, we pass this IP address to the `IsBlocked()` method to verify if the IP should be blocked, and if it should, we set `context.Response.StatusCode` to Forbidden and return from the method.

Finally, if the IP address is not blocked, we call Invoke() on RequestDelegate to ensure the request continues through the rest of the middleware pipeline.

With our middleware defined, we add it to the pipeline in the `Program` class:

```csharp
var app = builder.Build();

app.UseMiddleware<IpBlockMiddelware>();

// other middleware...
```

Now we are ready to test the IP blocking functionality.

## Test Middleware With ngrok

To demonstrate the IP address blocking, we can use the reverse proxy tool ngrok. This allows us to tunnel our locally running application to the public-facing internet. Ngrok will provide us with a URL that we can make requests to, which will, in turn, be forwarded to localhost. It also provides a public IP address that we can add to our blocklist to verify our IP blocking is working as expected.

With ngrok installed, we create a tunnel to our locally running application: 

```ts
ngrok http https://localhost:7116
```

To ensure we receive the correct IP address in our middleware, we forward the `XForwardedFor` and `XForwardedProto` headers in `Program`:

```csharp
var app = builder.Build(); 

app.UseForwardedHeaders(new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
});

app.UseMiddleware<IpBlockMiddelware>(); 

// other middleware...
```

We add `app.UseForwardedHeaders()` to our middleware pipeline, ensuring to call it before the `IpBlockMiddleware`.

Now, we can run our application and navigate to `http://127.0.0.1:4040`, which is a web interface provided by ngrok. Here we will see a randomly generated URL, which looks similar to `https://<random-string>.eu.ngrok.io`.

We can make a request to `https://<random-string>.eu.ngrok.io/api/ipblock/unblocked` and will receive Unblocked access as our response, showing that we can successfully make requests to our application.

In the ngrok web interface, we find an IP address:

![image](https://user-images.githubusercontent.com/11207864/195809605-b89f9e0b-bd1e-4fad-a2fb-42a4b07ef97d.png)

Let’s add this to our `BlockedIPs `setting (if not using ngrok, you would add `::1` here for `localhost`):

```
"BlockedIPs": "84.13.150.143"
```

When we run our application again and make the same request, we will receive a 403 Forbidden response, which will be the same for any endpoint we make a request to.

Now that we’ve seen how to block all requests to our application, let’s look at how to do it selectively with action filters.

## Create an Action Filter For Blocking IP Addresses

We start by creating `IpBlockActionFilter`:

```csharp
public class IpBlockActionFilter : ActionFilterAttribute
{
    private readonly IIpBlockingService _blockingService;

    public IpBlockActionFilter(IIpBlockingService blockingService)
    {
        _blockingService = blockingService;
    }

    public override void OnActionExecuting(ActionExecutingContext context)
    {
        var remoteIp = context.HttpContext.Connection.RemoteIpAddress;

        var isBlocked = _blockingService.IsBlocked(remoteIp!); 

        if (isBlocked)
        {
            context.Result = new StatusCodeResult(StatusCodes.Status403Forbidden);
            return;
        }

        base.OnActionExecuting(context);
    }
}
```

First, we inherit from `ActionFilterAttribute`. Once again, we need our `IIpBlockingService` so we inject it through the constructor.

Next, we override `OnActionExecuting()`, and like our middleware, check if the `RemoteIpAddress` is blocked, setting `context.Result` to `Forbidden` if so.

Finally, we call `base.OnActionExecuting()` to ensure the base class method is called.

Also, we need to register the action filter as a scoped service in the `Program` class:

```
builder.Services.AddScoped<IpBlockActionFilter>();
```

## Test the Action Filter

With our custom action filter created, we’ll revisit `IpBlockController` and add it to the blocked endpoint:

```csharp
[ServiceFilter(typeof(IpBlockActionFilter))]
[HttpGet("blocked")]
public string Blocked()
{
    return "Blocked access";
}

Also, we still have our middleware enabled, so we can simply comment out the middleware registration in the `Program` class:

```csharp
var app = builder.Build();

app.UseForwardedHeaders(new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
});

//app.UseMiddleware<IpBlockMiddelware>();
```

Running our application again and navigating to api/ipblock/unblocked, we receive Unblocked access as we expect.

However, if we attempt to make a request to api/ipblock/blocked we receive a 403 Forbidden, due to our action filter.

# Conclusion & Thanks

In this article, we looked at different ways to block IP addresses from making requests to our applications, which can be very useful when we want to ensure we only allow a certain list of users to access our application or block known malicious threats.
