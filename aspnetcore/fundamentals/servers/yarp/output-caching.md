---
uid: fundamentals/servers/yarp/output-caching
title: YARP Output Caching
description: YARP Output Caching
author: wadepickett
ms.author: wpickett
ms.date: 2/6/2025
ms.topic: article
content_well_notification: AI-contribution
ai-usage: ai-assisted
---

# YARP Output Caching

## Introduction
The reverse proxy can be used to cache proxied responses and serve requests before they are proxied to the destination servers. This can reduce load on the destination servers, add a layer of protection, and ensure consistent policies are implemented across your applications.

> This feature is only available when using .NET 7 or later

## Defaults

No output caching is performed unless enabled in the route or application configuration.

## Configuration
Output Cache policies can be specified per route via [RouteConfig.OutputCachePolicy](xref:Yarp.ReverseProxy.Configuration.RouteConfig) and can be bound from the `Routes` sections of the config file. As with other route properties, this can be modified and reloaded without restarting the proxy. Policy names are case insensitive.

Example:
```json
{
  "ReverseProxy": {
    "Routes": {
      "route1" : {
        "ClusterId": "cluster1",
        "OutputCachePolicy": "customPolicy",
        "Match": {
          "Hosts": [ "localhost" ]
        }
      }
    },
    "Clusters": {
      "cluster1": {
        "Destinations": {
          "cluster1/destination1": {
            "Address": "https://localhost:10001/"
          }
        }
      }
    }
  }
}
```

[Output cache policies](/aspnet/core/performance/caching/output) are an ASP.NET Core concept that the proxy utilizes. The proxy provides the above configuration to specify a policy per route and the rest is handled by existing ASP.NET Core output caching middleware.

Output cache policies can be configured in Program.cs as follows:
```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("customPolicy", builder => builder.Expire(TimeSpan.FromSeconds(20)));
});
```

Then add the output caching middleware:

```csharp
var app = builder.Build();

app.UseOutputCache();

app.MapReverseProxy();
```

See the [Output Caching](/aspnet/core/performance/caching/output) docs for setting up your preferred kind of output caching.
