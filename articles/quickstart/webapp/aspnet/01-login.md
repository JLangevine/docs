---
title: Login
default: true
description: This tutorial will show you how to add user login to an ASP.NET application using the Auth0 ASP.NET SDK.
budicon: 448
topics:
  - quickstarts
  - webapp
  - aspnet
  - login
github:
  path: Quickstart/00-Starter-Seed/auth0-aspnet-mvc4-sample
---
<%= include('../_includes/_getting_started', { library: 'ASP.NET', callback: 'http://localhost:PORT/LoginCallback.ashx' }) %>

## Integrate Auth0 in your Application

### Install the Auth0-ASPNET NuGet Package

Install the Auth0-ASPNET package. In the NuGet Package Manager, click **Tools** > **Library package manager** > **Package Manager Console**. In the console, run the command: 

${snippet(meta.snippets.dependencies)}

::: note
This package adds the `LoginCallback.ashx` file to your project to process the login. 
:::

### Fill Web.Config With Your Auth0 Settings

The NuGet package creates three settings on `<appSettings>`. Replace them with the following settings:

${snippet(meta.snippets.setup)}

## Trigger Authentication

To authenticate the user, redirect them to Auth0's `/authorize` endpoint:

```c#
// Controllers/AccountController.cs
public ActionResult Login(string returnUrl)
{
    var client = new AuthenticationApiClient(
        new Uri(string.Format("https://{0}", ConfigurationManager.AppSettings["auth0:Domain"])));


    var request = this.Request;
    var redirectUri = new UriBuilder(request.Url.Scheme, request.Url.Host, this.Request.Url.IsDefaultPort ? -1 : request.Url.Port, "LoginCallback.ashx");

    var authorizeUrlBuilder = client.BuildAuthorizationUrl()
        .WithClient(ConfigurationManager.AppSettings["auth0:ClientId"])
        .WithRedirectUrl(redirectUri.ToString())
        .WithResponseType(AuthorizationResponseType.Code)
        .WithScope("openid profile")
        .WithAudience("https://" + @ConfigurationManager.AppSettings["auth0:Domain"] + "/userinfo");

    if (!string.IsNullOrEmpty(returnUrl))
    {
        var state = "ru=" + HttpUtility.UrlEncode(returnUrl);
        authorizeUrlBuilder.WithState(state);
    }

    return new RedirectResult(authorizeUrlBuilder.Build().ToString());
}
```

## Display User Information

When the user logs in to the application, a `ClaimsPrincipal` class is generated. You can access it through the `Current` property:

```cs
// Controllers/HomeController.cs
public ActionResult Index()
{
  string name = ClaimsPrincipal.Current.FindFirst("name")?.Value;
}
```

The user profile you receive is always a normalized user profile. The profile includes the following attributes:
* `name`
* `nickname`
* `picture`
* `updated_at`

For more information about the user profile, read the [user profile documentation](/user-profile).

## Further Reading

### Handle authorization

On each request, the `LoginCallback.ashx` handler and the `Http` module generate an `IPrincipal`. Because of that, you can use the following authorization methods: 
* The declarative `[Authorization]` protection
* The `<location path='..'>` protection
* Code-based checks, for example, `User.Identity.IsAuthenticated`

### Automatically redirect to the login page

If the request is not authenticated, the `[Authorize]` attribute generates a 401 (Unauthorized) error. If you want to automatically redirect users to the login page, you can use the Forms Authentication module. 

In `web.config`, configure the following:

```xml
<!--Web.config-->
<system.web>
  <authentication mode="Forms">
    <forms loginUrl="Account/Login" />
  </authentication>
</system.web>
```

In the above example, we are redirecting to the `Login` action in an `Account` controller, which in turn redirects to Auth0's `/authorize` endpoint for authentication, as described in [#4](#4-authenticating-the-user).

### Set up Logout

To clear the cookie generated on login, use the `FederatedAuthentication.SessionAuthenticationModule.SignOut()` method on the `AccountController\Logout` method.

The example below shows a typical logout action on ASP.Net MVC:

```cs
// Controllers/AccountController.cs
public RedirectResult Logout()
{
  // Clear the session cookie
  FederatedAuthentication.SessionAuthenticationModule.SignOut();

  // Redirect to Auth0's logout endpoint
  var returnTo = Url.Action("Index", "Home", null, protocol: Request.Url.Scheme );
  return this.Redirect(
    string.Format(CultureInfo.InvariantCulture,
      "https://{0}/v2/logout?returnTo={1}",
      ConfigurationManager.AppSettings["auth0:Domain"],
      this.Server.UrlEncode(returnTo)));
}
```

The destination URL is stored in the `returnTo` value. 

::: note 
The destination URL must be on the`Allowed Logout URLs` list. Read more about redirecting users after they log out in the [Logout](/logout#redirecting-users-after-logout) article.
:::

### Flow the user's identity to a WCF service

If you want to flow the logged-in user's identity to a WCF service or an API, use the `responseType: 'token'` parameter on the login widget constructor. When the parameter is sent, Auth0 generates an ID Token. You can send the ID Token to your service or use it to generate an `ActAs` token. The ID Token is a [JSON Web Token](http://tools.ietf.org/html/draft-ietf-oauth-json-web-token-06).
