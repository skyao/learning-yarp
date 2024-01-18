---
title: "配置文件"
linkTitle: "配置文件"
weight: 30
date: 2024-01-18
description: >
  YARP 配置文件
---

> https://microsoft.github.io/reverse-proxy/articles/config-files.html

## 简介

反向代理可以使用 Microsoft.Extensions 中的 IConfiguration 抽象从文件加载路由和群集的配置。这里给出的示例使用的是 JSON，但任何 IConfiguration 源文件都可以使用。如果源文件发生变化，也无需重启代理即可更新配置。

## 加载配置

要从 IConfiguration 加载代理配置，请在 Program.cs 中添加以下代码：

```c#
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// Add the reverse proxy capability to the server
builder.Services.AddReverseProxy()
    // Initialize the reverse proxy from the "ReverseProxy" section of configuration
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

// Register the reverse proxy routes
app.MapReverseProxy();

app.Run();
```

注：有关中间件顺序的详细信息，请参阅此处。

可使用配置过滤器在加载过程中修改配置。

## 多个配置源

从 1.1 开始，YARP 支持从多个来源加载代理配置。LoadFromConfig 可以多次调用，引用不同的 IConfiguration 部分，也可以与不同的配置源（如 InMemory）相结合。路由可以引用其他来源的群集。注意，不支持为给定路由或群集合并不同来源的部分配置。

```c#
services.AddReverseProxy()
    .LoadFromConfig(Configuration.GetSection("ReverseProxy1"))
    .LoadFromConfig(Configuration.GetSection("ReverseProxy2"));
```

或者

```c#
services.AddReverseProxy()
    .LoadFromMemory(routes, clusters)
    .LoadFromConfig(Configuration.GetSection("ReverseProxy"));
```

## 配置合约

基于文件的配置由 IProxyConfigProvider 实现动态映射到 Yarp.ReverseProxy.Configuration 名称空间中的类型，在应用程序启动时和每次配置更改时进行转换。

## 配置结构

配置由上文通过 Configuration.GetSection("ReverseProxy") 指定的命名部分组成，并包含路由和群集的子部分。

示例:

```json
{
  "ReverseProxy": {
    "Routes": {
      "route1" : {
        "ClusterId": "cluster1",
        "Match": {
          "Path": "{**catch-all}",
          "Hosts" : [ "www.aaaaa.com", "www.bbbbb.com"],
        },
      }
    },
    "Clusters": {
      "cluster1": {
        "Destinations": {
          "cluster1/destination1": {
            "Address": "https://example.com/"
          }
        }
      }
    }
  }
}
```

## 路由

路由部分是路由匹配及其相关配置的无序集合。路由至少需要以下字段：

- RouteId - 唯一名称
- ClusterId - 指群集部分中的条目名称。
- Match - 包含 Hosts 数组或 Path 模式字符串。Path 是 ASP.NET Core 路由模板，可按此处的说明进行定义。路由匹配基于具有最高优先级的最特定路由，如此处所述。可以使用 order 字段实现显式排序，值越小优先级越高。

可以在每个路由条目上配置 Headers, Authorization, CORS 和其他基于路由的策略。有关其他字段，请参阅 RouteConfig。

代理将应用给定的匹配标准和策略，然后将请求传递给指定的群集。

## 集群

集群部分是命名集群的无序集合。一个群组主要包含命名的目的地及其地址，其中任何一个目的地都被认为可以处理给定路由的请求。代理将根据路由和群组配置处理请求，以选择目的地。

有关其他字段，请参阅 ClusterConfig。

## 所有配置属性

```json
{
  // 服务器监听的基本 URL，必须独立于下面的路由进行配置
  "Urls": "http://localhost:5000;https://localhost:5001",
  "Logging": {
    "LogLevel": {
      "default": "Information",
      // 取消注释可从运行时和代理中隐藏诊断信息
      // "Microsoft": "Warning",
      // "Yarp" : "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "ReverseProxy": {
    // 路由告诉代理转发哪些请求
    "Routes": {
      "minimumroute" : {
        // 匹配任何内容并将其路由到 www.example.com
        "ClusterId": "minimumcluster",
        "Match": {
          "Path": "{**catch-all}"
        }
      },
      "allrouteprops" : {
        // 匹配 /something/* 和路由到 "allclusterprops"
        "ClusterId": "allclusterprops", // 其中一个群集的名称
        "Order" : 100, // 数字越小优先级越高
        "MaxRequestBodySize" : 1000000, // 以字节为单位。可选择覆盖服务器的限制（默认为 30MB）。设置为 -1 则禁用。
        "AuthorizationPolicy" : "Anonymous", // 策略名称或 "Default"，"Anonymous"
        "CorsPolicy"："Default"，// 适用于此路由的 CorsPolicy 名称或 "Default","Disable"
        "Match": {
          "Path": "/something/{**remainder}", // 使用 ASP.NET 语法匹配的路径。
          "Hosts" : [ "www.aaaaa.com", "www.bbbbb.com"], // 要匹配的主机名，未指定的为任意主机名
          "Methods" : [ "GET", "PUT" ], // 匹配的 HTTP 方法，未指定的为所有方法
          "Headers": [// 要匹配的头信息，未指定的为任何头信息
            {
              "名称": "MyCustomHeader", // 标头名称
              "Values": [ "value1", "value2", "another value" ], // 匹配这些值中的任何一个
              "Mode": "ExactHeader", // 或 "HeaderPrefix", "Exists" , "Contains", "NotContains", "NotExists"
              "IsCaseSensitive": true
            }
          ],
          "QueryParameters": [ // 要匹配的查询参数，未指定的为任意参数
            {
              "名称": "MyQueryParameter", // 查询参数名称
              "Values": [ "value1", "value2", "another value" ], // 与这些值中的任意值进行匹配
              "模式": "精确"，//或 "前缀","存在","包含","不包含"
              "IsCaseSensitive": true
            }
          ]
        },
        "MetaData"：{ // 可由自定义扩展使用的键值对列表
          "MyName"："MyValue"
        },
        "Transforms" : [ // 转换列表。有关详细信息，请参阅 "变换 "文章
          {
            "RequestHeader": "MyHeader",
            "Set": "MyValue",
          }
        ]
      }
    },
    // 集群告诉代理转发请求的位置和方式
    "Clusters": {
      "minimumcluster": {
        "Destinations": {
          "example.com": {
            "Address": "http://www.example.com/"
          }
        }
      },
      "allclusterprops": {
        "Destinations": {
          "first_destination": {
            "Address": "https://contoso.com"
          },
          "another_destination": {
            "Address": "https://10.20.30.40",
            "Health" : "https://10.20.30.40:12345/test" // 覆盖主动健康检查
          }
        },
        "LoadBalancingPolicy" : "PowerOfTwoChoices", // 可选择 "FirstAlphabetical", "Random", "RoundRobin", "LeastRequests" 或 "FirstAlphabetical", "Random", "RoundRobin", "LeastRequests".
        "SessionAffinity": {
          Enabled": true, // 默认为 "false
          "Policy": "Cookie"，//默认，或者 "CustomHeader"（自定义头文件
          "FailurePolicy": "Redistribute"，//默认，或者 "Return503Error"。
          "Settings" : {
              "CustomHeaderName": MySessionHeaderName" // 默认为 "X-Yarp-Proxy-Affinity`"。
          }
        },
        "HealthCheck": {
          "Active": { // 调用 API 来验证健康状况。
            "Enabled": "true",
            "Interval": "00:00:10",
            "Timeout": "00:00:10",
            "Policy": "ConsecutiveFailures",
            "Path": "/api/health" // 用于查询健康状态的 API 端点
          },
          "Passive": { // 根据 HTTP 响应代码禁用目的地
            "Enabled": true, // 默认为 false
            "Policy" : "TransportFailureRateHealthPolicy", // Required
            "ReactivationPeriod" : "00:00:10" // 10s
          }
        },
        "HttpClient" : {用于联系目的地的 HttpClient 实例的配置
          "SSLrotocols" : "Tls13",
          "DangerousAcceptAnyServerCertificate" : false,
          "MaxConnectionsPerServer" : 1024,
          "EnableMultipleHttp2Connections" : true,
          "RequestHeaderEncoding" : "Latin1", // 如何解释请求头值中的非 ASCII 字符
          "ResponseHeaderEncoding" : "Latin1" // 如何解释响应头值中的非 ASCII 字符
        },
        "HttpRequest" : { // 向目的地发送请求的选项
          "ActivityTimeout" : "00:02:00",
          "Version" : "2",
          "VersionPolicy" : "RequestVersionOrLower",
          "AllowResponseBuffering" : "false" （允许响应缓冲）。
        },
        "MetaData" : { // 自定义键值对
          "TransportFailureRateHealthPolicy.RateLimit"： "0.5", // 被动健康策略使用
          "MyKey" : "MyValue" 
        }
      }
    }
  }
}
```

更多信息，请参阅日志配置和 HTTP 客户端配置。