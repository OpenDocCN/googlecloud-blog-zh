# 在 Google App Engine 上部署您的 Go 应用

> 原文：<https://medium.com/google-cloud/deploying-your-go-app-on-google-app-engine-5f4a5c2a837?source=collection_archive---------0----------------------->

![](img/2b911cc684c16f70b79c906c4ed60f42.png)

谷歌云平台

*本教程将向您展示如何在 GCP 应用引擎标准和灵活环境中部署 Go 应用*

# 先决条件

*   您已经安装了`gcloud`工具
*   您已经安装了 [Go](https://golang.org/doc/install)
*   你在 GCP 有一个项目

# 我们开始吧

首先，让我们制作一个目录并输入它:

```
mkdir ~/hello-world-go && cd hello-world-go
```

现在确保我们将 GOPATH 设置到这个目录:

```
export GOPATH=$(pwd):$GOPATH
```

接下来，我们将设置项目目录:

```
mkdir -p src/go-app && cd src/go-app
```

现在，让我们去获取 GCP 应用引擎库:

```
go get -u google.golang.org/appengine/...
```

最后，我们将我们的项目 ID(来自 GCP)设置为一个环境变量，并配置`gcloud`来使用它:

```
PROJECT_ID=<project-id>
gcloud config set project $PROJECT_ID
```

# 是时候写一些代码了

首先我们需要制作一个`app.yaml`文件来配置你的项目设置:

```
runtime: go
api_version: go1
env: standard
handlers:
- url: /.*
  script: _go_app
```

这告诉应用程序引擎我们想要使用 Go 运行时，我们想要使用标准环境，并为服务器收到请求时设置处理程序。

> 注意第 3 行，这里写着`*env: standard*`

现在让我们为我们的服务器创建`main.go`文件:

```
package mainimport (
  "fmt"
  "net/http"
  "google.golang.org/appengine"
)func handler(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, "Hello, world!") // Response to request
}func main() {
  http.HandleFunc("/", handler) // Set endpoint handler
  appengine.Main() // Start the server
}
```

这首先导入我们需要的库，然后为您点击`/`端点时设置一个处理程序，最后启动服务器。

# 开始部署吧！

这太简单了，只需运行:

```
gcloud app deploy
```

当它问你`Do you want to continue (Y/n)?`时，确保点击`Y`

> 保护提示:如果您想跳过确认，请使用`*yes | gcloud app deploy*`

现在，这应该需要大约 30 秒来部署(**这么快**)。

完成后，运行`gcloud app browse`或打开浏览器进入[https://<project-id>. appspot . com/](https://go-project69.appspot.com/)

# 奖金

上面，我们将 Go 应用程序部署到了 App Engine 标准环境中。但是现在让我们看看部署到灵活的环境有多容易。

还记得我们的`app.yaml`文件吗？让我们改变第三行中的一个词:

```
runtime: go
api_version: go1
env: flex
handlers:
- url: /.*
  script: _go_app
```

你能看出区别吗？我们将`env: standard`改为`env: flex`

现在只需重新运行`gcloud app deploy`并再次点击`Y`进行确认。由于我们正在部署到 Flex 环境中，这将需要更长的时间(大约 3-5 分钟)。

每种环境都有不同的优势，我不会在这里讨论它们。然而，如果你想深入了解，这里有一个关于[选择应用引擎环境](https://cloud.google.com/appengine/docs/the-appengine-environments)的快速指南。

# 👋 ✌️

感谢阅读！如有疑问，在下方评论！