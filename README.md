# Dependency Injection in .NET

## Table of Contents

1. [Introduction](#introduction)
2. [Dependency Inversion Principle (DIP)](#dependency-inversion-principle-dip)
3. [What is Dependency Injection (DI)?](#what-is-dependency-injection-di)
4. [Inversion of Control (IoC) Pattern](#inversion-of-control-ioc-pattern)
5. [Built-in .NET Dependency Injection](#built-in-net-dependency-injection)
6. [Object Lifetimes](#object-lifetimes)
7. [Managing IDisposable Instances](#managing-idisposable-instances)
8. [Deferred Configuration](#deferred-configuration)
9. [Thread-Safety in DI](#thread-safety-in-di)
10. [Asynchronous Dependency Resolution](#asynchronous-dependency-resolution)
11. [Service Locator Pattern (Anti-pattern)](#service-locator-pattern-anti-pattern)
12. [Best Practices](#best-practices)
13. [Advanced Examples](#advanced-examples)

---

## Introduction
Dependency Injection (DI) is a fundamental design pattern for building loosely coupled, testable, and maintainable applications. .NET has a built-in DI container since .NET Core, providing standardized ways to register and resolve dependencies.

---

## Dependency Inversion Principle (DIP)
High-level modules should not depend on low-level modules. Both should depend on abstractions.

Bad:
```csharp
public class BeerService {
    private readonly BeerRepository _beerRepository;
}
```

Good:
```csharp
public class BeerService : IBeerService {
    private readonly IBeerRepository _beerRepository;
    public BeerService(IBeerRepository beerRepository) => _beerRepository = beerRepository;
}
```

---

## What is Dependency Injection (DI)?
Dependency Injection is the practice of providing an object's dependencies from outside rather than letting the object construct them.

DI Methods:
- Constructor Injection (most common)
- Method Injection
- Property Injection

---

## Inversion of Control (IoC) Pattern
Instead of the application controlling the flow of instantiation, a framework controls it.

ASP.NET Core example:
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IBeerService, BeerService>();
var app = builder.Build();
app.MapGet("/", (IBeerService service) => service.Get());
app.Run();
```

---

## Built-in .NET Dependency Injection
Key Interfaces:
- `IServiceCollection` — registers services
- `IServiceProvider` — resolves services

Manual example:
```csharp
var services = new ServiceCollection();
services.AddTransient<IBeerService, BeerService>();
services.AddTransient<IBeerRepository, BeerRepository>();

var provider = services.BuildServiceProvider();
var service = provider.GetRequiredService<IBeerService>();
```

---

## Object Lifetimes
- **Transient**: Created each time requested.
- **Scoped**: Created once per request (scope).
- **Singleton**: Created once for the application's lifetime.

```csharp
services.AddTransient<IService, Service>();
services.AddScoped<IService, Service>();
services.AddSingleton<IService, Service>();
```

> ⚠️ Be cautious when mixing lifetimes. Avoid injecting scoped services into singletons.

---

## Managing IDisposable Instances
Use `IDisposable` carefully:

```csharp
class Disposable : IDisposable {
    public void Dispose() => Console.WriteLine("Disposed");
}
```

.NET DI container disposes services automatically when the provider or scope is disposed.

```csharp
using var scope = provider.CreateScope();
var svc = scope.ServiceProvider.GetRequiredService<IMyService>();
```

> Do not manually dispose injected dependencies.

---

## Deferred Configuration
Instead of resolving config on app start, use `IOptions<T>` for lazy loading.

```csharp
public class BeerOptions {
    public string ConnectionString { get; set; }
    public int Retries { get; set; }
}

services.Configure<BeerOptions>(config.GetSection("Beers"));
services.AddSingleton<IBeerService, BeerService>();
```

In service:
```csharp
public BeerService(IOptions<BeerOptions> options) { }
```

---

## Thread-Safety in DI
The .NET DI container is thread-safe. Services and factories don’t need manual locks or synchronization mechanisms when resolving dependencies.

---

## Asynchronous Dependency Resolution
Built-in DI does **not** support async constructors.

Wrong:
```csharp
services.AddSingleton<IService>(async sp => await InitAsync());
```

Right (Factory Pattern):
```csharp
services.AddSingleton<Func<Task<IService>>>(_ => async () => await InitAsync());
```

Usage:
```csharp
var factory = provider.GetRequiredService<Func<Task<IService>>>();
var service = await factory();
```

---

## Service Locator Pattern (Anti-pattern)
```csharp
var provider = services.BuildServiceProvider();
var service = provider.GetService<IMyService>();
```

❌ **Avoid** using `IServiceProvider` at runtime. Prefer constructor injection.

---

## Best Practices
- Always inject abstractions (interfaces), not concrete types.
- Understand and control service lifetimes.
- Use scopes to manage dependency disposal.
- Favor deferred configuration using `IOptions<T>`.
- Avoid building the service provider manually.
- Use factory patterns for async scenarios.
- Avoid the Service Locator pattern.

---

## Advanced Examples

### Scoped Services in Middleware
```csharp
public class CustomMiddleware {
    private readonly RequestDelegate _next;

    public CustomMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context, IScopedService service) {
        await service.DoWorkAsync();
        await _next(context);
    }
}
```

### Factory Pattern with Complex Dependencies
```csharp
services.AddSingleton<Func<IBeerService>>(provider => () => {
    var config = provider.GetRequiredService<IOptions<BeerOptions>>().Value;
    return new BeerService(config.ConnectionString, config.Retries);
});
```

### ActivatorUtilities for Complex Constructors
```csharp
var instance = ActivatorUtilities.CreateInstance<BeerService>(provider, "conn", 5);
```

### Conditional Service Registration
```csharp
if (env.IsDevelopment()) {
    services.AddScoped<IEmailSender, MockEmailSender>();
} else {
    services.AddScoped<IEmailSender, SmtpEmailSender>();
}
```

### Register Services via Extension Method
```csharp
public static class ServiceExtensions {
    public static IServiceCollection AddBeerModule(this IServiceCollection services, IConfiguration config) {
        services.Configure<BeerOptions>(config.GetSection("Beers"));
        services.AddScoped<IBeerService, BeerService>();
        return services;
    }
}
```

---

> Mastering .NET Dependency Injection means writing clean, testable, and scalable applications.

Happy coding! 💡

