---
layout: post
title: Client Credentials Grant Flow with Azure AD B2C
tags: Azure B2C, Azure
---

One of the known limitations of Azure AD B2C is not directly supporting the OAuth 2.0 client credentials grant flow as it is clearly stated in [the documentation](https://docs.microsoft.com/en-us/azure/active-directory-b2c/application-types#daemonsserver-side-applications). The documentation also hint that you can use the OAuth 2.0 client credentials flow because An Azure AD B2C tenant shares some functionality with Azure AD enterprise tenants however there is no details on how to achieve that.

In this post, I will write down how to issue an access token using the OAuth 2.0 client credentials flow and use it invoke a secure API.

## Register a web application in Azure Active Directory B2C

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Select the **Directory + Subscription** icon in the portal toolbar, and then select the directory that contains your Azure AD B2C tenant.
1. In the Azure portal, search for and select **Azure AD B2C**.
1. Select **App registrations**, and then select **New registration**.
1. Enter a **Name** for the application. For example, *myapp*.
1. Under **Supported account types**, select **Accounts in any organizational directory or any identity provider. For authenticating users with Azure AD B2C**.
1. Under **Permissions**, select the *Grant admin consent to openid and offline_access permissions* check box.
1. Select **Register**.

## Create a client secret

1. In the **Azure AD B2C - App registrations** page, select the application you created, for example *webapp1*.
1. In the left menu, under **Manage**, select **Certificates & secrets**.
1. Select **New client secret**.
1. Enter a description for the client secret in the **Description** box. For example, *clientsecret1*.
1. Under **Expires**, select a duration for which the secret is valid, and then select **Add**.
1. Record the secret's **Value**. You use this value as the application secret in your application's code.

## Configure scope

Scopes provide a way to govern access to protected resources. You need to set the scope of the application so you can use the application URI to request an access token:

1. Select **App registrations**.
1. Select the *myapp* application to open its **Overview** page.
1. Under **Manage**, select **Expose an API**.
1. Next to **Application ID URI**, select the **Set** link.
1. Select **Save**. The full URI is shown, and should be in the format `https://your-tenant-name.onmicrosoft.com/guid`. When your application requests an access token for the API, it should add this URI as the prefix for each scope.


## Get a token

You can get a token by using the client credentials grant via sending a POST request to the /token Microsoft identity platform endpoint. The following snippet show how to use the excellent [IdentityModel library](https://www.nuget.org/packages/IdentityModel) to get a token

```csharp
  var response = await httpClient.RequestClientCredentialsTokenAsync(new ClientCredentialsTokenRequest
  {
      Address = "https://login.microsoftonline.com/b2c-tenant.onmicrosoft.com/oauth2/v2.0/token",
      ClientId = "application GUID",
      ClientSecret = "client secret",
      Scope = "https://b2c-tenant.onmicrosoft.com/c0976379-3467-4e8a-8ff9-05f652d1cde2/.default"
  });
  var accessToken = response.AccessToken;
```

| Parameter | Description |
| --- | --- |
| `Address` | The token endpoint of the directory tenant that you want to request permission from. Make sure to replace `b2c-tenant` with your directory name. |
| `client_id` | The **Application (client) ID** that has been assigned to your app. |
| `client_secret` | The client secret that you generated for your app in the app registration portal. |
| `Scope` | The value passed for the scope parameter in this request should be the Application ID URI that was configured in "Configure scope" step, affixed with the .default suffix. Make sure to replace `b2c-tenant` with your directory name. |

## Secure ASP.NET Core API

Now that you have the access token, you can invoke a remote API with the `Authorization` header holding the access token but how would you make sure that the API accepts this token.

Securing ASP.NET Core API with JWT tokens is straightforward. You can use the Middleware that exists in the `Microsoft.AspNetCore.Authentication.JwtBearer` package by calling `AddAuthentication` and `AddJwtBearer` in the `ConfigureServices` of the `Startup` class

```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
  .AddJwtBearer(options =>
  {
      options.Audience = "Application (client) ID";
      options.Authority = "https://login.microsoftonline.com/<tenant GUID>";
      options.TokenValidationParameters = new Microsoft.IdentityModel.Tokens.TokenValidationParameters
      {
          ValidAudience = "Application (client) ID";
          ValidIssuer = "https://login.microsoftonline.com/<tenant GUID>/v2.0"
      };
  });
```

Then, in the `Configure` method, add `UseAuthentication`:

```csharp
app.UseAuthentication();
```

I always forget adding `UseAuthentication()` so make sure that you didn't forget to add it to your class :).

Finally, Add the [Authorize] attribute on the controllers or action methods that you want to secure:

```csharp
[HttpGet]
[Authorize]
public string Secure()
{
    return "secure api response";
}
```

## Secure API.NET Core API with tokens issued by other OAuth 2.0 flows

You might be wondering how to get your API to accept tokens issued by other OAuth 2.0 flows like OAuth 2.0 with PKCE. ASP.NET Core can accept multiple authentication methods by adding another `AddJwtBearer` in your `ConfigureServices` method.

```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
  .AddJwtBearer(options =>
  {
      options.Authority = $"https://my-b2c-tenant.b2clogin.com/my-b2c-tenant.onmicrosoft.com/SignUpSignInPolicyId/v2.0";
      options.Audience = "api app client GUID";
  })
  .AddJwtBearer("AzureAD", options =>
  {
      options.Audience = "Application (client) ID";
      options.Authority = "https://login.microsoftonline.com/<tenant GUID>";
      options.TokenValidationParameters = new Microsoft.IdentityModel.Tokens.TokenValidationParameters
      {
          ValidAudience = "Application (client) ID";
          ValidIssuer = "https://login.microsoftonline.com/<tenant GUID>/v2.0"
      };
  });
```

Then, You update your controllers or action method `Authorize` attribute to define the `AuthenticationSchemes`

```csharp
[HttpGet]
[Authorize(AuthenticationSchemes =
    JwtBearerDefaults.AuthenticationScheme)]
public string Secure()
{
    return "secure api response";
}
```

If you don't like specifying the `AuthenticationSchemes` on controllers or action methods and would like to accept both authentication schemes by updating the default authorization policy:

```csharp
services.AddAuthorization(options =>
{
    var defaultAuthorizationPolicyBuilder = new AuthorizationPolicyBuilder(
        JwtBearerDefaults.AuthenticationScheme,
        "AzureAD");
    defaultAuthorizationPolicyBuilder = defaultAuthorizationPolicyBuilder.RequireAuthenticatedUser();
    options.DefaultPolicy = defaultAuthorizationPolicyBuilder.Build();
});
```

Congratulations, You know how to issue an access token with Azure AD B2C Client Credentials grant flow and how to use it to call a secure ASP.NET Core API.

## References
1. [Register a web application in Azure Active Directory B2C](https://docs.microsoft.com/en-us/azure/active-directory-b2c/tutorial-register-applications?tabs=app-reg-ga)
1. [Add a web API application to your Azure Active Directory B2C tenant](https://docs.microsoft.com/en-us/azure/active-directory-b2c/add-web-api-application?tabs=app-reg-ga)
1. [Authorize with a specific scheme in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/limitingidentitybyscheme?view=aspnetcore-3.1)
