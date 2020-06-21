---
date: "2016-12-04T00:00:00Z"
tags: ["asp.net", "asp.net-core"]
title: Configure Serilog With SQL Server Sink Inside ASP.NET Core
---

ASP.NET Core configuration has been re-architected and doesn't depend on Xml configurations any more, check [this excellent article](http://docs.asp.net/en/latest/fundamentals/configuration.html) for an introduction about the new configuration system.

In order to use Serilog with SQL Server Sink follow the below steps:
## Step1: Update the project.json
Update the project.json to reference the `Serilog` and `Serilog.Sinks.MSSqlServer` packages by adding the following lines at the end of the dependencies section.

```json
"Serilog": "1.5.14",
"Serilog.Sinks.MSSqlServer": "3.0.48"
```

## Step 2: Add Serilog SQL Server Sink settings into appsettings.json
Update `appsettings.json` file to include all the required Serilog SQL Server Sink configuration by adding the following JSON at the end of the appsettings.json file and before the last closing curly braces, make sure to update the connection string to your relevant values.

```json
    "Serilog": {
        "ConnectionString": "Server=(local);Database=serilogdemo;trusted_connection=true",
        "TableName": "Logs"
      }
```

## Step 3: Update the Startup class to configure Serilog.ILogger
ASP.NET Core has a new builtin Dependency Injection feature that can be used by  registering the services and their implementations through the `ConfigureServices` method inside `Startup` class so add the following section add the end of the `ConfigureServices` method. The [Dependency Injection](http://docs.asp.net/en/latest/fundamentals/dependency-injection.html) feature provide three types of registrations *Transient* , *Scoped* and *Singleton* and for this question I have used Singleton just for demo purpose.

```csharp
    services.AddSingleton<Serilog.ILogger>(x=>
    {
        return new LoggerConfiguration().WriteTo.MSSqlServer(Configuration["Serilog:ConnectionString"], Configuration["Serilog:TableName"],autoCreateSqlTable:true).CreateLogger();
    });
```
As you can see, the nested JSON configurations inside the appsettings.json could be read used the property `Configuration` and the configuration values can be retrieved using `:` as separator.

Note the above code is using `autoCreateSqlTable` parameter to automatically create the log table if doesn't exist; if you don't like this approach you can use sql script to create the table at any step you like.

##Step 4: Get reference to the Serilog.ILogger
You can get instance of the Serilog.ILogger via the Dependency Injection feature by simply adding variable in your constructor with object of the type Serilog.ILogger and then use the instance in any action method

```csharp
    private Serilog.ILogger _logger;
    public HomeController(Serilog.ILogger logger)
    {
       this._logger = logger;
    }
```

