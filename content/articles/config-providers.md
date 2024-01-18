---
title: "配置提供者"
linkTitle: "配置提供者"
weight: 40
date: 2024-01-18
description: >
  YARP 配置提供者
---

> https://microsoft.github.io/reverse-proxy/articles/config-providers.html

## 简介

[基本 Yarp 示例](https://github.com/microsoft/reverse-proxy/tree/main/samples/BasicYarpSample) 显示代理配置是从 appsettings.json 中加载的。不过，代理配置可以通过编程从你选择的源代码中加载。您可以通过提供几个实现 [IProxyConfigProvider](https://microsoft.github.io/reverse-proxy/api/Yarp.ReverseProxy.Configuration.IProxyConfigProvider.html) 和 [IProxyConfig](https://microsoft.github.io/reverse-proxy/api/Yarp.ReverseProxy.Configuration.IProxyConfig.html) 的类来实现这一点。

请参阅 [ReverseProxy.Code.Sample](https://github.com/microsoft/reverse-proxy/tree/main/samples/ReverseProxy.Code.Sample) 以了解自定义配置提供程序的示例。

可以使用 [Configuration Filters](https://microsoft.github.io/reverse-proxy/articles/config-filters.html) 在加载过程中修改配置。

## 结构体

[IProxyConfigProvider](https://microsoft.github.io/reverse-proxy/api/Yarp.ReverseProxy.Configuration.IProxyConfigProvider.html) 有一个方法 `GetConfig()`，该方法应返回一个 [IProxyConfig](https://microsoft.github.io/reverse-proxy/api/Yarp.ReverseProxy.Configuration.IProxyConfig.html) 实例。IProxyConfig 包含当前路由和群集的列表，以及一个 `IChangeToken` 用来在这些信息过时并需要重新加载时通知代理，这将导致再次调用 `GetConfig()`。

### 路由

路由部分是命名路由的无序集合。路由包含匹配项及其相关配置。路由至少需要以下字段：

- RouteId - 唯一名称
- ClusterId - 指群集部分中的条目名称。
- Match - 包含 Hosts 数组或 Path 模式字符串。Path 是 ASP.NET Core 路由模板，可定义为[此处解释](https://docs.microsoft.com/aspnet/core/fundamentals/routing#route-templates)。

每个路由条目都可以配置 [Headers](https://microsoft.github.io/reverse-proxy/articles/header-routing.html)、[Authorization](https://microsoft.github.io/reverse-proxy/articles/authn-authz.html)、[CORS](https://microsoft.github.io/reverse-proxy/articles/cors.html) 和其他基于路由的策略。有关其他字段，请参阅 [RouteConfig](https://microsoft.github.io/reverse-proxy/api/Yarp.ReverseProxy.Configuration.RouteConfig.html)。

代理将应用给定的匹配标准和策略，然后将请求传递给指定的群集。

### 群集

群集部分是命名群集的无序集合。一个群集主要包含一组命名的目的地及其地址，其中任何一个目的地都被认为有能力处理给定路由的请求。代理将根据路由和集群配置处理请求，以选择目的地。

有关其他字段，请参阅 [ClusterConfig](https://microsoft.github.io/reverse-proxy/api/Yarp.ReverseProxy.Configuration.ClusterConfig.html)。

## 内存配置

InMemoryConfigProvider 实现了 IProxyConfigProvider，可通过调用 LoadFromMemory 直接在代码中指定路由和集群。

```c#
services.AddReverseProxy().LoadFromMemory(routes, clusters)；
```

要稍后更新配置，请解析服务容器中的 `InMemoryConfigProvider` 并使用新的路由和集群列表调用 `Update`。

```c#
httpContext.RequestServices.GetRequiredService<InMemoryConfigProvider>().Update(routes, clusters)；
```

## 生命周期

### 启动

应将 `IProxyConfigProvider` 作为单例在 DI 容器中注册。启动时，代理将解析该实例并调用 `GetConfig()`。在第一次调用时，提供程序可以选择

- 如果提供程序因故无法生成有效的代理配置，则抛出异常。这将阻止应用程序启动。
- 在加载配置时同步阻塞。这将阻止应用程序启动，直到有效路由数据可用。
- 或者，它可以选择在后台加载配置时返回一个空的 `IProxyConfig` 实例。当配置可用时，提供程序需要触发 `IChangeToken`。

代理会验证给定的配置，如果配置无效，就会产生异常，导致应用程序无法启动。提供程序可通过使用 [IConfigValidator](https://microsoft.github.io/reverse-proxy/api/Yarp.ReverseProxy.Configuration.IConfigValidator.html) 对路由和群集进行预验证，并采取任何其认为适当的措施（如排除无效条目）来避免这种情况。

### 原子性

一旦通过 `GetConfig()` 交给代理，提供给代理的配置对象和集合应为只读，不得修改。

## 重新加载

如果 IChangeToken 支持 ActiveChangeCallbacks，代理处理完初始配置集后，就会使用此令牌注册一个回调。如果提供商不支持回调，则将每 5 分钟轮询一次 HasChanged。

当提供商想向代理提供新配置时，它应该这样做：

### 重新加载

如果 "IChangeToken" 支持 "ActiveChangeCallbacks"（活动更改回调），代理处理完初始配置集后，就会使用此令牌注册一个回调。如果提供商不支持回调，则将每 5 分钟轮询一次 "HasChanged"。

当提供程序要向代理提供新配置时，它应该:

- 在后台加载该配置。
  - 路由和集群对象是不可变的，因此必须为任何新数据创建新实例。
  - 不变路由和群集的对象可以重复使用，也可以创建新实例，但会通过差分来检测变化。
- 可选择使用 [IConfigValidator](https://microsoft.github.io/reverse-proxy/api/Yarp.ReverseProxy.Configuration.IConfigValidator.html) 验证配置，然后才从先前的 `IProxyConfig` 实例向 `IChangeToken` 发送新数据可用的信号。代理会再次调用 `GetConfig()` 来获取新数据。

重新加载配置与首次加载配置有重要区别。

- 新配置将与当前配置进行比较，只有修改过的路由或群集才会被更新。更新将以原子方式应用，只会影响新请求，而不会影响当前正在处理的请求。
- 重载过程中出现的任何错误都将被记录并抑制。应用程序将继续使用上次已知的良好配置。
- 如果 `GetConfig()`抛出，代理将无法监听未来的更改，因为 `IChangeToken`是一次性的。

一旦验证并应用了新配置，代理将使用新的 `IChangeToken` 注册一个回调。请注意，如果接连发出多个重新加载信号，代理可能会跳过其中一些，并在下一个可用配置准备就绪时立即加载。每个 `IProxyConfig` 都包含完整的配置状态，因此不会丢失任何内容。

## 示例

InMemoryConfigProvider](https://github.com/microsoft/reverse-proxy/blob/main/src/ReverseProxy/Configuration/InMemoryConfigProvider.cs) 提供了一个 `IProxyConfigProvider` 的示例，其中手动加载了路由和群集。
