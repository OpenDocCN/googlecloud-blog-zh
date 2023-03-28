# Firebase:使用 Python 和云 Firestore 开发应用引擎服务

> 原文：<https://medium.com/google-cloud/firebase-developing-an-app-engine-service-with-python-and-cloud-firestore-1640f92e14f4?source=collection_archive---------0----------------------->

![](img/d3e9819aea38221812d23a42683526a0.png)

在我之前的文章中，我展示了如何使用[谷歌云功能](https://cloud.google.com/functions/)部署 Python web 服务。该服务使用 [Firebase 管理 SDK](https://firebase.google.com/docs/admin/setup) 来读写 [Firebase 实时数据库](https://firebase.google.com/docs/database/)。在这篇文章中，我们将把同样的服务迁移到[谷歌应用引擎(GAE)](https://cloud.google.com/appengine/) 标准环境中。但更重要的是，我们将通过使用[谷歌云 Firestore](https://firebase.google.com/docs/firestore/) 而不是实时数据库来增加赌注。

这是一件大事，因为开发人员在很长一段时间内无法从 GAE 标准环境中访问云 Firestore。谷歌云 Firestore 的 Python 客户端使用多线程，这在 GAE 标准中是不允许的。那么是什么改变了呢？

谷歌云平台最近为 GAE 推出了第二代 Python 3.7 运行时[。这个新的运行时没有旧的 Python 2.7 运行时的线程限制。它还放松了其他几个恼人的限制，这些限制过去常常让开发者抓狂。具体来说，新的运行时允许开发人员使用任何依赖项，直接访问远程端点，甚至访问本地文件系统的`/tmp`目录。旧运行时](https://cloud.google.com/blog/products/gcp/introducing-app-engine-second-generation-runtimes-and-python-3-7)的限制可以追溯到 2008 年左右，有很好的理由——主要是为了确保运行在共享基础设施上的应用程序的安全性和隔离性。但是从那以后，容器和用户空间内核技术，如 [gVisor](https://github.com/google/gvisor) 开始发挥作用。因此，开发人员现在可以享受更多的自由和权力，同时保持与以前相同的安全和隔离级别。

记住这一点，让我们继续实现一个从 GAE 标准环境连接到 Firestore 的 web 服务。

## 设置开发环境

您将需要一个启用计费的 Firebase 项目，以及一个本地安装的`[gcloud](https://cloud.google.com/sdk/)`命令行工具。如果你的项目是全新的，确保为它启用 [Firestore 支持](https://firebase.google.com/docs/firestore/quickstart)和 [App 引擎支持](https://cloud.google.com/appengine/docs/standard/python3/quickstart)。您还需要 Python 3 和`[virtualenv](https://virtualenv.pypa.io/en/stable/)`实用程序。在 Linux/Unix shell 中执行以下命令来设置开发环境。

```
$ gcloud config set project <your-project-id>
$ virtualenv -p python3 env
$ source env/bin/activate
(env) $ mkdir heroes
(env) $ cd heroes/
```

这些命令为您的应用程序创建一个新的`virtualenv`沙箱，并创建一个名为`heroes`的新的空目录。我们将使用这个目录来存放与您的服务相关的所有源文件。

接下来，您需要设置 Google [应用程序默认凭证](https://cloud.google.com/docs/authentication/production)。这使您能够在将服务部署到 GAE 之前对其进行本地测试。本地测试确实产生了一些额外的工作，但是从长远来看，它节省了大量的时间和精力。为您的 Firebase 项目下载一个[服务帐户](https://console.firebase.google.com/project/_/settings/serviceaccounts/adminsdk) JSON 文件，并设置`GOOGLE_APPLICATION_CREDENTIALS`环境变量指向它。

```
(env) $ export GOOGLE_APPLICATION_CREDENTIALS=path/to/creds.json
```

服务帐户 JSON 文件不应与您的应用程序在同一个目录中。单独保存，确保不会不小心上传到云端。

## 实现服务

我们的 web 服务将只包含 3 个文件:

*   `requirements.txt` —声明所需的依赖关系
*   `main.py` —包含实现我们服务的 Python 源代码
*   `app.yaml`—GAE 部署描述符

让我们从创建`requirements.txt`文件开始。过去开发者不得不将[厂商在](https://cloud.google.com/appengine/docs/standard/python/tools/using-libraries-python-27)的所有依赖项放到一个单独的子目录中，并将其作为应用程序的一部分上传。但是对于新的运行时，我们只是在一个`requirements.txt`文件中声明依赖关系。GAE 自动获取并安装以这种方式声明的依赖项。清单 1 显示了这个文件在我们的例子中应该是什么样子。

清单 1:依赖声明(requirements.txt)

通常你不需要宣布`google-cloud-firestore`为属地。Firebase Admin SDK 应该会自动为您安装。但是有一个[已知问题](https://github.com/firebase/firebase-admin-python/issues/184)是由 `[pip](https://github.com/pypa/pip/issues/4957)`中的 [bug 引起的，这有时会阻止它正确工作。解决这个问题最简单的方法是将`google-cloud-firestore`声明为直接依赖。清单 1 就绪后，运行下面的命令在开发环境中安装依赖项，这样就可以在本地测试服务了。](https://github.com/pypa/pip/issues/4957)

```
(env) $ pip install -r requirements.txt
```

现在是时候编写我们的 web 服务了。我们使用 Flask 来处理所有的请求路由，使用 Firebase Admin SDK 来初始化 Firestore 客户端实例。清单 2 展示了完整的实现。

清单 2: Web 服务实现(main.py)

请注意，这几乎与几个月前我们为 Google 计算引擎环境[实现的 Python 应用程序](/google-cloud/firebase-developing-a-web-service-with-admin-sdk-flask-and-google-cloud-6fb97eb38b80)相同。事实上，如果将该实现中的实时数据库调用替换为 Firestore 调用，就会得到清单 2。`__main__`部分仅在本地测试期间被调用。在生产环境中，GAE 将为您提供合适的应用服务器，如`gunicorn`。

最后，我们创建名为`app.yaml`的 GAE 部署描述符。这里唯一需要注意的是`runtime: python37`条目。清单 3 显示了最终结果。

清单 3: GAE 部署描述符(app.yaml)

## 在本地测试

是时候去兜一圈了。只需按如下方式启动`main.py`即可启动并运行服务。

```
(env) $ python main.py
```

这将启动一个监听端口 8080 的 web 服务器。您可以向它发送几个请求，以确保一切按预期运行。

```
(env) $ curl -v -X POST -d '{"name":"Spider-Man"}' -H "Content-type: application/json" [http://localhost:8080/heroes](http://localhost:5000/heroes)
< HTTP/1.0 201 CREATED
< Content-Type: application/json
< Content-Length: 35
< Server: Werkzeug/0.14.1 Python/3.6.1
< Date: Fri, 07 Sep 2018 22:54:57 GMT
<
{
   "id": "oit8FKTbPVHgrtTtgKZ7"
}(env) $ curl -v [http://localhost:8080/heroes/oit8FKTbPVHgrtTtgKZ7](http://localhost:8080/heroes/oit8FKTbPVHgrtTtgKZ7)
< HTTP/1.0 200 OK
< Content-Type: application/json
< Content-Length: 35
< Server: Werkzeug/0.14.1 Python/3.6.1
< Date: Fri, 07 Sep 2018 22:55:30 GMT
<
{
  "name": "Spider-Man"
}
```

当您与 web 服务交互时，通过 Firebase 控制台检查您的云 Firestore 数据库的内容。您应该会看到`superheroes`集合中的数据随着您的请求而改变。

## 部署到 GAE

使用`gcloud`命令行实用程序将代码部署到 GAE。从与您的`app.yaml`文件相同的目录中运行以下命令。

```
(env) $ gcloud app deploy
```

部署需要几分钟时间。回想一下，GAE 必须基于`requirements.txt`文件在云中安装依赖项。完成后，CLI 会记录实时应用程序的 URL。

```
Beginning deployment of service [heroes]...
╔════════════════════════════════════════════════════════════╗
╠═ Uploading 2 files to Google Cloud Storage                ═╣
╚════════════════════════════════════════════════════════════╝File upload done.
Updating service [heroes]...done.
Setting traffic split for service [heroes]...done.
Deployed service [heroes] to [https://heroes-dot-my-project.appspot.com]
```

让我们通过进行几个 HTTP 调用来结束工作，以确保一切正常。第一次请求可能会多花一两秒钟，因为 GAE·莱希会在第一次请求时加载应用程序。

```
(env) $ curl -v -X POST -d '{"name":"Spider-Man"}' -H "Content-type: application/json" [https://heroes-dot-my-project.appspot.com/heroes](https://heroes-dot-fireflicks-io.appspot.com/heroes)
< HTTP/2 201
< content-type: application/json
< date: Fri, 07 Sep 2018 23:00:23 GMT
< server: Google Frontend
...
<
{"id":"ZWJcJNCLSw7L2WxOqguQ"}
```

您可以运行以下命令直接在控制台上流式传输应用程序日志:

```
(env) $ gcloud app logs tail -s heroes
Waiting for new log entries...
2018-09-07 23:00:18 heroes[20180907t155731]  "POST /heroes HTTP/1.1" 201
2018-09-07 23:01:41 heroes[20180907t155731]  "GET /heroes/ZWJcJNCLSw7L2WxOqguQ HTTP/1.1" 200
2018-09-07 23:01:53 heroes[20180907t155731]  "DELETE /heroes/ZWJcJNCLSw7L2WxOqguQ HTTP/1.1" 200
```

## 结论

我是谷歌应用引擎的长期用户。实际上，我为我的研究生研究学习了它的架构和操作([例如](https://dl.acm.org/citation.cfm?id=2806842))。在许多方面，GAE 是将 PaaS 和无服务器带入主流的技术。看到不同的 GAE 语言运行时是如何随着时间的推移而演变的令人耳目一新。他们放宽了过去的许多限制，从而给了开发者更多的自由。与此同时，GAE 作为一个平台，通过更好的工具、监控、日志和模块化，极大地改进了对应用治理的支持。

GAE 的新 Python 3.7 运行时是构建与 Firebase 交互的微服务的绝佳平台。我希望你和我一样觉得它强大而令人兴奋。