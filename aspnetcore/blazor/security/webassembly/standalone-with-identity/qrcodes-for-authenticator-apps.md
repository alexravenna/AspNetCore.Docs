---
title: Enable QR code generation for TOTP authenticator apps in ASP.NET Core Blazor WebAssembly with ASP.NET Core Identity
author: guardrex
description: Learn how to configure an ASP.NET Core Blazor WebAssembly app with Identity for QR code generation with TOTP authenticator apps.
ms.author: wpickett
monikerRange: '>= aspnetcore-8.0'
ms.date: 11/21/2024
uid: blazor/security/webassembly/standalone-with-identity/qrcodes-for-authenticator-apps
---
# Enable QR code generation for TOTP authenticator apps in ASP.NET Core Blazor WebAssembly with ASP.NET Core Identity

[!INCLUDE[](~/includes/not-latest-version-without-not-supported-content.md)]

This article explains how to configure an ASP.NET Core Blazor WebAssembly app with Identity for two-factor authentication (2FA) with QR codes generated by Time-based One-time Password Algorithm (TOTP) authenticator apps.

For an introduction to 2FA with TOTP authenticator apps, see <xref:security/authentication/identity-enable-qrcodes>.

> [!WARNING]
> TOTP codes should be kept secret because they can be used to authenticate multiple times before they expire.

## Namespaces and article code examples

The namespaces used by the examples in this article are:

* `Backend` for the backend server web API project, described as the "server project" in this article.
* `BlazorWasmAuth` for the front-end client standalone Blazor WebAssembly app, described as the "client project" in this article.

These namespaces correspond to the projects in the `BlazorWebAssemblyStandaloneWithIdentity` sample solution in the [`dotnet/blazor-samples` GitHub repository](https://github.com/dotnet/blazor-samples). For more information, see <xref:blazor/security/webassembly/standalone-with-identity/index#sample-apps>.

If you aren't using the `BlazorWebAssemblyStandaloneWithIdentity` sample, change the namespaces in the code examples to use the namespaces of your projects.

All of the changes to the solution covered by this article take place in the `BlazorWasmAuth` project of the `BlazorWebAssemblyStandaloneWithIdentity` solution.

In article examples, code lines are split to reduce horizontal scrolling. These breaks don't affect execution but can be removed when pasting into your project.

## Optional account confirmation and password recovery

Although apps that implement 2FA usually adopt account confirmation and password recovery features, 2FA doesn't require it. The guidance in this article can be followed to implement 2FA without following the guidance in <xref:blazor/security/webassembly/standalone-with-identity/account-confirmation-and-password-recovery>.

## Add a QR code library to the app

A QR code generated by the app to set up 2FA with an TOTP authenticator app must be generated by a QR code library.

The guidance in this article uses [`manuelbl/QrCodeGenerator`](https://github.com/manuelbl/QrCodeGenerator), but you can use any QR code generation library.

Add a package reference to the client project for the [`Net.Codecrete.QrCodeGenerator`](https://www.nuget.org/packages/Net.Codecrete.QrCodeGenerator) NuGet package.

[!INCLUDE[](~/includes/package-reference.md)]

## Set the TOTP organization name

Set the site name in the app settings file of the client project. Use a meaningful site name that users can identify easily in their authenticator app. Developers usually set a site name that matches the company's name. We recommend limiting the site name length to 30 characters or less to allow the site name to display on narrow mobile device screens.

In the following example, the company name is `Weyland-Yutani Corporation` (&copy;1986 20th Century Studios [*Aliens*](https://www.20thcenturystudios.com/movies/aliens)).

Added to `wwwroot/appsettings.json`:

```json
"TotpOrganizationName": "Weyland-Yutani Corporation"
```

The app settings file after the TOTP organization name configuration is added:

```json
{
  "BackendUrl": "https://localhost:7211",
  "FrontendUrl": "https://localhost:7171",
  "TotpOrganizationName": "Weyland-Yutani Corporation"
}
```

## Add model classes

Add the following `LoginResponse` class to the `Models` folder. This class is populated for requests to the `/login` endpoint of <xref:Microsoft.AspNetCore.Routing.IdentityApiEndpointRouteBuilderExtensions.MapIdentityApi%2A> in the server app.

`Identity/Models/LoginResponse.cs`:

```csharp
namespace BlazorWasmAuth.Identity.Models;

public class LoginResponse
{
    public string? Type { get; set; }
    public string? Title { get; set; }
    public int Status { get; set; }
    public string? Detail { get; set; }
}
```

Add the following `TwoFactorRequest` class to the `Models` folder. This class is populated for requests to the `/manage/2fa` endpoint of <xref:Microsoft.AspNetCore.Routing.IdentityApiEndpointRouteBuilderExtensions.MapIdentityApi%2A> in the server app.

`Identity/Models/TwoFactorRequest.cs`:

```csharp
namespace BlazorWasmAuth.Identity.Models;

public class TwoFactorRequest
{
    public bool? Enable { get; set; }
    public string? TwoFactorCode { get; set; }
    public bool? ResetSharedKey { get; set; }
    public bool? ResetRecoveryCodes { get; set; }
    public bool? ForgetMachine {  get; set; }
}
```

Add the following `TwoFactorResponse` class to the `Models` folder. This class is populated by the response to a 2FA request made to the `/manage/2fa` endpoint of <xref:Microsoft.AspNetCore.Routing.IdentityApiEndpointRouteBuilderExtensions.MapIdentityApi%2A> in the server app.

`Identity/Models/TwoFactorResponse.cs`:

```csharp
namespace BlazorWasmAuth.Identity.Models;

public class TwoFactorResponse
{
    public string SharedKey { get; set; } = string.Empty;
    public int RecoveryCodesLeft { get; set; } = 0;
    public string[] RecoveryCodes { get; set; } = [];
    public bool IsTwoFactorEnabled { get; set; }
    public bool IsMachineRemembered { get; set; }
    public string[] ErrorList { get; set; } = [];
}
```

## `IAccountManagement` interface

Add the following class signatures to the `IAccountManagement` interface. The class signatures represent methods added to the cookie authentication state provider for the following client requests:

* Log in with a 2FA TOTP code (`/login` endpoint): `LoginTwoFactorCodeAsync`
* Log in with a 2FA recovery code (`/login` endpoint): `LoginTwoFactorRecoveryCodeAsync`
* Make a 2FA management request (`/manage/2fa` endpoint): `TwoFactorRequestAsync`

`Identity/IAccountManagement.cs` (paste the following code at the bottom of the file):

```csharp
public Task<FormResult> LoginTwoFactorCodeAsync(
    string email, 
    string password, 
    string twoFactorCode);

public Task<FormResult> LoginTwoFactorRecoveryCodeAsync(
    string email, 
    string password, 
    string twoFactorRecoveryCode);

public Task<TwoFactorResponse> TwoFactorRequestAsync(
    TwoFactorRequest twoFactorRequest);
```

## Update the cookie authentication state provider

Update the `CookieAuthenticationStateProvider` with features to add the following features:

* Authenticate users with either a TOTP authenticator app code or a recovery code.
* Manage 2FA in the app.

At the top of the `CookieAuthenticationStateProvider.cs` file, add a `using` statement for <xref:System.Text.Json.Serialization?displayProperty=fullName>:

```csharp
using System.Text.Json.Serialization;
```

In the <xref:System.Text.Json.JsonSerializerOptions>, add the <xref:System.Text.Json.JsonSerializerOptions.DefaultIgnoreCondition> option set to <xref:System.Text.Json.Serialization.JsonIgnoreCondition.WhenWritingNull?displayProperty=nameWithType>, which avoids serializing null properties:

```diff
private readonly JsonSerializerOptions jsonSerializerOptions =
    new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
+       DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    };
```

The `LoginAsync` method is updated with the following logic:

* Attempt a normal login at the `/login` endpoint with an email address and password.
* If the server responds with a success status code, the method returns a `FormResult` with the `Succeeded` property set to `true`.
* If the server responds with the *401 - Unauthorized* status code and a detail code of "`RequiresTwoFactor`," a `FormResult` is returned with `Succeeded` set to `false` and the `RequiresTwoFactor` detail in the error list.

In `Identity/CookieAuthenticationStateProvider.cs`, replace the `LoginAsync` method with the following code:

```csharp
public async Task<FormResult> LoginAsync(string email, string password)
{
    try
    {
        using var result = await httpClient.PostAsJsonAsync(
            "login?useCookies=true", new
            {
                email,
                password
            });

        if (result.IsSuccessStatusCode)
        {
            NotifyAuthenticationStateChanged(GetAuthenticationStateAsync());

            return new FormResult { Succeeded = true };
        }
        else if (result.StatusCode == HttpStatusCode.Unauthorized)
        {
            using var responseJson = await result.Content.ReadAsStringAsync();
            var response = JsonSerializer.Deserialize<LoginResponse>(
                responseJson, jsonSerializerOptions);

            if (response?.Detail == "RequiresTwoFactor")
            {
                return new FormResult
                {
                    Succeeded = false,
                    ErrorList = [ "RequiresTwoFactor" ]
                };
            }
        }
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "App error");
    }

    return new FormResult
    {
        Succeeded = false,
        ErrorList = [ "Invalid email and/or password." ]
    };
}
```

A `LoginTwoFactorCodeAsync` method is added, which sends a request to the `/login` endpoint with a 2FA TOTP code (`twoFactorCode`). The method processes the response in a similar fashion to a normal, non-2FA login request.

Add the following method and class to `Identity/CookieAuthenticationStateProvider.cs` (paste the following code at the bottom of the class file):

```csharp
public async Task<FormResult> LoginTwoFactorCodeAsync(
    string email, string password, string twoFactorCode)
{
    try
    {
        using var result = await httpClient.PostAsJsonAsync(
            "login?useCookies=true", new
            {
                email,
                password,
                twoFactorCode
            });

        if (result.IsSuccessStatusCode)
        {
            NotifyAuthenticationStateChanged(GetAuthenticationStateAsync());

            return new FormResult { Succeeded = true };
        }
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "App error");
    }

    return new FormResult
    {
        Succeeded = false,
        ErrorList = [ "Invalid two-factor code." ]
    };
}
```

A `LoginTwoFactorRecoveryCodeAsync` method is added, which sends a request to the `/login` endpoint with a 2FA recovery code (`twoFactorRecoveryCode`). The method processes the response in a similar fashion to a normal, non-2FA login request.

Add the following method and class to `Identity/CookieAuthenticationStateProvider.cs` (paste the following code at the bottom of the class file):

```csharp
public async Task<FormResult> LoginTwoFactorRecoveryCodeAsync(string email, 
    string password, string twoFactorRecoveryCode)
{
    try
    {
        using var result = await httpClient.PostAsJsonAsync(
            "login?useCookies=true", new
            {
                email,
                password,
                twoFactorRecoveryCode
            });

        if (result.IsSuccessStatusCode)
        {
            NotifyAuthenticationStateChanged(GetAuthenticationStateAsync());

            return new FormResult { Succeeded = true };
        }
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "App error");
    }

    return new FormResult
    {
        Succeeded = false,
        ErrorList = [ "Invalid recovery code." ]
    };
}
```

A `TwoFactorRequestAsync` method is added, which is used to manage 2FA for the user:

* Reset the shared 2FA key when `TwoFactorRequest.ResetSharedKey` is `true`. Resetting the shared key implicitly disables 2FA. This forces the user to prove that they can provide a valid TOTP code from their authenticator app to enable 2FA after receiving a new shared key.
* Reset the user's recovery codes when `TwoFactorRequest.ResetRecoveryCodes` is `true`.
* Forget the machine when `TwoFactorRequest.ForgetMachine` is `true`, which means that a new 2FA TOTP code is required on the next login attempt.
* Enable 2FA using a TOTP code from a TOTP authenticator app when `TwoFactorRequest.Enable` is `true` and `TwoFactorRequest.TwoFactorCode` has a valid TOTP value.
* Obtain 2FA status with an empty request when all of `TwoFactorRequest`'s properties are `null`.

Add the following `TwoFactorRequestAsync` method to `Identity/CookieAuthenticationStateProvider.cs` (paste the following code at the bottom of the class file):

```csharp
public async Task<TwoFactorResponse> TwoFactorRequestAsync(TwoFactorRequest twoFactorRequest)
{
    string[] defaultDetail = 
        [ "An unknown error prevented two-factor authentication." ];

    using var response = await httpClient.PostAsJsonAsync("manage/2fa", twoFactorRequest, 
        jsonSerializerOptions);

    // successful?
    if (response.IsSuccessStatusCode)
    {
        return await response.Content
            .ReadFromJsonAsync<TwoFactorResponse>() ??
            new()
            { 
                ErrorList = [ "There was an error processing the request." ]
            };
    }

    // body should contain details about why it failed
    var details = await response.Content.ReadAsStringAsync();
    var problemDetails = JsonDocument.Parse(details);
    var errors = new List<string>();
    var errorList = problemDetails.RootElement.GetProperty("errors");

    foreach (var errorEntry in errorList.EnumerateObject())
    {
        if (errorEntry.Value.ValueKind == JsonValueKind.String)
        {
            errors.Add(errorEntry.Value.GetString()!);
        }
        else if (errorEntry.Value.ValueKind == JsonValueKind.Array)
        {
            errors.AddRange(
                errorEntry.Value.EnumerateArray().Select(
                    e => e.GetString() ?? string.Empty)
                .Where(e => !string.IsNullOrEmpty(e)));
        }
    }

    // return the error list
    return new TwoFactorResponse
    {
        ErrorList = problemDetails == null ? defaultDetail : [.. errors]
    };
}
```

## Replace `Login` component

Replace the `Login` component. The following version of the `Login` component:

* Accepts a user's email address and password for an initial login attempt.
* If login is successful (2FA is disabled), the component notifies the user that they're authenticated.
* If the login attempt results in a response indicating that 2FA is required, a 2FA input element appears to receive either a 2FA TOTP code from an authenticator app or a recovery code. Depending on which code the user enters, login is attempted again by calling either `LoginTwoFactorCodeAsync` for a TOTP code or `LoginTwoFactorRecoveryCodeAsync` for a recovery code.

`Components/Identity/Login.razor`:

```razor
@page "/login"
@using System.ComponentModel.DataAnnotations
@using BlazorWasmAuth.Identity
@using BlazorWasmAuth.Identity.Models
@inject IAccountManagement Acct
@inject ILogger<Login> Logger
@inject NavigationManager Navigation

<PageTitle>Login</PageTitle>

<h1>Login</h1>

<AuthorizeView>
    <Authorized>
        <div class="alert alert-success">
            You're logged in as @context.User.Identity?.Name.
        </div>
    </Authorized>
    <NotAuthorized>
        @foreach (var error in formResult.ErrorList)
        {
            <div class="alert alert-danger">@error</div>
        }
        <div class="row">
            <div class="col">
                <section>
                    <EditForm Model="Input" method="post" OnValidSubmit="LoginUser" 
                            FormName="login" Context="editform_context">
                        <DataAnnotationsValidator />
                        <h2>Use a local account to log in.</h2>
                        <hr />
                        <div style="display:@(requiresTwoFactor ? "none" : "block")">
                            <div class="form-floating mb-3">
                                <InputText @bind-Value="Input.Email" 
                                    id="Input.Email" 
                                    class="form-control" 
                                    autocomplete="username" 
                                    aria-required="true" 
                                    placeholder="name@example.com" />
                                <label for="Input.Email" class="form-label">
                                    Email
                                </label>
                                <ValidationMessage For="() => Input.Email" 
                                    class="text-danger" />
                            </div>
                            <div class="form-floating mb-3">
                                <InputText type="password" 
                                    @bind-Value="Input.Password" 
                                    id="Input.Password" 
                                    class="form-control" 
                                    autocomplete="current-password" 
                                    aria-required="true" 
                                    placeholder="password" />
                                <label for="Input.Password" class="form-label">
                                    Password
                                </label>
                                <ValidationMessage For="() => Input.Password" 
                                    class="text-danger" />
                            </div>
                        </div>
                        <div style="display:@(requiresTwoFactor ? "block" : "none")">
                            <div class="form-floating mb-3">
                                <InputText @bind-Value="Input.TwoFactorCodeOrRecoveryCode" 
                                    id="Input.TwoFactorCodeOrRecoveryCode" 
                                    class="form-control" 
                                    autocomplete="off" 
                                    placeholder="###### or #####-#####" />
                                <label for="Input.TwoFactorCodeOrRecoveryCode" class="form-label">
                                    Two-factor Code or Recovery Code
                                </label>
                                <ValidationMessage For="() => Input.TwoFactorCodeOrRecoveryCode" 
                                    class="text-danger" />
                            </div>
                        </div>
                        <div>
                            <button type="submit" class="w-100 btn btn-lg btn-primary">
                                Log in
                            </button>
                        </div>
                        <div class="mt-3">
                            <p>
                                <a href="forgot-password">Forgot password</a>
                            </p>
                            <p>
                                <a href="register">Register as a new user</a>
                            </p>
                        </div>
                    </EditForm>
                </section>
            </div>
        </div>
    </NotAuthorized>
</AuthorizeView>

@code {
    private FormResult formResult = new();
    private bool requiresTwoFactor;

    [SupplyParameterFromForm]
    private InputModel Input { get; set; } = new();

    [SupplyParameterFromQuery]
    private string? ReturnUrl { get; set; }

    public async Task LoginUser()
    {
        if (requiresTwoFactor)
        {
            if (!string.IsNullOrEmpty(Input.TwoFactorCodeOrRecoveryCode))
            {
                // The [RegularExpression] data annotation ensures that the input 
                // is either a six-digit authenticator code (######) or an 
                // eleven-character alphanumeric recovery code (#####-#####)
                if (Input.TwoFactorCodeOrRecoveryCode.Length == 6)
                {
                    formResult = await Acct.LoginTwoFactorCodeAsync(
                        Input.Email, Input.Password, 
                        Input.TwoFactorCodeOrRecoveryCode);
                }
                else
                {
                    formResult = await Acct.LoginTwoFactorRecoveryCodeAsync(
                        Input.Email, Input.Password, 
                        Input.TwoFactorCodeOrRecoveryCode);

                    if (formResult.Succeeded)
                    {
                        var twoFactorResponse = await Acct.TwoFactorRequestAsync(new());
                    }
                }
            }
            else
            {
                formResult = 
                    new FormResult
                    {
                        Succeeded = false,
                        ErrorList = [ "Invalid two-factor code." ]
                    };
            }
        }
        else
        {
            formResult = await Acct.LoginAsync(Input.Email, Input.Password);
            requiresTwoFactor = formResult.ErrorList.Contains("RequiresTwoFactor");
            Input.TwoFactorCodeOrRecoveryCode = string.Empty;

            if (requiresTwoFactor)
            {
                formResult.ErrorList = [];
            }
        }

        if (formResult.Succeeded && !string.IsNullOrEmpty(ReturnUrl))
        {
            Navigation.NavigateTo(ReturnUrl);
        }
    }

    private sealed class InputModel
    {
        [Required]
        [EmailAddress]
        [Display(Name = "Email")]
        public string Email { get; set; } = string.Empty;

        [Required]
        [DataType(DataType.Password)]
        [Display(Name = "Password")]
        public string Password { get; set; } = string.Empty;

        [RegularExpression(@"^([0-9]{6})|([A-Z0-9]{5}[-]{1}[A-Z0-9]{5})$", 
            ErrorMessage = "Must be a six-digit authenticator code (######) or " +
            "eleven-character alphanumeric recovery code (#####-#####, dash " +
            "required)")]
        [Display(Name = "Two-factor Code or Recovery Code")]
        public string TwoFactorCodeOrRecoveryCode { get; set; } = string.Empty;
    }
}
```

Using the preceding component, the user is remembered after a successful login with a valid TOTP code from an authenticator app. If you want to always require a TOTP code for login and not remember the machine, call the `TwoFactorRequestAsync` method with `TwoFactorRequest.ForgetMachine` set to `true` immediately after a successful two-factor login:

```diff
if (Input.TwoFactorCodeOrRecoveryCode.Length == 6)
{
    formResult = await Acct.LoginTwoFactorCodeAsync(Input.Email, Input.Password, 
        Input.TwoFactorCodeOrRecoveryCode);

+    if (formResult.Succeeded)
+    {
+        var forgetMachine = 
+            await Acct.TwoFactorRequestAsync(new() { ForgetMachine = true });
+    }
}
```

## Add a component to display recovery codes

Add the following `ShowRecoveryCodes` component to the app to display recovery codes to the user.

`Components/Identity/ShowRecoveryCodes.razor`:

```razor
<h3>Recovery codes</h3>

<div class="alert alert-warning" role="alert">
    <p>
        <strong>Put these codes in a safe place.</strong>
    </p>
    <p>
        If you lose your device and don't have an unused 
        recovery code, you can't access your account.
    </p>
</div>
<div class="row">
    <div class="col-md-12">
        @foreach (var recoveryCode in RecoveryCodes)
        {
            <div>
                <code class="recovery-code">@recoveryCode</code>
            </div>
        }
    </div>
</div>

@code {
    [Parameter]
    public string[] RecoveryCodes { get; set; } = [];
}
```

## Manage 2FA page

Add the following `Manage2fa` component to the app to manage 2FA for users.

If 2FA isn't enabled, the component loads a form with a QR code to enable 2FA with a TOTP authenticator app. The user adds the app to their authenticator app and then verifies the authenticator app and enables 2FA by providing a TOTP code from the authenticator app.

If 2FA is enabled, buttons appear to disable 2FA and regenerate recovery codes.

`Components/Identity/Manage2fa.razor`:

```razor
@page "/manage-2fa"
@using System.ComponentModel.DataAnnotations
@using System.Globalization
@using System.Text
@using System.Text.Encodings.Web
@using Net.Codecrete.QrCodeGenerator
@using BlazorWasmAuth.Identity
@using BlazorWasmAuth.Identity.Models
@attribute [Authorize]
@inject IAccountManagement Acct
@inject IAuthorizationService AuthorizationService
@inject IConfiguration Config
@inject ILogger<Manage2fa> Logger

<PageTitle>Manage 2FA</PageTitle>

<h1>Manage Two-factor Authentication</h1>
<hr />
<div class="row">
    <div class="col">
        @if (loading)
        {
            <p>Loading ...</p>
        }
        else
        {
            @if (twoFactorResponse is not null)
            {
                @foreach (var error in twoFactorResponse.ErrorList)
                {
                    <div class="alert alert-danger">@error</div>
                }
                @if (twoFactorResponse.IsTwoFactorEnabled)
                {
                    <div class="alert alert-success" role="alert">
                        Two-factor authentication is enabled for your account.
                    </div>

                    <div class="m-1">
                        <button @onclick="Disable2FA" class="btn btn-lg btn-primary">
                            Disable 2FA
                        </button>
                    </div>

                    @if (twoFactorResponse.RecoveryCodes is null)
                    {
                        <div class="m-1">
                            Recovery Codes Remaining: 
                            @twoFactorResponse.RecoveryCodesLeft
                        </div>
                        <div class="m-1">
                            <button @onclick="GenerateNewCodes" 
                                    class="btn btn-lg btn-primary">
                                Generate New Recovery Codes
                            </button>
                        </div>
                    }
                    else
                    {
                        <ShowRecoveryCodes 
                            RecoveryCodes="twoFactorResponse.RecoveryCodes" />
                    }
                }
                else
                {
                    <h3>Configure authenticator app</h3>
                    <div>
                        <p>To use an authenticator app:</p>
                        <ol class="list">
                            <li>
                                <p>
                                    Download a two-factor authenticator app, such 
                                    as either of the following:
                                    <ul>
                                        <li>
                                            Microsoft Authenticator for
                                            <a href="https://go.microsoft.com/fwlink/?Linkid=825072">
                                                Android
                                            </a> and
                                            <a href="https://go.microsoft.com/fwlink/?Linkid=825073">
                                                iOS
                                            </a>
                                        </li>
                                        <li>
                                            Google Authenticator for
                                            <a href="https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2">
                                                Android
                                            </a> and
                                            <a href="https://itunes.apple.com/us/app/google-authenticator/id388497605?mt=8">
                                                iOS
                                            </a>
                                        </li>
                                    </ul>
                                </p>
                            </li>
                            <li>
                                <p>
                                    Scan the QR Code or enter this key 
                                    <kbd>@twoFactorResponse.SharedKey</kbd> into your 
                                    two-factor authenticator app. Spaces and casing 
                                    don't matter.
                                </p>
                                <div>
                                    <svg xmlns="http://www.w3.org/2000/svg" height="300" 
                                            width="300" stroke="none" version="1.1" 
                                            viewBox="0 0 50 50">
                                        <rect width="300" height="300" fill="#ffffff" />
                                        <path d="@svgGraphicsPath" fill="#000000" />
                                    </svg>
                                </div>
                            </li>
                            <li>
                                <p>
                                    After you have scanned the QR code or input the 
                                    key above, your two-factor authenticator app 
                                    will provide you with a unique two-factor code. 
                                    Enter the code in the confirmation box below.
                                </p>
                                <div class="row">
                                    <div class="col-xl-6">
                                        <EditForm Model="Input" 
                                                FormName="send-code" 
                                                OnValidSubmit="OnValidSubmitAsync" 
                                                method="post">
                                            <DataAnnotationsValidator />
                                            <div class="form-floating mb-3">
                                                <InputText 
                                                    @bind-Value="Input.Code" 
                                                    id="Input.Code" 
                                                    class="form-control" 
                                                    autocomplete="off" 
                                                    placeholder="Enter the code" />
                                                <label for="Input.Code" 
                                                        class="control-label form-label">
                                                    Verification Code
                                                </label>
                                                <ValidationMessage 
                                                    For="() => Input.Code" 
                                                    class="text-danger" />
                                            </div>
                                            <button type="submit" 
                                                    class="w-100 btn btn-lg btn-primary">
                                                Verify
                                            </button>
                                        </EditForm>
                                    </div>
                                </div>
                            </li>
                        </ol>
                    </div>
                }
            }
        }
    </div>
</div>

@code {
    private TwoFactorResponse twoFactorResponse = new();
    private bool loading = true;
    private string? svgGraphicsPath;

    [SupplyParameterFromForm]
    private InputModel Input { get; set; } = new();

    [CascadingParameter]
    private Task<AuthenticationState>? authenticationState { get; set; }

    protected override async Task OnInitializedAsync()
    {
        twoFactorResponse = await Acct.TwoFactorRequestAsync(new());
        svgGraphicsPath = await GetQrCode(twoFactorResponse.SharedKey);
        loading = false;
    }

    private async Task<string> GetQrCode(string sharedKey)
    {
        if (authenticationState is not null && !string.IsNullOrEmpty(sharedKey))
        {
            var authState = await authenticationState;
            var email = authState?.User?.Identity?.Name!;
            var uri = string.Format(
                CultureInfo.InvariantCulture,
                "otpauth://totp/{0}:{1}?secret={2}&issuer={0}&digits=6",
                UrlEncoder.Default.Encode(Config["TotpOrganizationName"]!),
                email,
                twoFactorResponse.SharedKey);
            var qr = QrCode.EncodeText(uri, QrCode.Ecc.Medium);

            return qr.ToGraphicsPath();
        }

        return string.Empty;
    }

    private async Task Disable2FA()
    {
        await Acct.TwoFactorRequestAsync(new() { ForgetMachine = true });
        twoFactorResponse = 
            await Acct.TwoFactorRequestAsync(new() { ResetSharedKey = true });
        svgGraphicsPath = await GetQrCode(twoFactorResponse.SharedKey);
    }

    private async Task GenerateNewCodes()
    {
        twoFactorResponse = 
            await Acct.TwoFactorRequestAsync(new() { ResetRecoveryCodes = true });
    }

    private async Task OnValidSubmitAsync()
    {
        twoFactorResponse = await Acct.TwoFactorRequestAsync(
            new() 
            { 
                Enable = true, 
                TwoFactorCode = Input.Code 
            });
        Input.Code = string.Empty;

        // When 2FA is first enabled, recovery codes are returned.
        // However, subsequently disabling and re-enabling 2FA
        // leaves the existing codes in place and doesn't generate
        // a new set of recovery codes. The following code ensures
        // that a new set of recovery codes is generated each
        // time 2FA is enabled.
        if (twoFactorResponse.RecoveryCodes is null || 
            twoFactorResponse.RecoveryCodes.Length == 0)
        {
            await GenerateNewCodes();
        }
    }

    private sealed class InputModel
    {
        [Required]
        [RegularExpression(@"^([0-9]{6})$", 
            ErrorMessage = "Must be a six-digit authenticator code (######)")]
        [DataType(DataType.Text)]
        [Display(Name = "Verification Code")]
        public string Code { get; set; } = string.Empty;
    }
}
```

## Link to the Manage 2FA page

Add a link to the navigation menu for users to reach the `Manage2fa` component page.

In the `<Authorized>` content of the `<AuthorizeView>` in `Components/Layout/NavMenu.razor`, add the following markup:

```diff
<AuthorizeView>
    <Authorized>

        ...

+       <div class="nav-item px-3">
+           <NavLink class="nav-link" href="manage-2fa">
+               <span class="bi bi-key" aria-hidden="true"></span> Manage 2FA
+           </NavLink>
+       </div>

        ...

    </Authorized>
</AuthorizeView>
```

## Failures due to TOTP time skew

TOTP authentication depends on accurate time keeping on the TOTP authenticator app device and the app's host. TOTP tokens are only valid for 30 seconds. If logins are failing due to rejected TOTP codes, confirm accurate time is maintained, preferably synchronized to an accurate NTP service.

## Additional resources

* [`nimiq/qr-creator`](https://github.com/nimiq/qr-creator)
* <xref:Microsoft.AspNetCore.Routing.IdentityApiEndpointRouteBuilderExtensions.MapIdentityApi%2A>
