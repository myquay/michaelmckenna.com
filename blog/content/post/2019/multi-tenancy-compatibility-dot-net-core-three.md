+++
date = "2019-10-02T13:11:12+12:00"
description = "Upgrading multi-tenancy to .NET Core 3.0"
title = "Migrating multi-tenancy to .NET Core 3.0"
subtitle = "How to make the multi-tenant solution compatible with .NET Core 3.0"
url = "/multi-tenancy-compatibility-dot-net-core-three"
tags = ["guide", "azure", "dot net core", "multitenant"]
summary = ".NET Core 3.0 is out and our 4 part multi-tenancy series is already out of date! Here we'll cover off the breaking changes and the updates we need to make."
+++

## Introduction

ASP.NET Core 3 has removed the ability to return a service provider from the `ConfigureServices` method, if you upgrade a project to ASP.NET Core 3 then our solution will give you a nice not supported exception on start up

> System.NotSupportedException: 'ConfigureServices returning an System.IServiceProvider isn't supported.'

Which is due to how we [configure tenant specific containers](/multi-tenant-asp-dot-net-core-application-tenant-containers) - we return a custom service provider.

```csharp

 return services.UseMultiTenantServiceProvider<Tenant>((t, c) =>
    {
        c.RegisterInstance(new TenantSpecificInstance()).SingleInstance();
    });

```

### Why do we use a custom service provider?

We needed to support a custom lifetime management policy for a tenant specific singleton. Since the default ASP.NET service provider does not support custom lifetime management we switched to one that does, we chose [Autofac](https://autofac.org/). Custom lifetime management policies are still unsupported in ASP.NET Core 3 so we still need to register our custom service provider and will need to use a different integration point. 

_There's a good reason the default service container doesn't support it - the vast majority of apps do not need it!_ 😉👌

### The new integration point

In ASP.NET Core 3+ (and [Generic hosts](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-3.0)) there is a new method called `UseServiceProviderFactory` which you use to register your custom service provider factory during host configuration.

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                 ...
                .UseServiceProviderFactory(...)
```

Going back to our `UseMultiTenantServiceProvider` extension, it preformed two tasks

1. Register tenant specifc services
2. Return the new service provider

So our factory will need to also need to perform those tasks. We will call our service factory `MultiTenantServiceProviderFactory` and it will accept one argument, the callback to configure all of the multitenant services _(we don't want to configure them in the Host Configuration!)_.

Once we're done the new way to enable tenant specific containers will be like this

```csharp

public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                ...
                .UseServiceProviderFactory(
                    new MultiTenantServiceProviderFactory(Startup.ConfigureMultiTenatServices))
```

## The implementation

Our implementation has two parts, our service provider factory to inject our multi-tenant container and the configuration callback which registers the tenant specific serivices.

### MultiTenantServiceProviderFactory

The factory will do exactly what our extension method used to do, construct a new `MultiTenantContainer` and specifty the tenant specific services configuration

```charp

public class MultiTenantServiceProviderFactory<T> : IServiceProviderFactory<ContainerBuilder> where T : Tenant
    {

        public Action<T, ContainerBuilder> _tenantSerivcesConfiguration;

        public MultiTenantServiceProviderFactory(Action<T, ContainerBuilder> tenantSerivcesConfiguration)
        {
            _tenantSerivcesConfiguration = tenantSerivcesConfiguration;
        }

        /// <summary>
        /// Create a builder populated with global services
        /// </summary>
        /// <param name="services"></param>
        /// <returns></returns>
        public ContainerBuilder CreateBuilder(IServiceCollection services)
        {
            var builder = new ContainerBuilder();

            builder.Populate(services);

            return builder;
        }

        /// <summary>
        /// Create our serivce provider
        /// </summary>
        /// <param name="containerBuilder"></param>
        /// <returns></returns>
        public IServiceProvider CreateServiceProvider(ContainerBuilder containerBuilder)
        {
            MultiTenantContainer<T> container = null;
            
            Func<MultiTenantContainer<T>> containerAccessor = () =>
            {
                return container;
            };

            containerBuilder
                .RegisterInstance(containerAccessor)
                .SingleInstance();

            container = new MultiTenantContainer<T>(containerBuilder.Build(), _tenantSerivcesConfiguration);

            return new AutofacServiceProvider(containerAccessor());
        }
    }

```


### Startup.ConfigureMultiTenatServices

We still need to provide our tenant specifc service registrations, we will do this via. a callback as we don't want to set these up in the host configuration file. For our callback we've followed the framework's `ConfigureServices` idiom and placed the tenanted version in the Startup file with a very similar name `ConfigureMultiTenantServices`. This provides a nice intuative developer experience, global services go in `ConfigureServices`, and tenant specifc services of in `ConfigureMultiTenantServices`.


```csharp

public static void ConfigureMultiTenantServices(Tenant t, ContainerBuilder c)
{
    c.Register...
}

```

Since we no longer register or service provider in the `ConfigureServices` method, we can remove our old extension method `UseMultiTenantServiceProvider` and delete it from the `ConfigureSerivces` method.

### Tenant specific options

I couldn't track down why, but it seems the framework now seems to agressively resolves any `IOptions`, sometimes even before the `HttpContext` is available. Depending on our tenant resolution strategy we potentially need it which was causing issues with our [Tenant Specific Options](multi-tenant-asp-dot-net-core-application-tenant-specific-configuration-options) implementation. To resolve this we just shunted our registration down to the tenant container where we know our tenants are resolving correctly.

To do this we will create a new Extension method to register our `IOptions` implementation against the container builder

```csharp

 /// <summary>
/// Register tenant specific options
/// </summary>
/// <typeparam name="TOptions">Type of options we are apply configuration to</typeparam>
/// <param name="tenantOptionsConfiguration">Action to configure options for a tenant</param>
/// <returns></returns>
public static ContainerBuilder RegisterTenantOptions<TOptions, T>(this ContainerBuilder builder, Action<TOptions, T> tenantConfig) where TOptions : class, new() where T : Tenant
{
    builder.RegisterType<TenantOptionsCache<TOptions, T>>()
        .As<IOptionsMonitorCache<TOptions>>()
        .SingleInstance();

    builder.RegisterType<TenantOptionsFactory<TOptions, T>>()
        .As<IOptionsFactory<TOptions>>()
        .WithParameter(new TypedParameter(typeof(Action<TOptions, T>), tenantConfig))
        .SingleInstance();


    builder.RegisterType<TenantOptions<TOptions>>()
        .As<IOptionsSnapshot<TOptions>>()
        .SingleInstance();

    builder.RegisterType<TenantOptions<TOptions>>()
        .As<IOptions<TOptions>>()
        .SingleInstance();

    return builder;
}

```

And remove our old `WithPerTenantOptions` extension. Now the options can be registered in our Per-Tenant configuration section instead of the global services section.

```csharp
c.RegisterTenantOptions<CookiePolicyOptions, Tenant>((options, tenant) =>
             {
                 options.ConsentCookie.Name = tenant.Id + "-consent";
                 options.CheckConsentNeeded = context => false;
             })
```

## Wrapping up

I'm very excited about all the new features in .NET Core 3, especially C# 8.0 and the introduction of [Nullable Reference Types](https://docs.microsoft.com/en-us/dotnet/csharp/nullable-references), I'm glad to see the [migration path from .NET Core 2.2 to 3.0](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30?view=aspnetcore-3.0&tabs=visual-studio) for quite a complicated bit of middleware was fairly straight forward in this case.