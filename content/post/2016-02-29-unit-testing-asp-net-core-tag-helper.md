---
date: "2016-02-29T00:00:00Z"
tags: ["asp.net", "asp.net-core"]
title: Unit Testing ASP.NET Core Tag Helper
---

Tag helpers is one of the new additions to ASP.NET Core MVC 1.0 features which enables the tags to participate in the server side rendering with an easy syntax that is easy to write and read specially for front-end developers who don't have any previous knowledge with Razor syntax.

This post will go through testing and ASP.NET core MVC tag helper that I have explained how to build [in a previous post](http://www.hossambarakat.net/2016/02/15/authoring-asp-net-core-mvc-tag-helper/)

## Create a class library project
The first step is creating a class library project targeting .NET 4.5 or later. Name the project *TagHelperDemoTests*

## Reference xUnit
This post will use the [xUnit](http://xunit.github.io/) framework for testing so update the `project.json` to reference both xunit and xunit runner

```json
{
  "version": "1.0.0-*",
  "description": "",
  "authors": [ "" ],
  "tags": [ "" ],
  "projectUrl": "",
  "licenseUrl": "",
  "dependencies": {
    "xunit": "2.1.0-*",
    "xunit.runner.dnx": "2.1.0-rc1-build204"
  },
  "frameworks": {
    "dnx451": { },
    "dnxcore50": { }
  },
  "commands": {
    "test": "xunit.runner.dnx"
  }
}
```

## Reference the tag helper project
Update the dependencies section inside the `project.json` file to reference the project that contains the tag helper to be tested which is `TagHelperDemo` in my case.

```json
{
  "version": "1.0.0-*",
  "description": "",
  "authors": [ "" ],
  "tags": [ "" ],
  "projectUrl": "",
  "licenseUrl": "",
  "dependencies": {
    "xunit": "2.1.0-*",
    "xunit.runner.dnx": "2.1.0-rc1-build204",
    "TagHelperDemo": "1.0.0-*"
  },
  "frameworks": {
    "dnx451": { },
    "dnxcore50": { }
  },
  "commands": {
    "test": "xunit.runner.dnx"
  }
}
```

## Writing a simple test
Add new class to your project and name that class `ListGroupTagHelperTests`

```csharp
using Xunit;
namespace TagHelperDemoTests
{
    public class ListGroupTagHelperTests
    {
        [Fact]
        public void VerifyTestConfiguration()
        {
            Assert.True(1==1);
        }
    }
}
```

Now you can run the tests using visual studio test explorer or open the command prompt and navigate to the test project directory and run the tests using the command `dnx test`, the outcome should look like the following

```
xUnit.net DNX Runner (32-bit DNX 4.5.1)
  Discovering: TagHelperDemoTests
  Discovered:  TagHelperDemoTests
  Starting:    TagHelperDemoTests
  Finished:    TagHelperDemoTests
=== TEST EXECUTION SUMMARY ===
   TagHelperDemoTests  Total: 1, Errors: 0, Failed: 0, Skipped: 0, Time: 0.078s
```

## Write the tag helper test
Typically tag helpers implement `Process` or `ProcessAsync` methods so obviously your tests will be addressing those methods. The `Process` method accepts two parameters the `TagHelperContext` which contains information about the tag addressed by the tag helper and the `TagHelperOutput` that is used to generate the tag output.

### TagHelperContext
Create an instance of the `TagHelperContext`, its constructor accepts three parameters:

- **allAttributes**: List of attributes associated with the current HTML tag and in our case this is empty.
- **items**: Dictionary of objects which is usually used to transfer data between tag helpers and we are not using it in our case
- **uniqueId**: Unique id for the HTML tag.

```csharp
var context = new TagHelperContext(
                new TagHelperAttributeList(),
                new Dictionary<object, object>(),
                Guid.NewGuid().ToString("N"));
```

### TagHelperOutput
Create an instance of the TagHelperOutput, its constructor accepts three parameters:

- **tagName**: The tag name which is `list-group` in our case
- **attributes**: The list of attributes which is empty in our case
- **getChildContentAsync**: A delegate used to execute and retrieve the rendered child content asynchronously

```csharp
var output = new TagHelperOutput(
                "list-group",
                new TagHelperAttributeList(),
                (useCachedResult) =>
                    {
                        var tagHelperContent = new DefaultTagHelperContent();
                        tagHelperContent.SetContent(string.Empty);
                        return Task.FromResult<TagHelperContent>(tagHelperContent);
                    }
                );
```

### Unit test logic
Create an instance of the `ListGroupTagHelper` class, set the Items property and call the Process method with the prepared parameters, then assert the outcome by assessing the properties of the `TagHelperOutput`.

```csharp
using Microsoft.AspNet.Razor.TagHelpers;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Xunit;
using TagHelperDemo;

namespace TagHelperDemoTests
{
    public class ListGroupTagHelperTests
    {
        public static TheoryData<List<string>,string> GeneratesExpectedOutputDataSet
        {
            get
            {
                return new TheoryData<List<string>, string>
                {
                    { new List<string>() , "" },
                    { new List<string> { "Saturday" } , $"<li class=\"list-group-item\">Saturday</li>"}
                };
            }
        }
        [Theory]
        [MemberData(nameof(GeneratesExpectedOutputDataSet))]
        public void Process_GeneratesExpectedOutput(List<string> items,string expectedOutput)
        {
            ListGroupTagHelper myTagHelper = new ListGroupTagHelper();
            var context = new TagHelperContext(
                new TagHelperAttributeList(),
                new Dictionary<object, object>(),
                Guid.NewGuid().ToString("N"));
            myTagHelper.Items = items;

            var output = new TagHelperOutput(
                "list-group",
                new TagHelperAttributeList(),
                (useCachedResult) =>
                    {
                        var tagHelperContent = new DefaultTagHelperContent();
                        tagHelperContent.SetContent(string.Empty);
                        return Task.FromResult<TagHelperContent>(tagHelperContent);
                    }
                );

            myTagHelper.Process(context, output);
            Assert.Equal("ul", output.TagName);
            Assert.Equal(expectedOutput, output.Content.GetContent());
        }
    }
}
```

Run the command `dnx test`

```
xUnit.net DNX Runner (32-bit DNX 4.5.1)
  Discovering: TagHelperDemoTests
  Discovered:  TagHelperDemoTests
  Starting:    TagHelperDemoTests
  Finished:    TagHelperDemoTests
=== TEST EXECUTION SUMMARY ===
   TagHelperDemoTests  Total: 2, Errors: 0, Failed: 0, Skipped: 0, Time: 0.276s
```

## Summary
Writing unit test to address ASP.NET Core MVC tag helper could be done by doing the following steps:

1. Creating a class library project that reference xUnit as well as the project that holds the tag helper under test.
2. Create a test class with test method,
3. Prepare an instance of TagHelperContext, TagHelperOutput and the tag helper under test.
4. Call the `Process` or `ProcessAsync` methods and assert the the outcome.
