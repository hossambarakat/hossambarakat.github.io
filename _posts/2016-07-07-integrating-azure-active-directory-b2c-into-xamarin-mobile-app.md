---
layout: post
title: Integrating Azure Active Directory B2C into Xamarin Mobile App
tags: Xamarin, Mobility, .net
---


When building mobile apps, it's often required to add authentication to protect the app's content. Authentication could be through social identity providers like Facebook, Amazon, LinkedIn, etc or through application based credentials. [Azure Active Directory Business to Consumer (Azure AD B2C)](https://azure.microsoft.com/en-us/services/active-directory-b2c/) provides adding identity management functions into application with little coding. 

I will explain how to add customer credentials authentication into Xamarin Forms mobile app targeting iOS and Android using Azure Active Directory B2C:

- [Configuring Azure AD B2C tenant](#configuring-azure-ad-b2c-tenant)
 - [Create Azure AD B2C tenant](#create-azure-ad-b2c-tenant)
 - [Register application into the tenant](#register-application-into-the-tenant)
 - [Configure sign-up/sign-in policy](#configure-sign-up-sign-in-policy)
- [Building Xamarin Forms application](#building-xamarin-forms-application)
 - [Empty App](#empty-app)
 - [Welcome Screen](#welcome-screen)
 - [Sign in/Register](#sign-in-register)
 - [Secure Page](#secure-page)
 - [Sign out](#sign-out)

## <a id="configuring-azure-ad-b2c-tenant"></a>Configuring Azure AD B2C Tenant
To start using Azure AD B2C, you have to create a new tenant

### <a id="create-azure-ad-b2c-tenant"></a>Create a new directory
1. Log in to the [Azure portal](https://manage.windowsazure.com/). 
2. Click New.
3. Click App Services > Active Directory > Directory > Custom create > Create and manage a new Microsoft Azure AD directory. 
4. Complete the fields of the add directory page:

![](/assets/integrating-azure-b2c/Create-Active-Directory-Tenant-4-1.png)

### <a id="register-application-into-the-tenant"></a>Register application into the tenant 
On the quick start page on the created directory, under Administer, click Manage B2C settings. This will open the new Azure portal. 

On the B2C features blade on the Azure portal, click Applications then click Add at the top of the blade and fill the details:

![](/assets/integrating-azure-b2c/AddApplicationToTenant-1.png)

Click Create to register your application and then copy down the globally unique Application Client ID because we will use it later while building the application.

![](/assets/integrating-azure-b2c/Application-Client-ID-5.png)

### Choose identity providers
Azure AD B2C offers multiple social identity providers Microsoft, Google, Amazon, LinkedIn and Facebook in addition to the local accounts. For the sake of this article, we will keep it to the default identity provider so no need to apply changes.


### <a id="configure-sign-up-sign-in-policy"></a>Configure sign-up/sign-in policy
Policies define the behaviour of sign in, sign up, etc. We will use the unified "Sign-up or Sign-in" policy which will provide the application a single experience for the sign up and sign in. To configure this policy:

Click on "Sign-up or sign-in policies" > Add > then fill the the details:

- **Policy Name:** This field defines the name of the policy that we will use in our application.
- **Identity Providers:** The set of identity providers that will be supported in this policy - in our case it will be "Email signup".
![](/assets/integrating-azure-b2c/Identity-Providers-1.png)
- **Sign-up attributes:** The set of attributes the user must provide during the registration process.
![](/assets/integrating-azure-b2c/Signup-Attributes-3.png)
- **Application claims:** The claims that will be sent back to the application after the successful log in.
![](/assets/integrating-azure-b2c/Application-Claims-1.png)
- **Multifactor authentication:** You can enable Multifactor authentication where the email can use email or mobile to receive confirmation code before proceeding with the login process however we will not use it in this post.
- **Page UI customisation:** Policies allows customising the look and feel of pages such as sign up and sign in which is basically a CSHTML file.

Now we have finished preparing the AD B2C, let's jump into building the mobile application.


## <a id="building-xamarin-forms-application"></a> Building the Xamarin Forms application
We will build Xamarin forms application that targets both iOS and Android platforms. The application will have two screens:

1. A Welcome Screen that shows welcome message with sign in button to authenticate the user using Azure AD B2C.
2. A Secure Page that shows the logged in user name and sign out functionality.

### <a id="empty-app"></a> Empty App
Will start with creating Xamarin Forms app that target both iOS and Android in Xamarin Studio or Visual Studio.

### Adding Microsoft Authentication Library
MSAL is a developer library that helps you to obtain tokens from MSA, Azure AD or Azure B2C for accessing protected resources.

> MSAL helps you with showing the necessary authentication, multi-factor authentication and consent UX in a platform-appropriate fashion; it takes care of crafting, sending, receiving, validating and interpreting the protocol messages that are required for implementing the authentication flows you need; and it takes care of persisting tokens for you, transparently using all the tricks in the book (such as transparent usage of refresh tokens) to minimize the number of authentication prompts presented to your users.[[2]](#ms-identity-build)

So, for each project, add a reference to [the Microsoft Authentication Library (MSAL) NuGet package](https://www.nuget.org/packages/Microsoft.Identity.Client/1.0.304142221-alpha) and make sure you include the pre-release packages while installing.

#### Android Special Handling
We have to update the `OnActivityResult` of the `MainActivity` in the Android project with the line of code below to ensure the control goes back to MSAL once the interactive portion of the authentication flow ended:

```csharp
AuthenticationAgentContinuationHelper.SetAuthenticationAgentContinuationEventArgs(requestCode, resultCode, data);
protected override void OnActivityResult(int requestCode, Result resultCode, Intent data)
{
    base.OnActivityResult(requestCode, resultCode, data);
    AuthenticationAgentContinuationHelper.SetAuthenticationAgentContinuationEventArgs(requestCode, resultCode, data);
}
```

### <a id="welcome-screen"></a> Welcome Screen
Moving into adding the welcome screen, add new Xamarin forms Xaml content page with the name `WelcomePage` and update the Xaml:

```xml
<StackLayout Orientation="Vertical" VerticalOptions="FillAndExpand">
    <Label Text="Welcome to Xamarin Forms!" VerticalTextAlignment="Center" VerticalOptions="FillAndExpand" HorizontalOptions="Center" />
    <StackLayout Orientation="Horizontal" VerticalOptions="End" HorizontalOptions="CenterAndExpand">
        <Button Text="Sign In" x:Name="SignIn" Clicked="OnSignUpSignIn_Clicked" HorizontalOptions="Center"/>
    </StackLayout>
</StackLayout>
``` 


We will navigate between welcome screen and secure page so update the constructor inside the file `app.xaml.cs`:

```csharp
public App()
{
    InitializeComponent();
    MainPage = new NavigationPage(new WelcomePage());
}
```

The MSAL requires set of parameters so I will create class `AuthParameters` with all the required parameters and will initiate them with the values from the previous steps:

```csharp
public class AuthParameters
    {
        public const string Authority = "https://login.microsoftonline.com/HossamB2CDemo.onmicrosoft.com/";
        public const string ClientId = "1aa3e632-bd72-4040-8d4a-854242bd4f20";
        public static readonly string[] Scopes = { ClientId };
        public const string Policy = "B2C_1_default_signup_signin";
    }
```

The welcome screen will redirect the user to the secure page whenever there is a stored token, so we will update the `WelcomePage.OnAppearing`:

```csharp
try
{
    PublicClientApplication publicClientApplication = new PublicClientApplication(AuthParameters.Authority, AuthParameters.ClientId);
    var authResult = await publicClientApplication.AcquireTokenSilentAsync(AuthParameters.Scopes, "", AuthParameters.Authority, AuthParameters.Policy, false);
    await Navigation.PushAsync(new SecurePage());
}
catch
{
    
}
```

I am using the main class in MSAL `PublicClientApplication` trying to retrieve any stored token by calling the method `AcquireTokenSilentAsync`. `AcquireTokenSilentAsync` throughs an exception if no token found so we have to wrap our code with try catch. if  the call succeeds, we redirect the user to the `SecurePage`.

<a id="sign-in-register"></a>
### Sign in/Register
Add the sign in click button handler: 

```csharp
async void OnSignUpSignIn_Clicked(object sender, EventArgs e)
{
    try
    {
        PublicClientApplication publicClientApplication = new PublicClientApplication(AuthParameters.Authority, AuthParameters.ClientId);
        publicClientApplication.PlatformParameters = PlatformParameters;
        var authResult = await publicClientApplication.AcquireTokenAsync(AuthParameters.Scopes, "", UiOptions.SelectAccount, string.Empty, null, AuthParameters.Authority, AuthParameters.Policy);
        await Navigation.PushAsync(new SecurePage());
    }
    catch (Exception exception)
    {
        await DisplayAlert("An error has occurred", exception.Message, "Dismiss");
    }
}
```

The code is similar to the one we have used to acquire the token inside `OnAppearing` of the `WelcomePage` however there are two differences:

- `AcquireTokenAsync`: This method will get the user into the interactive mode of showing web view, to sign up or sign in. The functionalities that appear inside the web view, are mainly derived from the policy configurations which gives more flexibility like enabling multi-factor authentication at any time or adding extra attribute to the user profile.
- `PlatformParameters`: This parameter helps the MSAL to detect which platform is currently in use so it can correctly handle the interactive mode and the value for this parameter must be set through each platform so we have to update the `WelcomePage` to contain public variable that we set through a renderer inside each platform.

### Setting Platform Parameters
Add the public property `PlatformParameters` to the `WelcomePage`: 

```csharp
public IPlatformParameters PlatformParameters { get; set; }
```

Then, create `WelcomePageRender` inside the iOS project:

```csharp
[assembly: ExportRenderer(typeof(WelcomePage), typeof(WelcomePageRenderer))]
namespace B2CDemo.iOS
{
    class WelcomePageRenderer : PageRenderer
    {
        WelcomePage page;
        protected override void OnElementChanged(VisualElementChangedEventArgs e)
        {
            base.OnElementChanged(e);
            page = e.NewElement as WelcomePage;
        }
        public override void ViewDidLoad()
        {
            base.ViewDidLoad();
            page.PlatformParameters = new PlatformParameters(this);
        }
    }
}
``` 

And, another renderer inside the Android project:

```csharp
[assembly: ExportRenderer(typeof(WelcomePage), typeof(WelcomePageRenderer))]
namespace B2CDemo.Droid
{    
    class WelcomePageRenderer : PageRenderer
    {
        WelcomePage page;

        protected override void OnElementChanged(ElementChangedEventArgs<Page> e)
        {
            base.OnElementChanged(e);
            page = e.NewElement as WelcomePage;
            var activity = this.Context as Activity;
            page.PlatformParameters = new PlatformParameters(activity);
        }
    }
}
``` 

### <a id="secure-page"></a>Secure Page
This page will show the current user name so update the `OnAppearing`:

```csharp
protected async override void OnAppearing()
{
    try
    {
        PublicClientApplication publicClientApplication = new PublicClientApplication(AuthParameters.Authority, AuthParameters.ClientId);
        var authResult = await publicClientApplication.AcquireTokenSilentAsync(AuthParameters.Scopes, "", AuthParameters.Authority, AuthParameters.Policy, false);
        Token.Text = authResult.IdToken;
        Name.Text = "Welcome Secure User:" + authResult.User.Name;

    }
    catch(Exception ex)
    {
        //failed to retrieve token
        PublicClientApplication publicClientApplication = new PublicClientApplication(AuthParameters.Authority, AuthParameters.ClientId);
        publicClientApplication.UserTokenCache.Clear(AuthParameters.ClientId);
        await Navigation.PushAsync(new WelcomePage());
    }
}
```

The code is very similar to the one that we have used in the welcome screen with the addition of grabbing the user name from the result using `result.User.Name`

You can use `authResult.IdToken` to acquire the Access Token to call any secured APIs. 


### <a id="sign-out"></a>Sign out 
The sign out is easy, all we need to do is remove the access token from the cache and then navigate the user to the welcome screen:

```csharp
async void SignOut_Clicked(object sender, System.EventArgs e)
{
    PublicClientApplication publicClientApplication = new PublicClientApplication(App.Authority, App.ClientId);
    publicClientApplication.UserTokenCache.Clear(App.PCApplication.ClientId);
    await Navigation.PushAsync(new WelcomePage());
}
```

Here, you should be able to run the application with all the functionalities working.


## References
1. [Azure Active Directory B2C documentation](https://azure.microsoft.com/en-us/documentation/services/active-directory-b2c/)
2. <a id="ms-identity-build"></a>[Microsoft Identity at //build/ 2016](https://blogs.technet.microsoft.com/enterprisemobility/2016/03/31/microsoft-identity-at-build-2016/)
3. [Azure AD B2C and B2B Public Preview Announcement](https://blogs.technet.microsoft.com/enterprisemobility/2015/09/16/azure-ad-b2c-and-b2b-are-now-in-public-preview/)
4. [Azure Active Directory B2C documentation](https://azure.microsoft.com/en-us/documentation/services/active-directory-b2c/)
5. [More preview enhancements for Azure AD B2C](https://blogs.technet.microsoft.com/enterprisemobility/2016/05/05/more-preview-enhancements-for-azure-ad-b2c/)
