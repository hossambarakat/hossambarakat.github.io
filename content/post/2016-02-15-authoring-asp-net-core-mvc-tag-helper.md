---
date: "2016-02-15T00:00:00Z"
tags: ["asp.net", "asp.net-core"]
title: Authoring ASP.NET Core MVC Tag Helper
---

Tag helpers is one of the new additions to ASP.NET Core MVC 1.0 features which enables the tags to participate in the server side rendering with an easy syntax that is easy to write and read specially for front-end developers who don't have any previous knowledge with Razor syntax.

In this post, I will show how to develop custom tag helper to render one of the Bootstrap components which is [List Group](http://getbootstrap.com/components/#list-group). The target is to build tag helper that could generate that following Html based on `List<string>`

```html
<ul class="list-group">
  <li class="list-group-item">Cras justo odio</li>
  <li class="list-group-item">Dapibus ac facilisis in</li>
  <li class="list-group-item">Morbi leo risus</li>
</ul>
```

## The Target Tag
The first step is to decide what is the HTML tag that will participate into the server side processing. The tag could be one of the HTML built in tags such as `<ul>` or a totally new HTML tag such as the one that we will use in this post which will be `<list-group>`.

## The Tag Helper Class
In order to develop tag helper you have to create a class that inherits from `TagHelper`
```csharp
public class ListGroupTagHelper : TagHelper
{
}
```

The above class will make the runtime to target the HTML element `<list-group>` using the convention **TagName**TagHelper and because our tag has more than one word it will target the tag using [kebab-case](http://stackoverflow.com/a/12273101/499930) which is `<list-group>`. The `HtmlTargetElement` could be used to avoid using the convention and be explicit about the target element.

```csharp
[HtmlTargetElement("list-group",Attributes=ItemsAttribute)]
public class ListGroupTagHelper : TagHelper
{
   public const string ItemsAttribute = "asp-items";
}
```
The `HtmlTargetElement` constructor accepts the following parameters

 - Element: This is the html tag the will be targeted by the tag helper
 - Attributes: Comma separated list of attributes that all must exist to fire the tag helper

The `HtmlTargetElement` allows using it multiple time over the class which enables scenarios that require using the same tag helper against multiple tags or targeting the same tag with multiple attribute combinations.

## Tag Helper Parameters
We can provide parameters to the Tag helper using public properties.
For example, we can provide the list of strings that will be processed by the tag helper by creating a new property called `Items` of type `List<string>` then use the attribute `HtmlAttributeName` to link what is the html attribute that will be used to pass the items to the tag helper

```csharp
[HtmlTargetElement("list-group",Attributes=ItemsAttribute)]
public class ListGroupTagHelper : TagHelper
{
   public const string ItemsAttribute = "asp-items";
   [HtmlAttributeName(ItemsAttribute)]
   public List<string> Items { get; set; }
   public override void Process(TagHelperContext context, TagHelperOutput output)
   {
   }
}
```

In case your TagHelper has a public property that is not supposed to be bound through the HTML then use the attribute `[HtmlAttributeNotBound]`, for example to access the executing `ViewContext`, create a property of type `ViewContext` and add the attributes `ViewContext` and `[HtmlAttributeNotBound]` to it.

```csharp
[HtmlAttributeNotBound]
[ViewContext]
public ViewContext ViewContext { get; set; }
```

## The Processing Logic
The `TagHelper` class contains two methods `Process` and `ProcessAsync`,In this post we will use `Process`

```csharp
public override void Process(TagHelperContext context, TagHelperOutput output)
    {
    if (context == null)
    {
        throw new ArgumentNullException(nameof(context));
    }
    if (output == null)
    {
        throw new ArgumentNullException(nameof(output));
    }
    if (Items == null)
    {
        throw new InvalidOperationException($"{nameof(Items)} must be provided");
    }
    output.TagName = "ul";
    output.Attributes["class"] = "list-group";
    foreach (var item in Items)
    {
        TagBuilder itemBuilder = new TagBuilder("li");
        itemBuilder.AddCssClass("list-group-item");
        itemBuilder.InnerHtml.Append(item);
        output.Content.Append(itemBuilder);
    }
}
```

The `Process` method accepts two parameters:

 - `TagHelperContext`: This parameter contains information about the tag addressed by the tag helper including all its attributes and children elements.
 - `TagHelperOutput`: Used to generated the tag output.

The `Process` methodsstarts with checking that the method parameters are not null. The code is using the new C# [nameof](https://msdn.microsoft.com/en-au/library/dn986596.aspx) operator to make the code more maintainable.

The exception `InvalidOperationException` is thrown when the `Items` are null because most of the built-in tag helpers using that exception for similar cases.

The output.TagName is used to change the current tag from `<list-group>` into `<ul>`, then loop over the items and for each item generate new `<li>` tag using the `TagBuilder` class.

> The above code is using the RC1 version for setting the attributes which will change in RC2 according to this [announcement](https://github.com/aspnet/Announcements/issues/151)

## Using The Tag Helper
First we need to import the TagHelpers into the view to be able to use them. By updating the file called *_ViewImports.cshtml* located inside the *views* folder to include the following line

```csharp
@addTagHelper "*, projectname"
```

The below snippet shows how to use the `ListGroupTagHelper` inside view to display the list group of the Items property inside the model assuming that the Items property is a `List<string>`

```html
<list-group asp-items="Model.Items"></list-group>
```

## Summary
Authoring custom tag helper is a straight forward and here are the steps:

 1. Decide the target tag.
 2. Create class that inherits from TagHelper class and use the `HtmlTargetElement` to define the target HTML tag.
 3. Use public properties with `HtmlAttributeName` on them to define the bound HTML attributes.
 4. Override either Process or ProcessAsync method with your own rendering logic
 5. Import the developed tag into your views using `addTagHelper`
 6. Finally use your HTML tag inside your view.
