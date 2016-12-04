---
layout: post
title: Configuring Redis for ASP.NET Core Session Store
tags: asp.net
---

> As you may have seen in [this ASP.NET Community Standup](https://www.youtube.com/watch?v=FSf83_TU5Yg&list=PL0M0zPgJ3HSftTAAHttA3JQU4vOjXFquF&index=0) and Scott Hanselman's blog post [ASP.NET 5 is dead - Introducing ASP.NET Core 1.0 and .NET Core 1.0](http://www.hanselman.com/blog/ASPNET5IsDeadIntroducingASPNETCore10AndNETCore10.aspx), ASP.NET 5 is being renamed to ASP.NET Core so I will use the new names in this post.

[Redis](http://redis.io) is an open source (BSD licensed), in-memory data structure store, used as database, cache and message broker. It supports data structures such as strings, hashes, lists, sets, sorted sets with range queries. Redis works with an in-memory dataset. it also supports persisting the dataset to disk. Moreover, It provides master-slave asynchronous replication. For more details, check the [documentation](http://redis.io/documentation).

This post will go through the steps required to configure [Redis](http://redis.io) for <s>ASP.NET 5</s> ASP.NET Core session store targeting DNX on .NET Framework 4.5.1 - 4.6. 

## Installing Redis
Redis is not officially supported on windows. However, the [Microsoft Open Tech group](https://msopentech.com/) develops and maintains Windows port targeting Win64 [available here]( https://github.com/MSOpenTech/redis)

To install Redis on your local machine, install the chocolatey package [http://chocolatey.org/packages/redis-64/](http://chocolatey.org/packages/redis-64/) and run `redis-server` from a command prompt.

Redis could also be installed as windows service, for more details please review [this link]( https://raw.githubusercontent.com/MSOpenTech/redis/2.8/Windows%20Service%20Documentation.md)

## Configuring Distributed Cache
ASP.NET shipped with several caching implementations including Redis, To use <s>ASP.NET 5</s> ASP.NET Core distributed caching, add the following packages to your `project.json`

At the time of this writing, the package `Microsoft.Extensions.Caching.Redis` version "1.0.0-rc1-final" doesn't support .NET Core 1.0 however it is planned to be supported and for more details check this [issue](https://github.com/aspnet/Caching/issues/121)

```json
"dependencies": {
    "Microsoft.Extensions.Caching.Redis": "1.0.0-rc1-final"
    ...
  },
```

Update the `ConfigureServices` method inside the `Startup` class to add the Redis caching services as below

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<IDistributedCache>(
                serviceProvider =>
                    new RedisCache(new RedisCacheOptions
                    {
                        Configuration = "localhost",
                        InstanceName = "Sample"
                    }));
    //Other services configuration...
}
```

## Configuring Session

 The <s>ASP.NET 5</s> ASP.NET Core `Session` is not configured by default, `Session` configurations must be done before using it, otherwise you will receive `InvalidOperationException` whenever you try to access it.

To use session, add `Microsoft.AspNet.Session` to the `project.json` as shown below

```json
"dependencies": {
    "Microsoft.AspNet.Session": "1.0.0-rc1-final"
    ...
  },
```

Then, Update the `ConfigureServices` method inside the `Startup` class to add the Redis caching services as below

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSession();
    //Other services configuration...
}
```

Finally, add the following line to `Configure`:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    app.UseSession();
}
```
That's it, Now you can reference `Session` from `HttpContext` and all session entries will be saved to Redis.