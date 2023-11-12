---
title: .NET Aspire service discovery
description: Understand essential service discovery concepts in .NET Aspire.
ms.date: 11/11/2023
ms.topic: quickstart
ms.prod: dotnet
---

# .NET Aspire service discovery

In this article, you learn how service discovery works within a .NET Aspire app. .NET Aspire includes functionality for configuring service discovery at development and testing time. Service discovery functionality works by providing configuration in the format expected by the _configuration-based endpoint resolver_ from the .NET Aspire AppHost project to the individual service projects added to the application model. For more information, see [Service discovery in .NET](/dotnet/core/extensions/service-discovery.md).

## Implicit service discovery by reference

Configuration for service discovery is only added for services that are referenced by a given project. For example, consider the following AppHost program:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var catalog = builder.AddProject<Projects.CatalogService>("catalog");
var basket = builder.AddProject<Projects.BasketService>("basket");

var frontend = builder.AddProject<Projects.MyFrontend>("frontend")
                      .WithReference(basket)
                      .WithReference(catalog);
```

In the preceding example, the _frontend_ project references the _catalog_ project and the _basket_ project. The two `WithReference` calls instruct the .NET Aspire application to pass service discovery information for the referenced projects (_catalog_, and _basket_) into the _frontend_ project.

## Named endpoints

Some services expose multiple, named endpoints. Named endpoints can be resolved by specifying the endpoint name in the host portion of the HTTP request URI, following the format `http://_endpointName.serviceName`. For example, if a service named "basket" exposes an endpoint named "dashboard", then the URI `http://_dashboard.basket` can be used to specify this endpoint, for example:

```csharp
builder.Services.AddHttpClient<BasketServiceClient>(
    static client => client.BaseAddress = new("http://basket"));

builder.Services.AddHttpClient<BasketServiceDashboardClient>(
    static client => client.BaseAddress = new("http://_dashboard.basket"));
```

In the preceding example, two <xref:System.Net.Http.HttpClient> classes are added, one for the core basket service and one for the basket service's dashboard.

### Named endpoints using configuration

With the configuration-based endpoint resolver, named endpoints can be specified in configuration by prefixing the endpoint value with `_endpointName.`, where `endpointName` is the endpoint name. For example, consider this _appsettings.json_ configuration which defined a default endpoint (with no name) and an endpoint named "dashboard":

```json
{
  "Services": {
    "basket": [
      "10.2.3.4:8080",
      "_dashboard.10.2.3.4:9999"
    ]
  }
}
```

In the preceding JSON:

- The default endpoint, when resolving `http://basket` is `10.2.3.4:8080`.
- The "dashboard" endpoint, resolved via `http://_dashboard.basket` is `_dashboard.10.2.3.4:9999`.

### Named endpoints in .NET Aspire

```csharp
var basket = builder.AddProject<Projects.BasketService>("basket")
    .WithServiceBinding(hostPort: 8888, scheme: "http", name: "dashboard");
```

### Named endpoints in Kubernetes using DNS SRV

When deploying to Kubernetes, the DNS SRV service endpoint resolver can be used to resolve named endpoints. For example, the following resource definition will result in a DNS SRV record being created for an endpoint named "http" and an endpoint named "dashboard", both on the service named "basket".

```yml
apiVersion: v1
kind: Service
metadata:
  name: basket
spec:
  selector:
    name: basket-service
  clusterIP: None
  ports:
  - name: default
    port: 8080
  - name: dashboard
    port: 8888
```

To configure a service to resolve the "dashboard" endpoint on the "basket" service, add the DNS SRV service endpoint resolver to the host builder as follows:

```csharp
builder.Services.AddServiceDiscoveryCore();
builder.Services.AddDnsSrvServiceEndPointResolver();
```

The special port name "default" is used to specify the default endpoint, resolved using the URI `http://basket`.

As in the previous example, add service discovery to an `HttpClient` for the basket service:

```csharp
builder.Services.AddHttpClient<BasketServiceClient>(
    static client => client.BaseAddress = new("http://basket"));
```

Similarly, the "dashboard" endpoint can be targeted as follows:

```csharp
builder.Services.AddHttpClient<BasketServiceDashboardClient>(
    static client => client.BaseAddress = new("http://_dashboard.basket"));
```

<!--
# TODO: Configuring polling interval and pending status refresh interval via `ServiceEndPointResolverOptions`

# TODO: Configuring DNS SRV

# TODO: DNS resolver (non-SRV)

# TODO: Configuring DNS
-->

## See also

- [Service discovery in .NET](/dotnet/core/extensions/service-discovery.md)
- [Make HTTP requests with the HttpClient class](/dotnet/fundamentals/networking/http/httpclient.md)
- [IHttpClientFactory with .NET](/dotnet/core/extensions/httpclient-factory.md)