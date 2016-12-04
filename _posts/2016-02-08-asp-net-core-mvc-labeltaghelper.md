---
layout: post
title: ASP.NET Core MVC LabelTagHelper
tags: asp.net, asp.net-core
---


Tag Helpers enable server-side code to participate in creating and rendering HTML elements in Razor files.Tag Helpers reduce the explicit transitions between HTML and C# in Razor views.[^1]

`LabelTagHelper` could be considered one of the simplest tag helpers. It targets the html tag `<label>` with `asp-for` attribute. It is equivalent to `Html.LabelFor`.

Let's get into the usage steps. Assuming that the following class represent the model of the view

```csharp
public class ContactModel
{
    public string Mobile { get; set; }
}
```

First we need to import the TagHelpers into the view to be able to use them. By updating the file called *_ViewImports.cshtml* located inside the *views* folder to include the following line

```csharp
@addTagHelper "*, Microsoft.AspNet.Mvc.TagHelpers"
```

The below snippet shows how to use the LabelTagHelper inside view to display the label of the Mobile property inside the model

```html
<label asp-for="Mobile"></label>
```

the generated `HTML` will look like:

```html
<label for="Mobile">Mobile</label>
```
As shown above, the implementation of the `LabelTagHelper` provides default value to the label text.The label text could be defined by using the `DisplayAttribute` over the `Mobile` property so our model will be as shown underneath:

```csharp
public class ContactModel
{
    [Display(Name = "Mob")]
    public string Mobile { get; set; }
}
```

Then, the generated `HTML` will be as below

```html
<label for="Mobile">Mob</label>
```

On the other hand, the `LabelTagHelper` also takes into consideration the children of the `<Label>` tag, so the following snippet

```html
<label asp-for="Mobile"><img src="/images/mobile-image.png"/></label>
```
will generate the following `HTML`

```html
<label for="Mobile"><img src="/images/ASP-NET-Banners-01.png"></label>
``` 
Which means that the `DisplayAttribute` will take effect only if the content of the `<label>` tag was empty.

Hope that helps!

[^1]: Definition from the [*tag helpers* documentation](http://docs.asp.net/projects/mvc/en/latest/views/tag-helpers/intro.html)