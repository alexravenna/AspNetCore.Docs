---
title: Enable QR code generation for TOTP authenticator apps in ASP.NET Core
author: wadepickett
description: Discover how to enable QR code generation for TOTP authenticator apps that work with ASP.NET Core two-factor authentication.
monikerRange: '>= aspnetcore-2.1'
ms.author: wpickett
ms.date: 08/14/2018
uid: security/authentication/identity-enable-qrcodes
---

# Enable QR code generation for TOTP authenticator apps in ASP.NET Core

ASP.NET Core ships with support for authenticator applications for individual authentication. Two factor authentication (2FA) authenticator apps, using a Time-based One-time Password Algorithm (TOTP), are the industry recommended approach for 2FA. 2FA using TOTP is preferred to SMS 2FA. An authenticator app provides a 6 to 8 digit code which users must enter after confirming their username and password. Typically an authenticator app is installed on a smartphone.

> [!WARNING]
> An ASP.NET Core TOTP code should be kept secret because it can be used to authenticate successfully multiple times before it expires.

:::moniker range=">= aspnetcore-8.0"

The ASP.NET Core web app templates support authenticators but don't provide support for QR code generation. QR code generators ease the setup of 2FA. This document provides guidance for Razor Pages and MVC apps on how to add [QR code](https://wikipedia.org/wiki/QR_code) generation to the 2FA configuration page. For guidance that applies to Blazor Web Apps, see <xref:blazor/security/qrcodes-for-authenticator-apps>. For guidance that applies to Blazor WebAssembly apps, see <xref:blazor/security/webassembly/standalone-with-identity/qrcodes-for-authenticator-apps>.

:::moniker-end

:::moniker range="< aspnetcore-8.0"

The ASP.NET Core web app templates support authenticators but don't provide support for QR code generation. QR code generators ease the setup of 2FA. This document guides you through adding [QR code](https://wikipedia.org/wiki/QR_code) generation to the 2FA configuration page.

:::moniker-end

Two-factor authentication does not happen using an external authentication provider, such as [Google](xref:security/authentication/google-logins) or [Facebook](xref:security/authentication/facebook-logins). External logins are protected by whatever mechanism the external login provider provides. Consider, for example, the [Microsoft](xref:security/authentication/microsoft-logins) authentication provider requires a hardware key or another 2FA approach. If the default templates required 2FA for both the web app and the external authentication provider, then users would be required to satisfy two 2FA approaches. Requiring two 2FA approaches deviates from established security practices, which typically rely on a single, strong 2FA method for authentication.

## Adding QR codes to the 2FA configuration page

These instructions use `qrcode.js` from the https://davidshimjs.github.io/qrcodejs/ repo.

* Download the [`qrcode.js` JavaScript library](https://davidshimjs.github.io/qrcodejs/) to the `wwwroot\lib` folder in your project.
* Follow the instructions in [Scaffold Identity](xref:security/authentication/scaffold-identity) to generate `/Areas/Identity/Pages/Account/Manage/EnableAuthenticator.cshtml`.
* In `/Areas/Identity/Pages/Account/Manage/EnableAuthenticator.cshtml`, locate the `Scripts` section at the end of the file:

```cshtml
@section Scripts {
    @await Html.PartialAsync("_ValidationScriptsPartial")
}
```
* Create a new JavaScript file called `qr.js` in `wwwroot/js` and add the following code to generate the QR Code:

```javascript
window.addEventListener("load", () => {
  const uri = document.getElementById("qrCodeData").getAttribute('data-url');
  new QRCode(document.getElementById("qrCode"),
    {
      text: uri,
      width: 150,
      height: 150
    });
});
```

* Update the `Scripts` section to add a reference to the `qrcode.js` library previously downloaded.
* Add the `qr.js` file with the call to generate the QR code:

```cshtml
@section Scripts {
    @await Html.PartialAsync("_ValidationScriptsPartial")

    <script type="text/javascript" src="~/lib/qrcode.js"></script>
    <script type="text/javascript" src="~/js/qr.js"></script>
}
```

* Delete the paragraph which links you to these instructions.

Run your app and ensure that you can scan the QR code and validate the code the authenticator proves.

## Change the site name in the QR code

The site name in the QR code is taken from the project name you choose when initially creating your project. You can change it by looking for the `GenerateQrCodeUri(string email, string unformattedKey)` method in the `/Areas/Identity/Pages/Account/Manage/EnableAuthenticator.cshtml.cs`.

The default code from the template looks as follows:

```csharp
private string GenerateQrCodeUri(string email, string unformattedKey)
{
    return string.Format(
        AuthenticatorUriFormat,
        _urlEncoder.Encode("Razor Pages"),
        _urlEncoder.Encode(email),
        unformattedKey);
}
```

The second parameter in the call to `string.Format` is your site name, taken from your solution name. It can be changed to any value, but it must always be URL encoded.

## Using a different QR Code library

You can replace the QR Code library with your preferred library. The HTML contains a `qrCode` element into which you can place a QR Code by whatever mechanism your library provides.

The correctly formatted URL for the QR Code is available in the:

* `AuthenticatorUri` property of the model.
* `data-url` property in the `qrCodeData` element.

## TOTP client and server time skew

TOTP (Time-based One-Time Password) authentication depends on both the server and authenticator device having an accurate time. Tokens only last for 30 seconds. If TOTP 2FA logins are failing, check that the server time is accurate, and preferably synchronized to an accurate NTP service.
