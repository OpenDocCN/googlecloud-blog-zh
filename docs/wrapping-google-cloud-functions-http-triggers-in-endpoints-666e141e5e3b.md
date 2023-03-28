# 在端点中包装 Google Cloud 函数 HTTP 触发器

> 原文：<https://medium.com/google-cloud/wrapping-google-cloud-functions-http-triggers-in-endpoints-666e141e5e3b?source=collection_archive---------1----------------------->

随着谷歌云功能生态系统的发展，有许多有用的功能变得可用。然而，锁定 [GCF HTTP 函数](https://cloud.google.com/functions/docs/writing/http)仍然有点困难。例如，您可能希望添加身份验证并保护这些功能免受来自开放互联网的攻击。另一方面，[谷歌云端点](https://cloud.google.com/endpoints/docs/)已经存在了一段时间，并且是专门为受保护的 HTTP 访问而设计的。您可以使用 API 密钥来限制端点，将它们放在负载平衡器的后面以托管在一致的域上，并使用云盔甲来保护它们。这篇文章将演示如何将 Nodejs HTTP 云函数导入到 App Engine 上托管的端点 API 中。

**将云功能导入应用引擎**
en points API 可以托管在[多个不同的计算选项](https://cloud.google.com/docs/choosing-a-compute-option)上。这篇文章将解释如何在 App Engine 上运行一个例子。App Engine Nodejs 运行时使用 [Express](https://expressjs.com/) ，这是一个流行的 web 框架，它不是特别固执己见，并使将 GCF 函数导入 App Engine 应用程序变得简单。GCF 使用的 JavaScript 模块方法使得无需修改即可轻松移植代码。

遵循 [GCF 快速入门](https://cloud.google.com/functions/docs/quickstart)中的说明，包括[Google cloud platform/nodejs-docs-samples](https://github.com/GoogleCloudPlatform/nodejs-docs-samples)的克隆。快速入门导出函数 helloGET，该函数将“Hello World”文本写入 HTTP 响应对象。快速入门的代码包含在 nodejs-docs-samples/functions/hello world 目录中。注意，App Engine 中 Node.js 的
[快速入门包含在同一个 GitHub repo 中。](https://cloud.google.com/appengine/docs/standard/nodejs/quickstart)

从 GoogleCloudPlatform 目录开始，将 GCF 文件复制到
App Engine 标准 helloworld 示例目录:

```
cp functions/helloworld/index.js appengine/hello-world/standard/.
```

对 app.js 中的 App Engine 代码进行一些更改，以便按照以下代码行导入函数:

```
var mygcf = require(‘./index’); 
```

并将 app.js 中的/ route 替换为:

```
app.get(‘/’, (req, res) => {
 res.status(200);
 mygcf.helloGET(req, res);
 res.end();
});
```

安装一些 HTTP 触发器不需要的额外的东西，但是在 GCF Quickstart 的其他地方使用了
,所以我们不需要改变 GCF 文件:

```
cd appengine/hello-world/standard/
npm install -save [@google](http://twitter.com/google)-cloud/debug-agent
npm install -save pug
npm install -save safe-buffer
```

尝试在本地运行应用程序

```
npm start
```

检查它是否在本地正常运行:

```
curl [http://localhost:8080](http://localhost:8080)
```

部署到应用引擎标准:

```
gcloud app deploy
```

检查应用程序是否正常工作

```
gcloud app browse
```

**向云端点导入云函数**
让我们对 App Engine 端点的例子做同样的尝试。按照[App Engine Flexible
环境](https://cloud.google.com/endpoints/docs/openapi/get-started-app-engine#node)中的
说明开始使用端点。用 openapi-appengine.yaml 文件中的项目 ID 替换字符串 YOUR-PROJECT-ID，用完整的主机名替换 app.yaml 文件中的 ENDPOINTS-SERVICE-NAME。

从 GoogleCloudPlatform 目录开始，复制文件:

```
cp functions/helloworld/index.js endpoints/getting-started/.
cd endpoints/getting-started
```

根据本[要点](https://gist.github.com/alexamies/f2cda3c2db21dd4603d896a3b9079ab7)，对 App Engine 代码进行一些更改以导入函数。具体来说，在 app.js 的顶部附近添加下面一行来导入您的 GCF 函数:

```
const mygcf = require(‘./index’);
```

用这个实现替换 echo [Express route](https://expressjs.com/en/guide/routing.html) :

```
app.post(‘/echo’, (req, res) => {
 res.status(200).json({ message: mygcf.helloGET(req, res) }).end();
});
```

安装额外的节点模块

```
npm install -save [@google](http://twitter.com/google)-cloud/debug-agent
npm install -save pug
```

部署端点服务和后端:

```
gcloud endpoints services deploy openapi-appengine.yaml
gcloud app deploy
```

通过向 API 端点发送请求来测试结果:

```
PROJECT_ID={Your project}
ENDPOINTS_HOST=${PROJECT_ID}.appspot.com
ENDPOINTS_KEY={Your key}
curl — request POST \
 — header “content-type:application/json” \
 — data ‘{“message”:”hello world”}’ \
 “${ENDPOINTS_HOST}/echo?key=${ENDPOINTS_KEY}”
```

这很容易，可能比用 GCF 编写自己的认证和 DDoS 保护要容易得多。

修改端点 API

到目前为止，该解决方案利用了端点接口的 API。这可以很容易地改变，以更好地匹配您正在移植的功能的语义。用 gist 中的版本替换您的 openapi-appengine.yaml 和 app.js 文件，然后重新部署端点服务和后端:

```
gcloud endpoints services deploy openapi-appengine.yaml
gcloud app deploy
```

向新的 API 发送请求

```
curl --request GET \
    --header "content-type:application/json" \
    "${ENDPOINTS_HOST}/helloget?key=${ENDPOINTS_KEY}"
curl --request POST \
    --header "content-type:application/json" \
    --data '{"name":"Tester"}' \
    "${ENDPOINTS_HOST}/hellohttp?key=${ENDPOINTS_KEY}"
```

新的端点 API 现在更适合 GCF 示例:
对/helloget 的 GET 请求不带参数，对/hellohttp 的 POST 请求带一个名为‘name’的参数。