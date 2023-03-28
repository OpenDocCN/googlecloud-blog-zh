# Firebase:使用 Admin SDK、Flask 和 Google Cloud 开发 Web 服务

> 原文：<https://medium.com/google-cloud/firebase-developing-a-web-service-with-admin-sdk-flask-and-google-cloud-6fb97eb38b80?source=collection_archive---------0----------------------->

![](img/e33ac5c8d0daeb3a6d7a8398f40d05d5.png)

[Firebase 管理 SDK](https://firebase.google.com/docs/admin/setup)支持从可信环境访问 [Firebase](https://firebase.google.com) 。因此，Admin SDKs 可用于与各种公共云和私有云的 Firebase 进行交互。顺便提一下，Google Compute Engine、Google App Engine 和 Google Cloud Functions 等精选云平台产品是基于管理 SDK 的服务器端 Firebase 应用程序的绝佳部署目标。这些云环境在规模、便利性、DevOps 自动化和成本方面为开发人员提供了广泛的选择。

在本帖中，我们将使用 [Admin Python SDK](https://github.com/firebase/firebase-admin-python) 和 [Flask](http://flask.pocoo.org/) 开发一个简单的 web 服务。像在大多数服务器端开发项目中一样，我们将首先在本地编码和测试应用程序。然后，我们将通过在[谷歌计算引擎](https://cloud.google.com/compute/)中部署代码来进行生产演练，谷歌的基础设施即服务(IaaS)云。我们将学习如何从服务器端 Python 应用程序访问 Firebase，以及如何授权部署在[谷歌云平台(GCP)](https://cloud.google.com) 中的代码发出的 Firebase API 调用。我们还将探索一些潜在的陷阱，并提出解决方案。

## 设置开发环境

首先，我们需要一台 Python 2.7 或更高版本的工作站。我还建议使用`[virtualenv](https://virtualenv.pypa.io/en/stable/)`将你的项目和它的依赖项与其他所有东西隔离开来。在 Unix/Linux shell 中执行以下命令，启动一个新的`virtualenv`沙箱，并安装所需的模块。

```
$ virtualenv env
$ source env/bin/activate
(env) $ pip install firebase-admin flask
```

我们使用 [Google 应用程序默认凭证(ADC)](https://developers.google.com/identity/protocols/application-default-credentials) 来授权我们的应用程序发出的 Firebase API 调用。ADC 是部署在 GCP 的应用程序的推荐授权机制。它使我们不必将凭证硬编码到应用程序中，或者将敏感的凭证文件与代码一起发送。然而，为了对使用 ADC 的应用进行本地测试，我们需要做一些事情。首先，为您的 Firebase 项目下载[服务帐户凭证](https://firebase.google.com/docs/admin/setup#add_firebase_to_your_app)。然后设置`GOOGLE_APPLICATION_CREDENTIALS`环境变量指向下载的文件。

```
(env) $ export GOOGLE_APPLICATION_CREDENTIALS=path/to/creds.json
```

最后，我们需要确保系统中安装了 [Google Cloud SDK](https://cloud.google.com/sdk/) 。这给了我们`gcloud`命令行工具。我们稍后将使用它来与 Google Compute Engine 进行交互。

## 编写应用程序

事不宜迟，让我们继续编码我们的 Python web 服务。如果您正在跟进，您可以简单地将清单 1 的内容复制到一个名为`superheroes.py`的文件中。确保替换第 10 行的`<DB_NAME>`占位符，使它指向您的 Firebase 数据库。

清单 1:基于 Admin SDK 和 Flask 的 Python web 服务

在清单 1 的第 9 行，我们通过调用`initialize_app()`来初始化 Firebase Admin SDK。这是在函数之外完成的，以确保它只发生一次。请注意，我们没有向`initialize_app()`传递任何显式凭证。这将提示 SDK 寻找 ADC。然后我们获得一个对名为`superheroes`的实时数据库节点的引用。我们的 web 服务管理的所有数据都将存储在这个节点下。接下来，我们有四个函数，它们构成了 web 服务的公共接口。注意它们是如何使用 Flask decorators 映射到不同的 HTTP 方法和 URL 路径的。最后，我们有一个名为`_ensure_hero()`的内部助手函数。

部署时，超级英雄 web 服务公开了与典型 CRUD 操作相对应的四个操作:

*   创建一个新的超级英雄条目。
*   `GET /heroes/<id>`:通过 ID 检索超级英雄条目。
*   `PUT /heroes/<id>`:按 ID 更新超级英雄条目。
*   `DELETE /heroes/<id>`:按 ID 删除超级英雄条目。

## 带它去兜风

现在我们已经为本地试运行做好了准备。所以让我们点燃那个瓶子吧！

```
(env) $ export FLASK_APP=superheroes.py
(env) $ flask run
```

这将启动一个监听端口 5000 的 web 服务器。让我们尝试在数据库中创建一个新的超级英雄。清单 2 显示了我们将使用的 JSON 有效负载。只需将其复制到一个名为`spiderman.json`的文件中。

清单 2:创建条目的示例 JSON 有效负载

现在运行下面的 curl 命令将`spiderman.json`文件发送到我们的 web 服务。如果一切顺利，它将返回一个包含唯一 ID 的`201 Created`响应。

```
$ curl -v -X POST -d @spiderman.json -H "Content-type: application/json" [http://localhost:5000/heroes](http://localhost:5000/heroes)
< HTTP/1.0 201 CREATED
< Content-Type: application/json
< Content-Length: 35
< Server: Werkzeug/0.13 Python/2.7.6
< Date: Fri, 15 Dec 2017 22:16:39 GMT
< 
{
  "id": "-L0RF2E2upW9jhCjmi6R"
}
```

此时，您还应该返回 Firebase 控制台，检查实时数据库的内容。在`superheroes/`路径下创建一个新条目。您会注意到新条目的密钥与我们的 web 服务返回的 ID 相同。您可以将这个 ID 发送回 web 服务来检索超级英雄条目。

```
$ curl -v http://localhost:5000/heroes/-L0RF2E2upW9jhCjmi6R
< HTTP/1.0 200 OK
< Content-Type: application/json
< Content-Length: 197
< Server: Werkzeug/0.13 Python/2.7.6
< Date: Fri, 15 Dec 2017 22:18:11 GMT
< 
{
  "name": "Spider-Man", 
  "realName": "Peter Parker", 
...
```

也可以随意尝试上传和删除请求。查看每个操作如何改变 Firebase 数据库中的内容。如果你发送一个带有无效超级英雄 id 的请求，服务器会回复一个`404 Not Found`响应。这个检查是在清单 1 中的`_ensure_hero()`助手函数中实现的。

```
$ curl -v http://localhost:5000/heroes/invalid_id
< HTTP/1.0 404 NOT FOUND
< Content-Type: text/html
< Content-Length: 233
< Server: Werkzeug/0.13 Python/2.7.6
< Date: Fri, 15 Dec 2017 22:18:16 GMT
< 
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>404 Not Found</title>
...
```

## 部署到 Google 计算引擎:第 1 课

将生产 web 服务部署到 IaaS 云时，需要考虑很多因素。要使用多少个虚拟机实例？虚拟机实例的大小是多少？如何处理负载均衡？使用什么样的监控？我们如何根据不同的负载条件进行扩展？这份问题清单真的令人震惊。由于这个原因，在像 Google App Engine 这样的平台即服务(PaaS)云中部署面向 web 的应用程序通常更简单、更好。他们为我们处理所有繁重的工作。但是 IaaS 有它的用途，这个练习的目的是看看我们如何从计算引擎访问 Firebase。因此，我们将保持事情绝对简单，只将我们的代码部署到单个 VM 实例。在以后的文章中，我将讨论如何将这个 web 服务移植到 Google App Engine。

最重要的是。我们需要确保`gcloud`命令行工具被配置为与“正确的”GCP 项目交互——特别是当您有多个项目在使用的时候。回想一下，我们的应用程序使用 ADC 来授权 Firebase 交互。为此，我们应该在与 Firebase 数据库相同的 GCP 项目中启动我们的 VM。Firebase 项目实际上是一个伪装的 GCP 项目——这意味着当您创建 Firebase 项目时，您实际上创建了一个 GCP 项目。我们需要确保我们的 Firebase 数据库和计算引擎 VM 实例驻留在同一个项目中。可以理解的是，项目`X`中的 ADC 不能授权访问项目`Y`中的 Firebase 数据库——至少在没有额外配置的情况下不能。要为`gcloud`指定项目，从 Firebase 控制台找出您的项目 ID，并运行以下命令。

```
$ gcloud config set project *my_firebase_project_id*
```

现在让我们启动一个新的 Linux VM 实例，并通过 SSH 连接到它。我将把这个实例称为`flask-demo`。

```
$ gcloud compute instances create flask-demo
$ gcloud compute instances start flask-demo
$ gcloud compute instances ssh flask-demo
```

计算引擎实例相当简单。所以你必须做一些工作来安装所有必要的软件。在虚拟机中运行以下命令。

```
(vm) $ sudo apt-get install -y python-pip
(vm) $ pip install --user firebase-admin google-auth-oauthlib
(vm) $ sudo pip install flask
```

这里需要注意一些事情。我们的应用程序并不真正需要包`google-auth-oauthlib`。它的安装只是为了克服一个加密模块加载问题。在我写这篇文章的时候，这个问题正在被解决，所以你可能不需要它了。其次，我们全局安装 Flask，这样我们就可以从 shell 中运行`flask`命令行工具。请注意，我们在这里没有使用`virtualenv`，但是如果您愿意，可以随意使用。

一旦 VM 实例完全设置好，就将 web 服务实现复制到 VM 的文件系统中。您可以`scp`源文件，或者简单地手动复制和粘贴文件内容，因为`superheroes.py`很小。在现实世界中，您将拥有版本控制系统和其他工具来处理此类任务。最后，点燃那个瓶子！

```
(vm) $ export FLASK_APP=superheroes.py
(vm) $ flask run
```

我们还没有配置虚拟机的防火墙。因此，我们不能通过远程发送请求来测试这一点。但是我们应该能够在虚拟机上运行 curl 命令。然而，当我们这么做的时候，我们得到了这个。

```
< HTTP/1.0 500 INTERNAL SERVER ERROR
< Content-Type: text/html
< Content-Length: 291
< Server: Werkzeug/0.13 Python/2.7.13
< Date: Fri, 15 Dec 2017 01:47:01 GMT
<
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
...
```

Flask 服务器还记录了如下内容。

```
[2017-12-15 01:47:01,051] ERROR in app: Exception on /heroes/spiderman [GET]
Traceback (most recent call last):
  File "/usr/local/lib/python2.7/dist-packages/flask/app.py", line 1982, in wsgi_app
    response = self.full_dispatch_request()
  File "/usr/local/lib/python2.7/dist-packages/flask/app.py", line 1614, in full_dispatch_request
    rv = self.handle_user_exception(e)
  File "/usr/local/lib/python2.7/dist-packages/flask/app.py", line 1517, in handle_user_exception
    reraise(exc_type, exc_value, tb)
  File "/usr/local/lib/python2.7/dist-packages/flask/app.py", line 1612, in full_dispatch_request
    rv = self.dispatch_request()
  File "/usr/local/lib/python2.7/dist-packages/flask/app.py", line 1598, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/home/hkj/flask-demo/demo.py", line 23, in read_hero
    hero = superheroes.child(id).get()
  File "/usr/local/lib/python2.7/dist-packages/firebase_admin/db.py", line 147, in get
    return self._client.body('get', self._add_suffix())
  File "/usr/local/lib/python2.7/dist-packages/firebase_admin/_http_client.py", line 93, in body
    resp = self.request(method, url, **kwargs)
  File "/usr/local/lib/python2.7/dist-packages/firebase_admin/db.py", line 769, in request
    raise ApiCallError(self._extract_error_message(error), error)
ApiCallError: **401 Client Error: Unauthorized for url: https://my-db-name.firebaseio.com/superheroes/****-L0RF2E2upW9jhCjmi6R****.json**
Reason: Unauthorized request.
127.0.0.1 - - [15/Dec/2017 01:47:01] "GET /heroes/-L0RF2E2upW9jhCjmi6R HTTP/1.1" 500 -
```

哎呀！发生了什么事？似乎我们的 web 服务从 Firebase 数据库收到了一个`401 Unauthorized`响应。换句话说，我们的 VM 实例上的 ADC 未能授权 Firebase 数据库调用。但是为什么呢？

为了完全理解发生了什么，我们需要对什么是 *OAuth2 作用域*有一个粗略的概念。

## 在虚拟机上设置 OAuth2 作用域

[Scopes](https://www.oauth.com/oauth2-servers/scope/) 提供了一种方法来限制用户从 OAuth2 令牌获得的访问级别。通过将令牌的范围缩小到几个服务，服务提供商可以确保令牌的持有者只能访问这些服务。这在像 GCP 这样有许多服务的环境中非常有用。作用域构成了防止意外或恶意滥用云服务的一层防御。它们还有助于以更细粒度的方式授权客户端。例如，在同一个 GCP 项目中，一个用户只能访问云存储，另一个用户只能访问云 Firestore。这是通过在颁发给两个用户的令牌中包含适当的范围子集来实现的。

各种 GCP 服务和访问它们所需的 OAuth2 范围在这里[有所记载](https://developers.google.com/identity/protocols/googlescopes)。要访问 Firebase 数据库，客户端应用程序必须提供具有以下两个范围的 OAuth2 令牌:

*   `https//www.googleapis.com/auth/firebase.database`
*   `https://www.googleapis.com/auth/userinfo.email`

但是，计算引擎中的 ADC 不会将这些范围添加到颁发的令牌中。因此，Firebase database 拒绝了我们在云中的 web 服务发出的请求。这反过来导致了我们所看到的错误。

计算引擎中的每个虚拟机实例都有一组固定的关联范围。VM 实例上的 ADC 将只在 OAuth2 令牌中包括这些范围。因此，我们问题的解决方案是用所需的作用域配置 VM 实例。我们可以通过运行以下命令来检查它当前的作用域。

```
$ gcloud compute instances describe flask-demo --format json
...
"serviceAccounts": [
   {
    "email": "your.gserviceaccount.com",
    "scopes": [
     "https://www.googleapis.com/auth/cloud-platform",
     "https://www.googleapis.com/auth/userinfo.email"
     ]
    }
  ],
...
```

输出中的`serviceAccounts`部分显示了 VM 中当前设置的 OAuth2 范围。我们可以看到一个必需的 Firebase 范围丢失了。因此，我们需要停止 VM 实例，添加所需的作用域，然后再次启动它。

```
$ gcloud compute instances stop flask-demo
$ export SCOPES=https://www.googleapis.com/auth/firebase.database,https://www.googleapis.com/auth/userinfo.email,https://www.googleapis.com/auth/cloud-platform
$ gcloud compute instances set-service-account flask-demo \
    --scopes $SCOPES
$ gcloud compute instances start flask-demo
```

还可以通过将`--scopes`标志传递给实例创建命令来创建具有必要范围的 VM。这可以帮助我们避免许多额外的步骤。

## 部署到 Google 计算引擎:第二步

现在，我们已经准备好在云中进行另一次测试。像往常一样启动 Flask 应用程序，并尝试发送一些请求。这一次一切都会好的。重新配置的 VM 中的 ADC 应该使所需的作用域在 OAuth2 令牌中可用。

您可能还对从远程机器向超级英雄 web 服务发送请求感兴趣。要做到这一点，我们需要做两件事。

*   将 Flask web 服务绑定到虚拟机的公共网络接口。这可以通过向烧瓶启动命令传递一个标志来实现。

```
(vm) $ flask run --host=0.0.0.0
```

*   从虚拟机的防火墙暴露烧瓶 HTTP 端口(5000)。这是通过本地开发环境中的`gcloud`工具来完成的。

```
$ gcloud compute firewall-rules create open-flask-rule --allow tcp:5000 --source-tags=flask-demo --source-ranges=0.0.0.0/0
```

现在，使用 VM 实例的公共 IP 地址，我们可以从任何地方访问云中的超级英雄 web 服务。

## 结论

我们在 Google Compute Engine 中成功开发并部署了一款服务器端 Firebase 应用。我们的是一个简单的基于 Python 和 Flask 的 web 服务，但是它可以是任何东西。可能性包括:

*   一个 Python Scikit-Learn 脚本，定期根据 Firebase 数据训练模型
*   构建在云 Firestore 之上的 Java JAX-RS API
*   用于批量发送 FCM 通知的 Node.js Express 服务器应用程序
*   一个独立的 Go 二进制文件，可以增量更新用户的权限

我们在这个例子中经历的开发和部署过程非常简单。只有两件事需要注意，这两件事都与我们使用 ADC 有关:

1.  在与 Firebase 数据库相同的 GCP 项目中启动 VM 实例。这就是我们最初使用 ADC 的原因。
2.  将所需的 Firebase OAuth2 作用域分配给 VM 实例。这是必需的，以便虚拟机上的 ADC 可以授权我们的代码进行的 Firebase 交互。

我很快会写一篇关于将这个应用程序部署到 Google App Engine 的帖子。您将会看到，由于更高的抽象级别，部署过程变得更加简单。同时，你可以在 [GitHub](https://github.com/hiranya911/firecloud/tree/master/python/flask-on-gce) 上找到与这个迷你项目相关的所有资源和支持材料。

我希望这能让您了解使用 Firebase Admin SDKs 可以构建什么类型的东西，以及如何在 Google Cloud 中部署这样的应用程序。一如既往，我欢迎评论和问题。我也很想知道你打算用 Firebase 和 Google Cloud 构建什么。