---
date: "2016-02-16T00:00:00Z"
tags: ["asp.net", "asp.net-core"]
title: ASP.NET Core MVC Feature Folders
---

This post will explain how to extend the ASP.NET Core MVC Razor view engine to support feature folders project structure.

## Project Structure
After creating a new ASP.NET Core project, the folder structure will be

```
Project
- Controllers
- Models
- Services
- ViewModels
- Views
```

The issue with this structure is when the project grow the number of classes inside each folder will keep growing and the situation becomes messy. That's why I prefer to use features folders structure where each feature is encapsulated inside a folder with all of its related files including Controllers, Views, Models, Services,...

```
App
- Features
-- Home
---- HomeController.cs
---- Index.cshtml
---- IndexViewModel.cs
---- Service.cs
```

Most of the files inside the feature folder will just work without issues considering that most of the classes will be constructed via Dependency Injection.

## Extend View Locations
The views doesn't work with the feature folder structure by default and In order to achieve that structure, The MVC view engine must be extended to search for the views inside the features folder. ASP.NET Core MVC provides a new interface `IViewLocationExpander` which is used by `RazorViewEngine` instances to determine search paths for a view. The following snippet shows how to implement `IViewLocationExpander` to extend `RazorViewEngine` to search for views inside the feature folder.

```csharp
public class FeaturesViewLocationExpander : IViewLocationExpander
{
    public void PopulateValues(ViewLocationExpanderContext context)
    {
        context.Values["customviewlocation"] = nameof(FeaturesViewLocationExpander);
    }

    public IEnumerable<string> ExpandViewLocations(
              ViewLocationExpanderContext context,
              IEnumerable<string> viewLocations)
    {
        var viewLocationFormats = new[]
        {
            "~/Features/{1}/{0}.cshtml",
            "~/Features/Shared/{0}.cshtml"
        };
        return viewLocationFormats;
    }
}
```

The `IViewLocationExpander` has two methods `PopulateValues` and `ExpandViewLocations` which are invoked in sequence.

**PopulateValues**

* `PopulateValues(ViewLocationExpanderContext)` is invoked each time a view is requested and the expected outcome is adding a value inside `ViewLocationExpanderContext` which could be consumed by `ExpandViewLocations`.
* The value added to `ViewLocationExpanderContext` determine a cache key that decides whether the `ExpandViewLocations` will be called or not.
* `ExpandViewLocations` will be called in the following cases
  * No result was found in the cache.
  * The values cached based on what was added to `ViewLocationExpanderContext` are different from the last time the `PopulateValues` was invoked.
  * The view was not found at the cached location.

**ExpandViewLocations**

* `ExpandViewLocations(ViewLocationExpanderContext, IEnumerable<string>)`  invoked to is invoked to determine all potential paths for a view.

# Configure RazorViewEngine
The last step is to update the `ConfigureServices` inside the `Startup` class to add the created expander to the `RazorViewEngineOptions`

```csharp
services.Configure<RazorViewEngineOptions>(options =>
{
    options.ViewLocationExpanders.Add(new CustomViewLocationExpander());
});
```

## Summary
Using the feature folders could be achieved in three steps:

1. Update project folder structure
2. Extend View Locations using IViewLocationExpander
3. Configure RazorViewEngine to use the IViewLocationExpander
