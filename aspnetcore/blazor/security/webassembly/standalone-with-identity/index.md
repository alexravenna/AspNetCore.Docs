---
title: Secure ASP.NET Core Blazor WebAssembly with ASP.NET Core Identity
author: guardrex
description: Learn how to secure Blazor WebAssembly apps with ASP.NET Core Identity.
monikerRange: '>= aspnetcore-8.0'
ms.author: wpickett
ms.custom: mvc
ms.date: 07/29/2025
uid: blazor/security/webassembly/standalone-with-identity/index
---
# Secure ASP.NET Core Blazor WebAssembly with ASP.NET Core Identity

[!INCLUDE[](~/includes/not-latest-version-without-not-supported-content.md)]

Standalone Blazor WebAssembly apps can be secured with ASP.NET Core Identity by following the guidance in this article.

## Endpoints for registering, logging in, and logging out

Instead of using the default UI provided by ASP.NET Core Identity for SPA and Blazor apps, which is based on Razor Pages, call <xref:Microsoft.AspNetCore.Routing.IdentityApiEndpointRouteBuilderExtensions.MapIdentityApi%2A> in a backend API to add JSON API endpoints for registering and logging in users with ASP.NET Core Identity. Identity API endpoints also support advanced features, such as two-factor authentication and email verification.

On the client, call the `/register` endpoint to register a user with their email address and password:

```csharp
using var result = await _httpClient.PostAsJsonAsync(
    "register", new
    {
        email,
        password
    });
```

On the client, log in a user with cookie authentication using the `/login` endpoint with `useCookies` query string set to `true`:

```csharp
using var result = await _httpClient.PostAsJsonAsync(
    "login?useCookies=true", new
    {
        email,
        password
    });
```

The backend server API establishes cookie authentication with a call to <xref:Microsoft.AspNetCore.Identity.IdentityCookieAuthenticationBuilderExtensions.AddIdentityCookies%2A> on the authentication builder:

```csharp
builder.Services
    .AddAuthentication(IdentityConstants.ApplicationScheme)
    .AddIdentityCookies();
```

## Token authentication

For native and mobile scenarios where some clients don't support cookies, the login API provides a parameter to request tokens.

> [!WARNING]
> We recommend using cookies for browser-based apps instead of tokens because the browser handles cookies without exposing them to JavaScript. If you opt to use token-based security in web apps, you're responsible for ensuring the tokens are kept secure.

A custom token (one that is proprietary to the ASP.NET Core Identity platform) is issued that can be used to authenticate subsequent requests. The token should be passed in the `Authorization` header as a bearer token. A refresh token is also provided. This token allows the app to request a new token when the old one expires without forcing the user to log in again.

The tokens are not standard JSON Web Tokens (JWTs). The use of custom tokens is intentional, as the built-in Identity API is meant primarily for simple scenarios. The token option is not intended to be a fully-featured identity service provider or token server, but instead an alternative to the cookie option for clients that can't use cookies.

The following guidance begins the process of implementing token-based authentication with the login API. Custom code is required to complete the implementation. For more information, see <xref:security/authentication/identity/spa>.

Instead of the backend server API establishing cookie authentication with a call to <xref:Microsoft.AspNetCore.Identity.IdentityCookieAuthenticationBuilderExtensions.AddIdentityCookies%2A> on the authentication builder, the server API sets up bearer token auth with the <xref:Microsoft.Extensions.DependencyInjection.BearerTokenExtensions.AddBearerToken%2A> extension method. Specify the scheme for bearer authentication tokens with <xref:Microsoft.AspNetCore.Identity.IdentityConstants.BearerScheme%2A?displayProperty=nameWithType>.

In `Backend/Program.cs`, change the authentication services and configuration to the following: 

```csharp
builder.Services
    .AddAuthentication()
    .AddBearerToken(IdentityConstants.BearerScheme);
```

In `BlazorWasmAuth/Identity/CookieAuthenticationStateProvider.cs`, remove the `useCookies` query string parameter in the `LoginAsync` method of the `CookieAuthenticationStateProvider`:

```diff
- login?useCookies=true
+ login
```

At this point, you must provide custom code to parse the <xref:Microsoft.AspNetCore.Authentication.BearerToken.AccessTokenResponse> on the client and manage the access and refresh tokens. For more information, see <xref:security/authentication/identity/spa>.

## Additional Identity scenarios

Scenarios covered by the Blazor documentation set:

* [Account confirmation, password management, and recovery codes](xref:blazor/security/webassembly/standalone-with-identity/account-confirmation-and-password-recovery)
* [Two-factor authentication (2FA) with a TOTP authenticator app](xref:blazor/security/webassembly/standalone-with-identity/qrcodes-for-authenticator-apps)

For information on additional Identity scenarios provided by the API, see <xref:security/authentication/identity/spa>:

* Secure selected endpoints
* User info management

## Use secure authentication flows to maintain sensitive data and credentials

[!INCLUDE[](~/blazor/security/includes/secure-authentication-flows.md)]

## Sample apps

In this article, sample apps serve as a reference for standalone Blazor WebAssembly apps that access ASP.NET Core Identity through a backend web API. The demonstration includes two apps:

* `Backend`: A backend web API app that maintains a user identity store for ASP.NET Core Identity.
* `BlazorWasmAuth`: A standalone Blazor WebAssembly frontend app with user authentication.

Access the sample apps through the latest version folder from the repository's root with the following link. The samples are provided for .NET 8 or later. See the `README` file in the `BlazorWebAssemblyStandaloneWithIdentity` folder for steps on how to run the sample apps.

[View or download sample code](https://github.com/dotnet/blazor-samples) ([how to download](xref:blazor/fundamentals/index#sample-apps))

## Backend web API app packages and code

The backend web API app maintains a user identity store for ASP.NET Core Identity.

### Packages

The app uses the following NuGet packages:

* [`Microsoft.AspNetCore.Identity.EntityFrameworkCore`](https://www.nuget.org/packages/Microsoft.AspNetCore.Identity.EntityFrameworkCore)
* [`Microsoft.EntityFrameworkCore.InMemory`](https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.InMemory)
* [`NSwag.AspNetCore`](https://www.nuget.org/packages/NSwag.AspNetCore)

If your app is to use a different EF Core database provider than the in-memory provider, don't create a package reference in your app for `Microsoft.EntityFrameworkCore.InMemory`.

In the app's project file (`.csproj`), [invariant globalization](xref:blazor/globalization-localization#invariant-globalization) is configured.

### Sample app code

[App settings](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/Backend/appsettings.json) configure backend and frontend URLs:

* `Backend` app (`BackendUrl`): `https://localhost:7211`
* `BlazorWasmAuth` app (`FrontendUrl`): `https://localhost:7171`

The [`Backend.http` file](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/Backend/Backend.http) can be used for testing the weather data request. Note that the `BlazorWasmAuth` app must be running to test the endpoint, and the endpoint is hardcoded into the file. For more information, see <xref:test/http-files>.

The following setup and configuration is found in the app's [`Program` file](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/Backend/Program.cs). 

User identity with cookie authentication is added by calling <xref:Microsoft.Extensions.DependencyInjection.AuthenticationServiceCollectionExtensions.AddAuthentication%2A> and <xref:Microsoft.AspNetCore.Identity.IdentityCookieAuthenticationBuilderExtensions.AddIdentityCookies%2A>. Services for authorization checks are added by a call to <xref:Microsoft.Extensions.DependencyInjection.PolicyServiceCollectionExtensions.AddAuthorizationBuilder%2A>.

Only recommended for demonstrations, the app uses the [EF Core in-memory database provider](/ef/core/providers/in-memory/) for the database context registration (<xref:Microsoft.Extensions.DependencyInjection.EntityFrameworkServiceCollectionExtensions.AddDbContext%2A>). The in-memory database provider makes it easy to restart the app and test the registration and login user flows. Each run starts with a fresh database, but the app includes [test user seeding demonstration code](#test-user-seeding-demonstration), which is described later in this article. If the database is changed to SQLite, users are saved between sessions, but the database must be created through [migrations](/ef/core/managing-schemas/migrations/), as shown in the [EF Core getting started tutorial](/ef/core/get-started/overview/first-app)&dagger;. You can use other relational providers such as SQL Server for your production code.

> [!NOTE]
> &dagger;The EF Core getting started tutorial uses PowerShell commands to execute database migrations when using Visual Studio. An alternative approach in Visual Studio is to use the Connected Services UI:
>
> In **Solution Explorer**, double-click **Connected Services**. In **Service Dependencies** > **SQL Server Express LocalDB**, select the ellipsis (`...`) followed by either **Add migration** to create a migration or **Update database** to update the database.

Configure Identity to use the EF Core database and expose the Identity endpoints via the calls to <xref:Microsoft.Extensions.DependencyInjection.IdentityServiceCollectionExtensions.AddIdentityCore%2A>, <xref:Microsoft.Extensions.DependencyInjection.IdentityEntityFrameworkBuilderExtensions.AddEntityFrameworkStores%2A>, and <xref:Microsoft.AspNetCore.Identity.IdentityBuilderExtensions.AddApiEndpoints%2A>.

A [Cross-Origin Resource Sharing (CORS)](xref:security/cors) policy is established to permit requests from the frontend and backend apps. Fallback URLs are configured for the CORS policy if app settings don't provide them:

* `Backend` app (`BackendUrl`): `https://localhost:5001`
* `BlazorWasmAuth` app (`FrontendUrl`): `https://localhost:5002`

:::moniker range=">= aspnetcore-9.0"

The project includes packages and configuration to produce [OpenAPI documents](xref:fundamentals/openapi/overview).

:::moniker-end

:::moniker range="< aspnetcore-9.0"

Services and endpoints for [Swagger/OpenAPI](xref:tutorials/web-api-help-pages-using-swagger) are included for web API documentation and development testing. For more information on NSwag, see <xref:tutorials/get-started-with-nswag>.

:::moniker-end

User role claims are sent from a [Minimal API](xref:fundamentals/minimal-apis/overview) at the `/roles` endpoint.

Routes are mapped for Identity endpoints by calling `MapIdentityApi<AppUser>()`.

A logout endpoint (`/Logout`) is configured in the middleware pipeline to sign users out.

To secure an endpoint, add the <xref:Microsoft.AspNetCore.Builder.AuthorizationEndpointConventionBuilderExtensions.RequireAuthorization%2A> extension method to the route definition. For a controller, add the [`[Authorize]` attribute](xref:Microsoft.AspNetCore.Authorization.AuthorizeAttribute) to the controller or action.

For more information on basic patterns for initialization and configuration of a <xref:Microsoft.EntityFrameworkCore.DbContext> instance, see
[DbContext Lifetime, Configuration, and Initialization](/ef/core/dbcontext-configuration/) in the EF Core documentation.

## Frontend standalone Blazor WebAssembly app packages and code

A standalone Blazor WebAssembly frontend app demonstrates user authentication and authorization to access a private webpage.

### Packages

The app uses the following NuGet packages:

* [`Microsoft.AspNetCore.Components.WebAssembly.Authentication`](https://www.nuget.org/packages/Microsoft.AspNetCore.Components.WebAssembly.Authentication)
* [`Microsoft.Extensions.Http`](https://www.nuget.org/packages/Microsoft.Extensions.Http)
* [`Microsoft.AspNetCore.Components.WebAssembly`](https://www.nuget.org/packages/Microsoft.AspNetCore.Components.WebAssembly)
* [`Microsoft.AspNetCore.Components.WebAssembly.DevServer`](https://www.nuget.org/packages/Microsoft.AspNetCore.Components.WebAssembly.DevServer)

### Sample app code

The `Models` folder contains the app's models:

* [`FormResult` (`Identity/Models/FormResult.cs`)](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Identity/Models/FormResult.cs): Response for login and registration.
* [`UserInfo` (`Identity/Models/UserInfo.cs`)](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Identity/Models/UserInfo.cs): User info from identity endpoint to establish claims.

The [`IAccountManagement` interface (`Identity/CookieHandler.cs`)](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Identity/IAccountManagement.cs) provides account management services.

The [`CookieAuthenticationStateProvider` class (`Identity/CookieAuthenticationStateProvider.cs`)](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Identity/CookieAuthenticationStateProvider.cs) handles state for cookie-based authentication and provides account management service implementations described by the `IAccountManagement` interface. The `LoginAsync` method explicitly enables cookie authentication via the `useCookies` query string value of `true`. The class also manages creating role claims for authenticated users.

The [`CookieHandler` class (`Identity/CookieHandler.cs`)](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Identity/CookieHandler.cs) ensures cookie credentials are sent with each request to the backend web API, which handles Identity and maintains the Identity data store.

The [`wwwroot/appsettings.file`](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/wwwroot/appsettings.json) provides backend and frontend URL endpoints.

The [`App` component](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Components/App.razor) exposes the authentication state as a cascading parameter. For more information, see <xref:blazor/security/index#expose-the-authentication-state-as-a-cascading-parameter>.

The [`MainLayout` component](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Components/Layout/MainLayout.razor) and [`NavMenu` component](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Components/Layout/NavMenu.razor) use the [`AuthorizeView` component](xref:blazor/security/index#authorizeview-component) to selectively display content based on the user's authentication status.

The following components handle common user authentication tasks, making use of `IAccountManagement` services:

* [`Register` component (`Components/Identity/Register.razor`)](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Components/Identity/Register.razor)
* [`Login` component (`Components/Identity/Login.razor`)](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Components/Identity/Login.razor)
* [`Logout` component (`Components/Identity/Logout.razor`)](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Components/Identity/Logout.razor)

The [`PrivatePage` component (`Components/Pages/PrivatePage.razor`)](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Components/Pages/PrivatePage.razor) requires authentication and shows the user's claims.

Services and configuration is provided in the [`Program` file (`Program.cs`)](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Program.cs):

* The cookie handler is registered as a scoped service.
* Authorization services are registered.
* The custom authentication state provider is registered as a scoped service.
* The account management interface (`IAccountManagement`) is registered.
* The base host URL is configured for a registered HTTP client instance.
* The base backend URL is configured for a registered HTTP client instance that's used for auth interactions with the backend web API. The HTTP client uses the cookie handler to ensure that cookie credentials are sent with each request.

Call <xref:Microsoft.AspNetCore.Components.Authorization.AuthenticationStateProvider.NotifyAuthenticationStateChanged%2A?displayProperty=nameWithType> when the user's authentication state changes. For an example, see the `LoginAsync` and `LogoutAsync` methods of the [`CookieAuthenticationStateProvider` class (`Identity/CookieAuthenticationStateProvider.cs`)](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Identity/CookieAuthenticationStateProvider.cs).

> [!WARNING]
> The <xref:Microsoft.AspNetCore.Components.Authorization.AuthorizeView> component selectively displays UI content depending on whether the user is authorized. All content within a Blazor WebAssembly app placed in an <xref:Microsoft.AspNetCore.Components.Authorization.AuthorizeView> component is discoverable without authentication, so sensitive content should be obtained from a backend server-based web API after authentication succeeds. For more information, see the following resources:
>
> * <xref:blazor/security/index#authorizeview-component>
> * <xref:blazor/call-web-api>
> * <xref:blazor/security/webassembly/additional-scenarios>

## Test user seeding demonstration

The `SeedData` class ([`SeedData.cs`](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/Backend/SeedData.cs)) demonstrates how to create test users for development. The test user, named Leela, signs into the app with the email address `leela@contoso.com`. The user's password is set to `Passw0rd!`. Leela is given `Administrator` and `Manager` roles for authorization, which enables the user to access the manager page at `/private-manager-page` but not the editor page at `/private-editor-page`.

> [!WARNING]
> Never allow test user code to run in a production environment. `SeedData.InitializeAsync` is only called in the `Development` environment in the `Program` file:
>
> ```csharp
> if (builder.Environment.IsDevelopment())
> {
>     await using var scope = app.Services.CreateAsyncScope();
>     await SeedData.InitializeAsync(scope.ServiceProvider);
> }
> ```

## Roles

Role claims aren't sent back from the `manage/info` endpoint to create user claims for users of the `BlazorWasmAuth` app. Role claims are managed independently via a separate request in the `GetAuthenticationStateAsync` method of the [`CookieAuthenticationStateProvider` class (`Identity/CookieAuthenticationStateProvider.cs`)](https://github.com/dotnet/blazor-samples/blob/main/8.0/BlazorWebAssemblyStandaloneWithIdentity/BlazorWasmAuth/Identity/CookieAuthenticationStateProvider.cs) after the user is authenticated in the `Backend` project. 

In the `CookieAuthenticationStateProvider`, a roles request is made to the `/roles` endpoint of the `Backend` server API project. The response is read into a string by calling <xref:System.Net.Http.HttpContent.ReadAsStringAsync>. <xref:System.Text.Json.JsonSerializer.Deserialize%2A?displayProperty=nameWithType> deserializes the string into a custom `RoleClaim` array. Finally, the claims are added to the user's claims collection.

In the `Backend` server API's `Program` file, a [Minimal API](xref:fundamentals/minimal-apis/overview) manages the `/roles` endpoint. Claims of <xref:System.Security.Claims.ClaimsIdentity.RoleClaimType%2A> are [selected](xref:System.Linq.Enumerable.Select%2A) into an [anonymous type](/dotnet/csharp/fundamentals/types/anonymous-types) and serialized for return to the `BlazorWasmAuth` project with <xref:Microsoft.AspNetCore.Http.TypedResults.Json%2A?displayProperty=nameWithType>.

The roles endpoint requires authorization by calling <xref:Microsoft.AspNetCore.Builder.AuthorizationEndpointConventionBuilderExtensions.RequireAuthorization%2A>. If you decide not to use Minimal APIs in favor of controllers for secure server API endpoints, be sure to set the [`[Authorize]` attribute](xref:Microsoft.AspNetCore.Authorization.AuthorizeAttribute) on controllers or actions.

## Cross-domain hosting (same-site configuration)

The [sample apps](#sample-apps) are configured for hosting both apps at the same domain. If you host the `Backend` app at a different domain than the `BlazorWasmAuth` app, uncomment the code that configures the cookie (<xref:Microsoft.Extensions.DependencyInjection.IdentityServiceCollectionExtensions.ConfigureApplicationCookie%2A>) in the `Backend` app's `Program` file. The default values are:

* Same-site mode: <xref:Microsoft.AspNetCore.Http.SameSiteMode.Lax?displayProperty=nameWithType>
* Secure policy: <xref:Microsoft.AspNetCore.Http.CookieSecurePolicy.SameAsRequest?displayProperty=nameWithType>

Change the values to:

* Same-site mode: <xref:Microsoft.AspNetCore.Http.SameSiteMode.None?displayProperty=nameWithType>
* Secure policy: <xref:Microsoft.AspNetCore.Http.CookieSecurePolicy.Always?displayProperty=nameWithType>

```diff
- options.Cookie.SameSite = SameSiteMode.Lax;
- options.Cookie.SecurePolicy = CookieSecurePolicy.SameAsRequest;
+ options.Cookie.SameSite = SameSiteMode.None;
+ options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
```

For more information on same-site cookie settings, see the following resources:

* [Set-Cookie: `SameSite=<samesite-value>` (MDN documentation)](https://developer.mozilla.org/docs/Web/HTTP/Headers/Set-Cookie#samesitesamesite-value)
* [Cookies: HTTP State Management Mechanism (RFC Draft 6265, Section 4.1)](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-rfc6265bis-03#section-4.1)

## Antiforgery support

Only the logout endpoint (`/logout`) in the `Backend` app requires attention to mitigate the threat of [Cross-Site Request Forgery (CSRF)](xref:security/anti-request-forgery).

The logout endpoint checks for an empty body to prevent CSRF attacks. By requiring a body, the request must be made from JavaScript, which is the only way to access the authentication cookie. The logout endpoint can't be accessed by a form-based POST. This prevents a malicious site from logging the user out.

Furthermore, the endpoint is protected by authorization (<xref:Microsoft.AspNetCore.Builder.AuthorizationEndpointConventionBuilderExtensions.RequireAuthorization%2A>) to prevent anonymous access.

The `BlazorWasmAuth` client app is simply required to pass an empty object `{}` in the body of the request.

Outside of the logout endpoint, [antiforgery mitigation](xref:security/anti-request-forgery) is only required when submitting form data to the server encoded as `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`. Blazor manages CSRF mitigation for forms in most cases. For more information, see <xref:blazor/security/index#antiforgery-support> and <xref:blazor/forms/index#antiforgery-support>.

Requests to other server API endpoints (web API) with `application/json`-encoded content and [CORS](xref:security/cors) enabled doesn't require CSRF protection. This is why no CSRF protection is required for the `Backend` app's data processing (`/data-processing`) endpoint. The roles (`/roles`) endpoint doesn't need CSRF protection because it's a GET endpoint that doesn't modify any state.

## Troubleshoot

### Logging

To enable debug or trace logging for Blazor WebAssembly authentication, see <xref:blazor/fundamentals/logging>.

### Common errors

Check each project's configuration. Confirm that the URLs are correct:

* `Backend` project
  * `appsettings.json`
    * `BackendUrl`
    * `FrontendUrl`
  * `Backend.http`: `Backend_HostAddress`
* `BlazorWasmAuth` project: `wwwroot/appsettings.json`
  * `BackendUrl`
  * `FrontendUrl`

If the configuration appears correct:

* Analyze application logs.
* Examine the network traffic between the `BlazorWasmAuth` app and `Backend` app with the browser's developer tools. Often, an exact error message or a message with a clue to what's causing the problem is returned to the client by the backend app after making a request. Developer tools guidance is found in the following articles:

* [Google Chrome](https://developers.google.com/web/tools/chrome-devtools/network) (Google documentation)
* [Microsoft Edge](/microsoft-edge/devtools-guide-chromium/network/)
* [Mozilla Firefox](https://firefox-source-docs.mozilla.org/devtools-user/network_monitor/index.html) (Mozilla documentation)

The documentation team responds to document feedback and bugs in articles. Open an issue using the **Open a documentation issue** link at the bottom of the article. The team isn't able to provide product support. Several public support forums are available to assist with troubleshooting an app. We recommend the following:

* [Stack Overflow (tag: `blazor`)](https://stackoverflow.com/questions/tagged/blazor)
* [ASP.NET Core Slack Team](https://join.slack.com/t/aspnetcore/shared_invite/zt-1mv5487zb-EOZxJ1iqb0A0ajowEbxByQ)
* [Blazor Gitter](https://gitter.im/aspnet/Blazor)

*The preceding forums are not owned or controlled by Microsoft.*

For non-security, non-sensitive, and non-confidential reproducible framework bug reports, [open an issue with the ASP.NET Core product unit](https://github.com/dotnet/aspnetcore/issues). Don't open an issue with the product unit until you've thoroughly investigated the cause of a problem and can't resolve it on your own and with the help of the community on a public support forum. The product unit isn't able to troubleshoot individual apps that are broken due to simple misconfiguration or use cases involving third-party services. If a report is sensitive or confidential in nature or describes a potential security flaw in the product that cyberattackers may exploit, see [Reporting security issues and bugs (`dotnet/aspnetcore` GitHub repository)](https://github.com/dotnet/aspnetcore/blob/main/CONTRIBUTING.md#reporting-security-issues-and-bugs).

### Cookies and site data

Cookies and site data can persist across app updates and interfere with testing and troubleshooting. Clear the following when making app code changes, user account changes, or app configuration changes:

* User sign-in cookies
* App cookies
* Cached and stored site data

One approach to prevent lingering cookies and site data from interfering with testing and troubleshooting is to:

* Configure a browser
  * Use a browser for testing that you can configure to delete all cookie and site data each time the browser is closed.
  * Make sure that the browser is closed manually or by the IDE for any change to the app, test user, or provider configuration.
* Use a custom command to open a browser in InPrivate or Incognito mode in Visual Studio:
  * Open **Browse With** dialog box from Visual Studio's **Run** button.
  * Select the **Add** button.
  * Provide the path to your browser in the **Program** field. The following executable paths are typical installation locations for Windows 10. If your browser is installed in a different location or you aren't using Windows 10, provide the path to the browser's executable.
    * Microsoft Edge: `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe`
    * Google Chrome: `C:\Program Files (x86)\Google\Chrome\Application\chrome.exe`
    * Mozilla Firefox: `C:\Program Files\Mozilla Firefox\firefox.exe`
  * In the **Arguments** field, provide the command-line option that the browser uses to open in InPrivate or Incognito mode. Some browsers require the URL of the app.
    * Microsoft Edge: Use `-inprivate`.
    * Google Chrome: Use `--incognito --new-window {URL}`, where the `{URL}` placeholder is the URL to open (for example, `https://localhost:5001`).
    * Mozilla Firefox: Use `-private -url {URL}`, where the `{URL}` placeholder is the URL to open (for example, `https://localhost:5001`).
  * Provide a name in the **Friendly name** field. For example, `Firefox Auth Testing`.
  * Select the **OK** button.
  * To avoid having to select the browser profile for each iteration of testing with an app, set the profile as the default with the **Set as Default** button.
  * Make sure that the browser is closed by the IDE for any change to the app, test user, or provider configuration.

### App upgrades

A functioning app may fail immediately after upgrading either the .NET SDK on the development machine or changing package versions within the app. In some cases, incoherent packages may break an app when performing major upgrades. Most of these issues can be fixed by following these instructions:

1. Clear the local system's NuGet package caches by executing [`dotnet nuget locals all --clear`](/dotnet/core/tools/dotnet-nuget-locals) from a command shell.
1. Delete the project's `bin` and `obj` folders.
1. Restore and rebuild the project.
1. Delete all of the files in the deployment folder on the server prior to redeploying the app.

> [!NOTE]
> Use of package versions incompatible with the app's target framework isn't supported. For information on a package, use the [NuGet Gallery](https://www.nuget.org).

### Inspect the user's claims

To troubleshoot problems with user claims, the following `UserClaims` component can be used directly in apps or serve as the basis for further customization.

`UserClaims.razor`:

```razor
@page "/user-claims"
@using System.Security.Claims
@attribute [Authorize]

<PageTitle>User Claims</PageTitle>

<h1>User Claims</h1>

**Name**: @AuthenticatedUser?.Identity?.Name

<h2>Claims</h2>

@foreach (var claim in AuthenticatedUser?.Claims ?? Array.Empty<Claim>())
{
    <p class="claim">@(claim.Type): @claim.Value</p>
}

@code {
    [CascadingParameter]
    private Task<AuthenticationState>? AuthenticationState { get; set; }

    public ClaimsPrincipal? AuthenticatedUser { get; set; }

    protected override async Task OnInitializedAsync()
    {
        if (AuthenticationState is not null)
        {
            var state = await AuthenticationState;
            AuthenticatedUser = state.User;
        }
    }
}
```

## Additional resources

* [`AuthenticationStateProvider` service](xref:blazor/security/index#authenticationstateprovider-service)
* [Client-side SignalR cross-origin negotiation for authentication](xref:blazor/fundamentals/signalr#client-side-signalr-cross-origin-negotiation-for-authentication)
