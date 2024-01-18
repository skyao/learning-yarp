---
title: "支持的运行时"
linkTitle: "支持的运行时"
weight: 20
date: 2024-01-18
description: >
  YARP 支持的运行时
---

> https://microsoft.github.io/reverse-proxy/articles/runtimes.html

YARP 2.0+ 支持 ASP.NET Core 6.0 及更新版本。您可以从 https://dotnet.microsoft.com/download/dotnet/ 下载 .NET SDK。有关具体的版本支持，请参阅 [版本](https://github.com/microsoft/reverse-proxy/releases)。

YARP 正在利用新的 .NET 功能和优化功能。这意味着如果您运行的是以前版本的 ASP.NET，某些功能可能不可用。

### 相关的 6.0 运行时改进

- [HTTP/3](https://microsoft.github.io/reverse-proxy/articles/http3.html) - 支持入站和出站连接（预览版）。
- [分布式跟踪](https://microsoft.github.io/reverse-proxy/articles/distributed-tracing.html) .NET 6.0 内置了可配置的支持，YARP 可利用这些支持启用更多开箱即用的方案。
- [Http.sys Delegation](https://microsoft.github.io/reverse-proxy/articles/httpsys-delegation.html)--内核级 ASP.NET Core 6 功能允许将请求转移到不同的进程。
- [UseHttpLogging](https://microsoft.github.io/reverse-proxy/articles/diagnosing-yarp-issues.html#using-aspnet-request-logging)--包含一个额外的中间件组件，可用于提供有关请求和响应的更多详细信息。
- [动态 HTTP/2 窗口缩放](https://github.com/dotnet/runtime/pull/54755) - 提高高延迟连接上的 HTTP/2 下载速度。
- [非验证标头](https://github.com/microsoft/reverse-proxy/pull/1507) - 通过使用非验证 HttpClient 标头提高性能。

### 相关的 7.0 运行时改进

- [HTTP/3](https://microsoft.github.io/reverse-proxy/articles/http3.html) - 支持入站和出站连接（稳定）。
- [HttpClient 响应流的零字节读取](https://github.com/dotnet/runtime/pull/61913) - 减少了内存使用量。
- [减少头分配](https://github.com/dotnet/runtime/pull/62981) - 减少内存使用量。
- [Kestrel Http/2 性能改进](https://github.com/dotnet/aspnetcore/pull/40925) - 改进一个连接上多个请求的争用和吞吐量。

> 备注：新发布的8.0没有提及。