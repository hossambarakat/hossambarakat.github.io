---
date: "2016-05-08T00:00:00Z"
tags: ["asp.net", "asp.net-core"]
title: Configure Camel Case Resolver for ASP.NET Core MVC
---

It is known that camelCase is the common convention while working with JavaScript so you will find yourself in an awkward situation when you integrate with WebApi because the default JSON output is formatted using PascalCase.

Assuming that we have an Employee model that is returned from Web API

```csharp
public class Employee
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
[HttpGet]
public Employee Get()
{
    return new Employee()
    {
        FirstName = "Hossam",
        LastName = "Barakat"
    };
}
```

The following will be the JSON outcome

```json
{"FirstName":"Hossam","LastName":"Barakat"}
```

The output could be changed to use camelCase by using a new contract resolver and thanks to the excellent library [Json.NET](http://www.newtonsoft.com/json) there is a builtin contract resolver that we can plug directly into the ASP.NET startup configurations.

Update the dependencies section inside the project.json to reference `Newtonsoft.Json`

```json
"Newtonsoft.Json": "8.0.2"
```

Update the `ConfigureServices` method inside the `Startup` class.

```csharp
services.AddMvc()
        .AddJsonOptions(options => options.SerializerSettings.ContractResolver = new CamelCasePropertyNamesContractResolver());
```

The output will be camel case formatted

```json
{"firstName":"Hossam","lastName":"Barakat"}
```
