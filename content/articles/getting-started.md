---
title: "YARP 入门"
linkTitle: "YARP 入门"
weight: 10
date: 2024-01-18
description: >
  YARP 入门
---

> https://microsoft.github.io/reverse-proxy/articles/getting-started.html

YARP 是一个提供核心代理功能的类库，您可以通过添加或替换模块对其进行定制。YARP 目前以 NuGet 软件包和代码片段的形式提供。我们计划在未来提供一个项目模板和预构建的 exe。

YARP 是在 .NET Core 基础架构之上实现的，可在 Windows、Linux 或 MacOS 上使用。开发工作可通过 SDK 和您最喜欢的编辑器（Microsoft Visual Studio 或 Visual Studio Code）完成。

YARP 2.0.0 支持 ASP.NET Core 6.0 及更新版本。您可以从 https://dotnet.microsoft.com/download/dotnet/ 下载 .NET SDK。

> 备注：这里我下载了 .NET 6.0 for macOS Arm64 版本并安装

## 创建新项目

使用以下步骤创建的项目的完整版本可在 Minimal YARP Sample 中找到。有关不使用顶层语句的版本，请参阅基本 YARP 示例。

首先，使用命令行创建一个 "empty" ASP.NET Core 应用程序：

```bash
dotnet new web -n MyProxy -f net6.0
```

完成之后的目录结构如下：

```bash
ls -ls

8 -rw-r--r--  1 sky  staff  219 Jan 18 21:25 MyProxy.csproj
8 -rw-r--r--  1 sky  staff  135 Jan 18 21:25 Program.cs
0 drwxr-xr-x  3 sky  staff   96 Jan 18 21:25 Properties
8 -rw-r--r--  1 sky  staff  127 Jan 18 21:25 appsettings.Development.json
8 -rw-r--r--  1 sky  staff  151 Jan 18 21:25 appsettings.json
0 drwxr-xr-x  7 sky  staff  224 Jan 18 21:25 obj
```

用 vs code 打开这个项目。

修改 `MyProxy.csproj` 文件，增加内容:

```xml
<ItemGroup> 
 <PackageReference Include="Yarp.ReverseProxy" Version="2.0.0" />
</ItemGroup> 
```

### 添加 YARP 中间件

更新 Program.cs 以使用 YARP 中间件：

```c#
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));
var app = builder.Build();
app.MapReverseProxy();
app.Run();
```

## 配置

YARP 的配置在 appsettings.json 文件中定义。详情请参阅配置文件。

也可以通过编程提供配置。请参阅配置提供程序了解详情。

您可以通过查看 RouteConfig 和 ClusterConfig 进一步了解可用的配置选项。

修改 `appsettings.json` 文件，内容如下：

```json
{
 "Logging": {
   "LogLevel": {
     "Default": "Information",
     "Microsoft": "Warning",
     "Microsoft.Hosting.Lifetime": "Information"
   }
 },
 "AllowedHosts": "*",
 "ReverseProxy": {
   "Routes": {
     "route1" : {
       "ClusterId": "cluster1",
       "Match": {
         "Path": "{**catch-all}"
       }
     }
   },
   "Clusters": {
     "cluster1": {
       "Destinations": {
         "destination1": {
           "Address": "https://example.com/"
         }
       }
     }
   }
 }
}
```

## 运行项目

使用在样本目录中调用的 dotnet run

```bash
dotnet run                         
Building...
info: Yarp.ReverseProxy.Configuration.ConfigProvider.ConfigurationConfigProvider[1]
      Loading proxy data from config.
warn: Microsoft.AspNetCore.DataProtection.KeyManagement.XmlKeyManager[35]
      No XML encryptor configured. Key {2e19e480-715d-4990-a187-831a2dbc72fc} may be persisted to storage in unencrypted form.
warn: Microsoft.AspNetCore.Server.Kestrel.Core.KestrelServer[5]
      The application is trying to access the ASP.NET Core developer certificate key. A prompt might appear to ask for permission to access the key. When that happens, select 'Always Allow' to grant 'dotnet' access to the certificate key in the future.
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: https://localhost:7243
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5260
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /Users/sky/work/code/yarp/MyProxy/
```

这时通过浏览器访问  http://localhost:5260 

可以得到和直接访问 https://example.com/ 一样的页面。反向代理成功！