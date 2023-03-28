# 云运行和秘密管理器

> 原文：<https://medium.com/google-cloud/cloud-run-and-secret-manager-3c5d43a72e87?source=collection_archive---------0----------------------->

![](img/0aa79267e647aae98080dfe9dfc5d3d1.png)

[Cloud Run](https://cloud.google.com/run) 是一个了不起的产品——一个完全无服务的产品，它代表你启动和管理容器。它处理伸缩性和并发性、安全性等等。它可能最类似于亚马逊的 [Fargate](https://aws.amazon.com/fargate/) 产品，但也与 [Lambda](https://aws.amazon.com/lambda) 重叠。

我在玩 GCP [秘密经理](https://cloud.google.com/secret-manager)，我想我会分享如何在使用云运行时利用它。最后还贴了一些不错的参考博文。

示例代码使用了 [Clojurescript](https://clojurescript.org/) 。我是函数式编程的忠实粉丝，一般来说是 Clojure。如果您不熟悉 Clojure，它是一个 LISP 派生程序，运行在 JVM 之上。Clojurescript 将 transpiles 转换成 Javascript，因此可以利用社区可用的巨大节点包。看看 [Shadow-CLJS](https://shadow-cljs.github.io/docs/UsersGuide.html) ，它使得将 javascript 库导入 Clojurescript 成为一个简单的过程。

# 你好，世界

我们将创建一个简单的 web 服务器。它将从秘密经理那里读入一个秘密。如果失败，它将寻找一个名为`TARGET`的环境变量。如果失败，它将简单地用`Hello, World!`来响应。

云运行允许将环境变量传递到您的运行容器中。与 secret manager 没有直接集成，所以必须通过 API 调用。有趣的是，[云构建](https://cloud.google.com/build/docs/securing-builds/use-secrets)，*允许秘密作为环境变量被注入。这简化了事情，尽管可以说环境变量在涉及敏感数据时是危险的工具；因为它们很容易通过日志或内存转储暴露出来。尽管如此，如果你想要的东西需要很少的编码，云构建与 Secret Manager 的集成是一条路要走。*

# *部署容器*

*我在本地构建了我的容器，并把它放到了谷歌容器注册中心。这简单而有效，但我会把[云构建](https://cloud.google.com/build)作为 CI/CD 系统，用于比简单演示更高级的东西。*

```
*$ gcloud run deploy hello-secret --image gcr.io/nicks-playground-3141/hello-secret:0.4 --platform managed
Deploying container to Cloud Run service [hello-secret] in project [nicks-playground-3141] region [us-central1]
✓ Deploying... Done.
  ✓ Creating Revision...
  ✓ Routing traffic...
Done.
Service [hello-secret] revision [hello-secret-00010-qit] has been deployed and is serving 100 percent of traffic.
Service URL: https://hello-secret-hrhyswd3ya-uc.a.run.app# Test against URL
$ curl https://hello-secret-hrhyswd3ya-uc.a.run.app/
Hello, World!*
```

# *部署时将环境变量注入到容器中*

```
*$ gcloud run deploy hello-secret --set-env-vars TARGET=GALAXY \
--image gcr.io/nicks-playground-3141/hello-secret:0.4 --platform managed# Test aginst URL
$ curl https://hello-secret-hrhyswd3ya-uc.a.run.app/
Hello, GALAXY!*
```

# *创建秘密*

*让我们在秘密管理器中创建一个秘密:*

```
*$ gcloud secrets create TARGET \
    --replication-policy="automatic"$ echo -n "Universe" | \
    gcloud secrets versions add TARGET  --data-file=-# Verify the secret
$ gcloud secrets versions access 1 --secret="TARGET"
Universe*
```

# *将 IAM 权限更新为从 Secret Manager 中读取*

*我第一次尝试时没有这样做，当然失败了。您可以为您的云运行容器创建一个具有适当权限的新服务帐户。或者，您可以扩展 Cloud Run 使用的[默认](https://cloud.google.com/run/docs/securing/service-identity)服务帐户的功能——我在这里就是这么做的。这**不**被认为是最佳实践，因为你不希望所有的容器都被允许读取所有的秘密。然而，对于这个项目，这是一个可以接受的选择。*

```
*$ gcloud iam service-accounts add-iam-policy-binding nicks-playground-3141 \
--member="serviceAccount:659824402950-compute@developer.gserviceaccount.com" \
--role="roles/secretmanager.secretAccessor"*
```

# *重新启动服务，并测试*

```
*$ curl https://hello-secret-hrhyswd3ya-uc.a.run.app
Hello, Universe!*
```

# *排除故障*

*GCP 日志控制台是惊人的，强大的眼睛糖果。但是，如果您想简单地跟踪日志，因为您的程序会将日志发送到 STDOUT，您可以执行以下命令:*

```
*gcloud beta logging tail "resource.type=cloud_run_revision AND \
resource.labels.service_name=hello-secret AND \
severity>=DEFAULT" \
--format="default(timestamp,resource["labels"]["service_name"],textPayload)"*
```

# *Clojurescript*

*对于那些想看看 Clojurescript 代码的人来说，这个库位于 Github [这里](https://github.com/nbrandaleone/hello-secret)。*

```
*(ns server.main
  (:require
    ["@google-cloud/secret-manager" :as Secret]
    ["express" :as express]
    [taoensso.timbre :as log]))

(defn get-word []
  "determine the TARGET value, to follow 'hello' if secret-manager fails"
  (let [t (get (env) "TARGET" "World")] t))

(defn secret-api []
  "Calls GCP Secret Manager, using a JS promise
   Use an atom to store state. Probably shouldn't, but it make things
   a bit easier to test."
  (let [client (Secret/SecretManagerServiceClient.)]
    (-> (.accessSecretVersion client (clj->js {:name mysecret}))
      (.then (fn [name]
               (let [payload (first name)] 
                 (reset! word (str (.. payload -payload -data)))))
             (fn [] (do
                      (reset! word (get-word))
                      (log/debug "Secret Manager Promise rejected")))))))*
```

*   *[谷歌云秘密管理器:储存秘密的好地方](https://www.inthepocket.com/blog/google-cloud-secret-manager-a-better-place-for-your-secrets)*
*   *[Google Cloud 上的 Secret Manager Libraries 带来了更少的秘密](https://dev.to/googlecloud/serverless-mysteries-with-secret-manager-libraries-on-google-cloud-3a1p)*
*   *[Secret Manager:不改变代码提高云运行安全性](/google-cloud/secret-manager-improve-cloud-run-security-without-changing-the-code-634f60c541e6)*