# 代码烹饪:Kubernetes

> 原文：<https://medium.com/google-cloud/code-cooking-kubernetes-e715728a578c?source=collection_archive---------0----------------------->

欢迎来到克里斯蒂昂大厨的代码烹饪！

今天我们要准备一个美味的网络汉堡。它将以 HTML 为基础，由 Kubernetes 在 nginx 上提供服务。汉堡通常与快餐联系在一起，所以我们要确保它可以立即食用。

![](img/843a86c8fb84bd2114331eeabf2feef9.png)

博克！博克！博克！

让我们从收集我们的原料和工具开始。我们需要在本地厨房安装这些工具:

*   [码头工人](https://www.docker.com/products/overview)
*   [gcloud](https://cloud.google.com/sdk/docs/quickstarts)
*   [库贝克特尔](https://cloud.google.com/container-engine/docs/quickstart#install_the_gcloud_command-line_interface)
*   [ab](https://httpd.apache.org/docs/2.4/programs/ab.html)
*   [jq](https://stedolan.github.io/jq/download/)

我假设您对这些工具至少有一点经验。如果没有:阅读一些文档，熟悉一些基础知识，或者如果你想冒险的话，就开始吧！

现在，我最喜欢的服务代码的方式是谷歌云平台。为了做准备，我在 console.cloud.google.com 上创建了一个名为代码烹饪的项目。今天我想用 Kubernetes。Kubernetes 将为我们管理码头集装箱。让我们开始构建一个不错的集群:

```
gcloud container clusters create code-cooking
```

这给了我们 3 个来自谷歌的服务器，我们可以在上面运行我们的汉堡应用程序。

当这些全新的服务器预热时，我们准备代码。为此，我们创建了`static/index.html`并按了几个键，使它看起来像这样:

```
<!DOCTYPE html>
<html>
  <h1>Best burger in town!</h1>
</html>
```

那看起来不是已经很美味了吗？不，不是的！你不能像那样提供生食。让我们把它放在 nginx 中。取一个新文件，命名为`nginx.conf`，然后输入:

```
server {
    listen 80;
    server_name localhost; location / {
        root /usr/share/nginx/html;
        index index.html; expires 1h;
        add_header Cache-Control "public";
    }
}
```

当我们处于超级打字模式时，让我们也创建一个`Dockerfile`:

```
FROM nginx
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY static /usr/share/nginx/html
```

哇，我们已经有一个集群和三个文件:`Dockerfile`、`nginx.conf`和`static/index.html`。进展得太快了。

进入下一部分。是时候让 Docker 做它的事情了:

```
docker build -t eu.gcr.io/code-cooking/burger:0.1 .
```

这使得 Docker 为我们建立了一个容器，里面有我们的汉堡应用程序。我们用名称和版本标记容器，这样我们以后就能找到它。

让它煮 3 秒钟，然后把它放入 Google 容器注册表:

```
gcloud docker -- push eu.gcr.io/code-cooking/burger:0.1
```

好吃，你闻到了吗？我相信你的客人已经迫不及待想看看你的最新作品了。我们可以使用我们的奇特工具 kubectl 来设置我们的服务器:

```
kubectl run burger --image=eu.gcr.io/code-cooking/burger:0.1 --port=80 --replicas=3kubectl expose deployment burger --port=80 --type=LoadBalancer
```

这将创建我们的 burger 应用程序的 3 个副本，在我们的集群上运行它们，并通过端口 80 上的负载平衡器向外界展示它们。

启动你的汉堡应用程序会非常快，但是第一次创建一个负载平衡器可能需要一分钟左右。请继续关注我们的汉堡服务，看看那个懒惰的负载平衡器何时最终获得外部 IP:

```
kubectl get service burger --watch
```

一旦开始，你就可以上菜了！记住总是先品尝你的作品:

```
ab -n 1000 -c 10 http://<the external ip of the burger service>/
```

在这种情况下，我在大约 40 毫秒内得到我的汉堡。

但是等等，我们可以做得更好。我喜欢旅行，当我旅行时，有时我想吃汉堡。我刚刚制作的这个汉堡服务器碰巧在欧洲，因为我在那里创建了集群。但是当我在美国的时候，我不想一路回到欧洲去买汉堡。如果我们也能在那边吃到这些美味的汉堡呢？让我告诉你:我们可以做到这一点！

首先，让我们来看看世界另一端的人们为了得到我的汉堡都经历了什么。在俄勒冈州和日本制造一些机器:

```
gcloud compute instances create oregon-test --zone us-west1-a
gcloud compute instances create japan-test --zone asia-northeast1-a
```

宋承宪合二为一:

```
gcloud compute ssh japan-test --zone asia-northeast1-a
```

和`sudo apt-get install apache2-utils -y`来安装`ab`。

让我们再次运行我们的`ab`测试，看看味道如何:

```
ab -n 1000 -c 10 http://<the external ip of the burger service>/
```

天哪，这太可怕了:俄勒冈州需要 279 毫秒，而日本需要 4.59 毫秒。这只是一个汉堡！想象一下，对于那些穷人来说，得到一个 HTML 汉堡、一份 CSS 开胃菜、一份图片沙拉和一大块 JS 甜点是什么感觉？恐怖啊！

作为一个好厨师，我知道我不能让我的客人等那么久。有几个选项可以解决这个问题，因为我们讨论的是静态内容，最简单的方法是添加一个 CDN。开始了。

我们首先删除我们的普通负载平衡器，并用不同的服务替换它:

```
kubectl delete svc burger
kubectl expose deployment burger --target-port=80 --type=NodePort
```

这允许我们将它连接到入口。入口的好处是你可以为它启用[谷歌 CDN](https://cloud.google.com/cdn/) ，这使得一切都非常快。

为了创建入口，我们创建了一个名为… `ingress.yaml`的文件！我们在里面放了这个:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: burger-ingress
spec:
  backend:
    serviceName: burger
    servicePort: 80
```

在集群上设置它:`kubectl create -f ingress.yaml`，像以前一样，我们可以观察它，直到它有一个地址:`kubectl get ing --watch`

入口有点像我们之前看到的负载平衡器，除了入口使用 [http 负载平衡器](https://cloud.google.com/compute/docs/load-balancing/http/)而不是[网络负载平衡器](https://cloud.google.com/compute/docs/load-balancing/network/)。

现在是时候添加神奇的成分了:

```
BACKEND=$(kubectl get ing burger-ingress -o json | jq -j '.metadata.annotations."ingress.kubernetes.io/backends"' | jq -j 'keys[0]')gcloud compute backend-services update $BACKEND --enable-cdn
```

Tada！这是我的大招，你在其他任何一本食谱上都找不到。基本上，它查看我们刚刚创建的入口，并使用`[jq](https://stedolan.github.io/jq/)`(伟大的工具 btw)对其进行分割，因此我们最终只得到[后端](https://cloud.google.com/compute/docs/load-balancing/http/backend-service)。我们告诉`gcloud`在后端启用 CDN，我们就完成了。

让我们来看看不同之处:

```
ab -n 1000 -c 10 http://<the address of our burger ingress>/
```

现在在欧洲，我只需 31 毫秒就能拿到汉堡。如果我们在日本的虚拟机上进行同样的测试，我们可以在 34 毫秒内完成，这比我们之前看到的 459 毫秒要好得多。在俄勒冈州的虚拟机上进行测试，我们甚至可以在 1 毫秒内获得服务！

由于谷歌的 CDN 服务器现在获得了所有的点击率，我们最初的 3 个服务器将很难再获得任何流量。这意味着我们可以将它们缩小到 1:

```
kubectl scale deployment burger --replicas=1
```

如果你只记得这个节目中的三件事，记住这个:

*   你可以用 Kubernetes 做美味的东西
*   汉堡需要快速供应
*   如果您使用入口，您可以启用 Google CDN

现在你知道了。祝你好运！