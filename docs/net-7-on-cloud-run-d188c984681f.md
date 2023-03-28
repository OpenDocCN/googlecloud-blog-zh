# 。云上运行的 NET 7

> 原文：<https://medium.com/google-cloud/net-7-on-cloud-run-d188c984681f?source=collection_archive---------2----------------------->

![](img/792c943220967f49e73d3f55d60ff165.png)

。NET 7 在几天前[发布了](https://devblogs.microsoft.com/dotnet/announcing-dotnet-7/)，带来了新的特性和性能改进，并且已经在 Google Cloud 上运行了！

在这篇简短的博文中，我将向您展示如何将. NET 7 web 应用程序部署到 Cloud Run。

# 创建一个. NET 7 web 应用程序

首先，确保你开机了。网络 7:

```
dotnet --version

7.0.100
```

创建简单的 web 应用程序:

```
dotnet new web -o helloworld-dotnet7
```

这将创建包含项目文件的`helloworld-dotnet7`文件夹。我喜欢现在默认的`Program.cs`文件如此简单:

```
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

对于云运行，您需要检查`PORT`并监听`0.0.0.0`。将`Program.cs`改为:

```
var builder = WebApplication.CreateBuilder(args);

// Add the following for Cloud Run
var port = Environment.GetEnvironmentVariable("PORT") ?? "8080";
var url = $"http://0.0.0.0:{port}";
builder.WebHost.UseUrls(url);

var app = builder.Build();

app.MapGet("/", () => "Hello World from .NET 7.0!");

app.Run();
```

# 本地测试

让我们在本地运行应用程序进行快速测试。

启动应用程序:

```
dotnet run

Building...
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://0.0.0.0:8080
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /Users/atamel/dev/local/helloworld-dotnet7
```

使用 curl 发送请求:

```
curl http://localhost:8080 

Hello World from .NET 7.0!
```

# 部署到云运行

部署到云运行:

```
gcloud run deploy helloworld-dotnet7 \
    --allow-unauthenticated \
    --region us-central1 \
    --source .
```

注意我们使用了`--source`标志。在幕后，这个命令使用 [Google Cloud buildpacks](https://github.com/GoogleCloudPlatform/buildpacks) 和 Cloud Build 从您的源代码自动构建容器映像，而不需要`Dockerfile`或者在您的机器上安装 Docker。

几分钟后，应用程序就部署到了 Cloud Run。

# 针对云运行的测试

为了测试部署的云运行，获取服务 url 并使用 curl:

```
SERVICE_URL=$(gcloud run services describe helloworld-dotnet7 --region us-central1 --format 'value(status.url)')
curl $SERVICE_URL

Hello World from .NET 7.0!
```

瞧啊。。运行在云上的 NET 7.0。

如有任何问题或反馈，请随时在 Twitter [@meteatamel](https://twitter.com/meteatamel) 上联系我。*最初发布于*[*https://atamel . dev*](https://atamel.dev/posts/2022/11-11_dotnet7_cloud_run/)*。*