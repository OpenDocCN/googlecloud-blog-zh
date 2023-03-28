# 我如何封装我的简历 API

> 原文：<https://medium.com/google-cloud/how-i-containerised-my-resume-api-91bb2d2f92f4?source=collection_archive---------0----------------------->

…或者**(这个)在谷歌云平台上运行 Docker 和 Kubernetes 的白痴指南**

由于我目前正在寻找有趣的新机会，并且对 api 特别感兴趣，几周前我为自己的简历创建了一个 API，并在这里写道:[https://medium . com/@ Mario menti/everything-is-a-API-including-me-and-my-CV-674 ea 433 f 283](/@mariomenti/everything-is-an-api-including-me-and-my-cv-674ea433f283)

从那以后，我花了一些时间研究微服务和容器(即 Kubernetes 和 Docker)，因为我觉得我想更熟悉其中的一些概念。尽管我几乎不知道自己在做什么，但和以往一样，我认为描述我如何将我的单片 API 转移到独立的微服务中会很有趣，所有这些服务都运行在 [Kubernetes](https://kubernetes.io/) 和[谷歌云平台](https://cloud.google.com/)上。

![](img/41e894fef42f72e393f3d2f9b99e7dd9.png)

当我开始写这篇文章时，我对容器的概念完全陌生，所以这将有望作为一个有用的白痴指南/类似情况下的人的起点——但同时，我几乎肯定以一种不太理想的(如果不是完全丑陋甚至错误的)方式做事情，所以如果任何专家阅读了这篇文章并感到恶心，请发表评论并让我知道我错在哪里！

我为什么要这么做？问得好——在我的特殊情况下，可能没有那么多原因，因为我的 CV API 几乎没有在负荷下呻吟(如果是的话，我就没有时间写这样的文章了！).但是想象一下这是一个生产级的 API:目前，所有的东西，API 的每个端点以及加载简历数据和 API 令牌的代码都包含在一个 Go 程序中。
如果一个端点需要更改，这意味着重新编译和重新部署整个东西。比方说，有一个 API 端点(例如向我发送电子邮件或 SMS 消息的“/contact”端点)突然变得非常流行，API 开始陷入困境，为了在现有设置中进行扩展，我必须增加运行 API 的整个计算引擎实例的容量。
相比之下，在容器化的世界中，因为每个 API 端点都作为自己的微服务运行，所以我将能够独立地扩展“/contact”端点，同时保持 API 的所有其他方面不变。
所以你突然对你的应用程序的不同部分有了更细粒度的控制。
类似地，如果一个 API 端点需要更新，它可以独立于 API 的其余部分进行部署，而不会有任何停机时间。此外，Kubernetes 具有自我修复功能，因此，如果运行某个应用程序的所谓 pod 出现故障，Kubernetes 会自动替换它们。

所以，废话不多说，前提条件(我正在使用 Ubuntu Linux 笔记本电脑，细节可能会因您使用的操作系统而异，所以我链接到相当通用的文档):

*   安装[对接器](https://docs.docker.com/engine/installation/)
*   安装 [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) ，它允许您在 Kubernetes 上部署和管理应用程序
*   在[谷歌云平台](https://cloud.google.com/)上拥有/获得一个账户(我们使用 GCP 的[容器引擎](https://cloud.google.com/container-engine/docs/)来运行我们的 Kubernetes 应用)
*   安装 [Google Cloud SDK，](https://cloud.google.com/sdk/)或者使用 [Google Cloud Shell](https://cloud.google.com/shell/)

好吗？很好。首先要做的是创建一个 Kubernetes 集群来运行我们的应用程序。这里我使用默认设置，创建一个 3 机集群:

```
$ gcloud container clusters create marioapi-cluster
```

这将需要几分钟时间——创建集群后，通过键入以下内容，使用您的 Google 帐户验证 Google Cloud SDK:

```
$ gcloud auth application-default login
```

现在让我们后退一步，看看我现有的 API 设置，以及我们希望如何将其转换为一组微服务。它非常简单，包含以下几个部分:

*   一个读取我的 JSON 格式简历的功能
*   该函数读取包含有效 API 令牌数据的 JSON 文件，并检查传递给 API 端点的令牌的有效性
*   处理不同 API 端点的 gorilla/mux 处理程序:

```
r.HandleFunc("/full", api.FullCVHandler).Methods("GET")
r.HandleFunc("/summary", api.SummaryHandler).Methods("GET")
r.HandleFunc("/contact", api.ContactHandler).Methods("GET")
r.HandleFunc("/contact", api.ContactPostHandler).Methods("POST")
r.HandleFunc("/experience", api.ExperienceHandler).Methods("GET")
r.HandleFunc("/experience/{id}", api.ExperienceIdHandler).Methods("GET")
r.HandleFunc("/projects", api.ProjectsHandler).Methods("GET")
r.HandleFunc("/projects/{id}", api.ProjectsIdHandler).Methods("GET")
r.HandleFunc("/tags", api.TagsHandler).Methods("GET")
r.HandleFunc("/tags/{tag}", api.TagsTagHandler).Methods("GET")
```

所有这些都包含在 one Go 程序文件中。因此，为了区分开来，我要做的是创建这些独立的微服务:

*   一个简历服务器，它加载我的简历并以 JSON 格式提供，
*   令牌服务器，它加载有效的 API 令牌定义，并根据客户端在 API 调用中发送的内容检查它们，
*   每个 API 端点(例如完整端点、汇总端点、接触端点等)的单独微服务。),
*   最后是 nginx (web 服务器)服务，它处理负载平衡并将请求传递给相关的内部服务。

让我们先看看简历服务器。这是一个简单的(也是不现实的)问题——一个“真正的”API 当然会有某种类型的数据存储/数据库，但是出于演示的目的，我只是从一个 JSON 文件中读取我的简历/CV 数据。

```
package main

import (
        "fmt"
        "io/ioutil"
        "log"
        "net/http"
)

func LoadResume() string {

        file, e := ioutil.ReadFile("/resume.json")
        if e != nil {
                return "{}"
        }
        return string(file)
}

func serve(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, LoadResume())
}

func main() {
        http.HandleFunc("/", serve)
        err := http.ListenAndServe(":80", nil)
        if err != nil {
                log.Fatal("ListenAndServe: ", err)
        }
}
```

我们接下来需要为此创建一个 docker 容器，这样我们就可以将它上传到 Kubernetes。同样，Dockerfile 文件本身也很简单:

```
FROM golang:1.6-onbuild
COPY resume.json /
```

…COPY 行确保我们将 resume.json 文件复制到容器中，以便 pod 应用程序可以读取它。像这样构建 docker 容器:

```
$ docker build -t gcr.io/<YOUR-GOOGLE-CLOUD-CLOUD-PROJECT>/resume-server:1.0 .
```

您可以通过键入…来获得应该用来代替<your-google-cloud-project>的值</your-google-cloud-project>

```
$ gcloud config get-value project
```

每当你在这里的任何例子中看到<your-google-cloud-project>时，用你实际的 Google Cloud 项目名称替换。一旦 docker 容器构建完成，我们需要将其推送到 Google Cloud:</your-google-cloud-project>

```
$ gcloud docker -- push gcr.io/<YOUR-GOOGLE-CLOUD-CLOUD-PROJECT>/resume-server:1.0
```

接下来，我们希望将应用程序部署到 Kubernetes。为此，我们为应用程序创建了一个 deployment.yaml 文件:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: resumeserver
spec:
  replicas: 2
  template:
    metadata:
      labels:
        name: resumeserver-pods
    spec:
      containers:
      - image: gcr.io/<YOUR-GOOGLE-CLOUD-CLOUD-PROJECT>/resume-server:1.0
        name: resumeserver-container
        imagePullPolicy: Always
        ports:
        - containerPort: 80
          name: http-server
```

在本例中，我们希望始终运行 2 个 resumeserver 实例，但是如果我们希望容错能力更强，我们可以指定更多的副本。正如我上面提到的，Kubernetes 将确保至少有 X 一直在运行，并根据需要重新启动它们。

部署应用程序:

```
$ kubectl apply -f deployment.yaml
```

这在 Kubernetes 中创建了 2 个正在运行的 pod，您可以通过这些命令查看详细信息——第一个命令列出了您的集群上的部署，第二个命令列出了 pod(正如我们指定的，有 2 个 pod 正在为我们的 resumeserver 部署运行)。

```
$ kubectl get deployments
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
resumeserver   2         2         2            2           2m
$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
resumeserver-3871548837-8tf9w   1/1       Running   0          1m
resumeserver-3871548837-ks0lk   1/1       Running   0          2m
```

到目前为止还不错，但是为了能够与这些正在运行的 pod 通信，我们需要创建一个命名服务。为此，我们使用这个服务。

```
apiVersion: v1
kind: Service
metadata:
  name: resumeserver
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    name: resumeserver-pods
```

这定义了一个名为“resumeserver”的服务，它包括所有运行名称为“resumeserver-pods”的 pod(我们在上面的 deployment.yaml 文件中指定了这一点)。要创建服务，请运行…

```
$ kubectl create -f service.yaml
```

我们现在可以检查并查看我们的服务是否正在运行:

```
$ kubectl get svc
NAME           CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes     10.23.240.1     <none>        443/TCP   13m
resumeserver   10.23.243.143   <none>        80/TCP    33s
```

您可以看到 resumeserver 就在那里，并且已经分配了一个内部 IP 地址。好的一面是 Kubernetes 有一个内置的 DNS 服务(更多内容见下文)，所以从现在开始，当我们想从我们集群中的不同 pod 连接到它时，我们将能够通过名称来引用我们的 resumeserver 服务。

好了，我们现在已经启动并运行了简历服务器部分，设置令牌服务器也非常类似，所以我不会在这里赘述，但是[所有的文件都在 Github](https://github.com/mmenti/marioapi-k8s-demo) 上。本质上，对于我们想要创建的每个服务，我们遵循以下步骤:

*   创建 docker 容器
*   将容器推送到谷歌云
*   运行/应用部署
*   创建服务

因此，按照上述步骤创建令牌服务器后，我们会看到 resumeserver 和令牌服务器都在我们的群集上运行:

```
$ kubectl get svc
NAME           CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes     10.23.240.1     <none>        443/TCP   30m
resumeserver   10.23.243.143   <none>        80/TCP    17m
tokenserver    10.23.244.115   <none>        80/TCP    7s
```

接下来，让我们创建一个 API 端点。最简单的是“/full”端点，它基本上只是以 JSON 格式返回整个简历/CV。其他一些端点稍微复杂一些(但不多，这是一个非常简单的演示:)

如果您查看 [full.go](https://github.com/mmenti/marioapi-k8s-demo/blob/master/full/full.go) ，可以看到我们指的是令牌服务器和 resumeserver:

```
func LoadResume() (resumeData Resume, loadedOk bool) {

        var res Resume

        rsp, err := http.Get("http://resumeserver")
        if err != nil {
                return res, false
        }
        defer rsp.Body.Close()
        bodyByte, err := ioutil.ReadAll(rsp.Body)
        if err != nil {
                return res, false
        }

        err = json.Unmarshal(bodyByte, &resumeData)
        if err != nil {
                return res, false
        }
        return resumeData, true

}func serve(w http.ResponseWriter, r *http.Request) {

        // check token, load resume, return relevant part(s)
        token := r.FormValue("token")
        rsp, err := http.Get("http://tokenserver?token=" + token)
        if err != nil {
                WriteApiError(w, 110, "Error checking token from token server")
                return
        }
        defer rsp.Body.Close()
        bodyBytes, err := ioutil.ReadAll(rsp.Body)
[...]
```

因为我们创建了名为“resumeserver”和“tokenserver”的服务，所以我们现在可以使用它们的名称(例如“http://resumeserver”)来访问它们— Kubernetes 的内部 DNS 服务会处理这些问题。注意这只是内部的，这些舱不能从外部世界的任何地方访问(我们也不希望它们被访问)，只能从我们集群中的其他舱访问。在这个演示中，我们通过 http 访问它们，但是如果它们是(比如)redis 或 mySql 服务器而不是 HTTP 服务器，它将以几乎相同的方式工作。

为了创建完整端点服务，我们再次经历相同的步骤:

*   创建 docker 容器:

```
$ docker build -t gcr.io/<YOUR-GOOGLE-CLOUD-CLOUD-PROJECT>/full-endpoint:1.0 .
```

*   将容器推送到 Google Cloud:

```
$ gcloud docker -- push gcr.io/<YOUR-GOOGLE-CLOUD-CLOUD-PROJECT>/full-endpoint:1.0
```

*   创建/应用部署:

```
$ kubectl apply -f deployment.yaml
```

*   创建服务:

```
$ kubectl create -f service.yam
```

完成后，我们可以在服务列表中看到新的端点(完整端点):

```
$ kubectl get svc
NAME            CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
full-endpoint   10.23.241.128   <none>        80/TCP    37s
kubernetes      10.23.240.1     <none>        443/TCP   45m
resumeserver    10.23.243.143   <none>        80/TCP    33m
tokenserver     10.23.244.115   <none>        80/TCP    15m
```

每个 API 端点的详细代码略有不同(您可以在 [Github](https://github.com/mmenti/marioapi-k8s-demo) 中看到所有文件)，但我们对每个端点都经历了完全相同的过程，直到我们的集群包含每个 API 端点的服务:

```
$ kubectl get svc
NAME                  CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
contact-endpoint      10.23.244.122   <none>        80/TCP    4m
experience-endpoint   10.23.252.142   <none>        80/TCP    2m
full-endpoint         10.23.241.128   <none>        80/TCP    14m
kubernetes            10.23.240.1     <none>        443/TCP   59m
projects-endpoint     10.23.244.164   <none>        80/TCP    1m
resumeserver          10.23.243.143   <none>        80/TCP    46m
summary-endpoint      10.23.249.134   <none>        80/TCP    8m
tags-endpoint         10.23.246.242   <none>        80/TCP    7s
tokenserver           10.23.244.115   <none>        80/TCP    29m
```

我们还可以检查我们的部署和 pod(在 resumeserver 服务之后，我只指定了每个服务 1 个副本，但这当然可以根据需要进行扩展):

```
$ kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
contact-endpoint      1         1         1            1           5m
experience-endpoint   1         1         1            1           3m
full-endpoint         1         1         1            1           15m
projects-endpoint     1         1         1            1           2m
resumeserver          2         2         2            2           56m
summary-endpoint      1         1         1            1           9m
tags-endpoint         1         1         1            1           1m
tokenserver           1         1         1            1           30m$ kubectl get pods       
NAME                                   READY     STATUS    RESTARTS   AGE
contact-endpoint-3271379398-6614g      1/1       Running   0          5m
experience-endpoint-4000525602-3qsw6   1/1       Running   0          3m
full-endpoint-960439747-wrch5          1/1       Running   0          15m
projects-endpoint-1973496552-q1p38     1/1       Running   0          2m
resumeserver-3871548837-8tf9w          1/1       Running   0          54m
resumeserver-3871548837-ks0lk          1/1       Running   0          56m
summary-endpoint-4173081044-2lf30      1/1       Running   0          8m
tags-endpoint-3958056375-vtpdv         1/1       Running   0          1m
tokenserver-4152829013-12w2j           1/1       Running   0          30m
```

**嘘…想知道一个秘密吗？**在我们进入最后一部分(实际上是将所有这些服务公开给外界)之前，让我们来看看“/contact”端点。这个比其他的更有趣(TBH 没有说太多)，因为它包含了一些通过 Twilio 发送 SMS 消息和通过 SendGrid 发送电子邮件的代码。你说没那么有趣——但这里有趣的部分是弄清楚我们如何最好地将像 API 密钥这样的“秘密”信息传递到 Kubernetes 上的一个容器，而暴露这些机密数据的风险最小。幸运的是，Kubernetes 有一个关于[秘密的概念，](https://kubernetes.io/docs/concepts/configuration/secret/#creating-a-secret-using-kubectl-create-secret)让这个变得非常容易和安全。

为了将秘密信息传递给 Kubernets pod/deployment，我们首先创建一个包含我们的机密 API 信息的秘密。这是我的秘密:

```
apiVersion: v1
kind: Secret
metadata:
  name: contact-secrets
type: Opaque
data:
  twiliosid: <base64-encoded version of your string>
  twiliotoken: <base64-encoded version of your string>
  twiliourl: <base64-encoded version of your string>
  twilionumber: <base64-encoded version of your string>
  alertnumber: <base64-encoded version of your string>
  sendgridkey: <base64-encoded version of your string>
```

然后，我们通过运行以下命令来创建秘密:

```
$ kubectl create -f ./secrets.yaml
```

*(显然不要以任何方式公开 secrets.yaml 文件，例如，通过签入源代码控制或类似方式……)*

现在，当我们为 contact-endpoint 服务创建部署时，我们可以引用我们命名为“contact-secret”的秘密，并将这些值作为环境变量传递:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: contact-endpoint
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: contact-endpoint-pods
    spec:
      containers:
      - image: gcr.io/airy-cortex-166611/contact-endpoint:1.0
        name: contact-endpoint-container
        imagePullPolicy: Always
        ports:
        - containerPort: 80
          name: http-server
        env:
        - name: TWILIO_SID
          valueFrom:
           secretKeyRef:
            name: contact-secrets
            key: twiliosid
        - name: TWILIO_TOKEN
          valueFrom:
           secretKeyRef:
            name: contact-secrets
            key: twiliotoken
        - name: TWILIO_URL
          valueFrom:
           secretKeyRef:
            name: contact-secrets
            key: twiliourl
        - name: TWILIO_NUMBER
          valueFrom:
           secretKeyRef:
            name: contact-secrets
            key: twilionumber
        - name: ALERT_NUMBER
          valueFrom:
           secretKeyRef:
            name: contact-secrets
            key: alertnumber
        - name: SENDGRID_KEY
          valueFrom:
           secretKeyRef:
            name: contact-secrets
            key: sendgridkey
```

在我们刚刚部署的 pod 中的 contacts.go 程序中，我们可以轻松获得这些环境值:

```
var (
        // twilio (for SMS) and SendGrid (for email) config
        twilioSid    string = os.Getenv("TWILIO_SID")
        twilioToken  string = os.Getenv("TWILIO_TOKEN")
        twilioUrl    string = os.Getenv("TWILIO_URL")
        twilioNumber string = os.Getenv("TWILIO_NUMBER")
        alertNumber  string = os.Getenv("ALERT_NUMBER")
        sendGridKey  string = os.Getenv("SENDGRID_KEY")
)
```

这很棒，对吧？；)

好了，现在到了最后一部分——我们运行了所有这些服务，但是没有任何东西可以访问它们。因此，我们需要的是向外界公开的东西，能够处理我们的 API 请求，并将它们传递给相关的服务进行处理。输入 nginx(当然)。

同样，这非常简单——我们只需创建一个定制的 nginx.conf，然后创建一个使用它的 docker 容器。nginx.conf 的内容如下所示:

```
resolver 10.23.240.10 valid=5s;

upstream summary-endpoint {
    server summary-endpoint.default.svc.cluster.local;
}
upstream full-endpoint {
    server full-endpoint.default.svc.cluster.local;
}
upstream projects-endpoint {
    server projects-endpoint.default.svc.cluster.local;
}
upstream experience-endpoint {
    server experience-endpoint.default.svc.cluster.local;
}
upstream tags-endpoint {
    server tags-endpoint.default.svc.cluster.local;
}
upstream contact-endpoint {
    server contact-endpoint.default.svc.cluster.local;
}

server {
    listen 80;

    root /usr/share/nginx/html;

    location / {
        index  index.html index.htm;
    }

    location /summary/ {
        proxy_pass http://summary-endpoint/;
    }
    location /full/ {
        proxy_pass http://full-endpoint/;
    }
    location /projects/ {
        proxy_pass http://projects-endpoint/;
    }
    location /experience/ {
        proxy_pass http://experience-endpoint/;
    }
    location /tags/ {
        proxy_pass http://tags-endpoint/;
    }
    location /contact/ {
        proxy_pass http://contact-endpoint/;
    }
    location = /summary {
        proxy_pass http://summary-endpoint/;
    }
    location = /full {
        proxy_pass http://full-endpoint/;
    }
    location = /projects {
        proxy_pass http://projects-endpoint/;
    }
    location = /experience {
        proxy_pass http://experience-endpoint/;
    }
    location = /tags {
        proxy_pass http://tags-endpoint/;
    }
    location = /contact {
        proxy_pass http://contact-endpoint/;
    }

} 
```

在第一行，我们指定要使用的 DNS 解析器—在您的情况下，这可能是一个不同的值，要获得您应该使用的值，请运行:

```
$ kubectl get services kube-dns --namespace=kube-system
```

剩下的就不言自明了——每个服务都有自己的服务名，所以我们只需将不同位置的请求代理给不同的服务。

创建容器的 Dockerfile 文件:

```
FROM nginx
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

让我们构建并推动我们的容器:

```
$ docker build -t gcr.io/<YOUR-GOOGLE-CLOUD-CLOUD-PROJECT>/my-nginx:1.0 .
$ gcloud docker -- push gcr.io/<YOUR-GOOGLE-CLOUD-CLOUD-PROJECT>my-nginx:1.0
```

接下来，我们创建部署(这也与上面的服务非常相似，我想您已经开始看到一种模式了)，然后创建服务。这就是我们之前创建的服务的一个不同之处:

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
  selector:
    name: nginx-pods
```

与之前的服务定义不同，这里我们将类型指定为“负载平衡器”。这将为服务提供一个外部 IP 地址。因此，在运行“kubectl create -f service.yaml”之后，您应该会看到如下内容:

```
$ kubectl get svc
NAME                  CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
contact-endpoint      10.23.244.122   <none>          80/TCP         48m
experience-endpoint   10.23.252.142   <none>          80/TCP         46m
full-endpoint         10.23.241.128   <none>          80/TCP         58m
kubernetes            10.23.240.1     <none>          443/TCP        1h
nginx                 10.23.252.234   35.189.117.234   80:30543/TCP   1m
projects-endpoint     10.23.244.164   <none>          80/TCP         45m
resumeserver          10.23.243.143   <none>          80/TCP         1h
summary-endpoint      10.23.249.134   <none>          80/TCP         53m
tags-endpoint         10.23.246.242   <none>          80/TCP         44m
tokenserver           10.23.244.115   <none>          80/TCP         1h
```

*(外部 IP 地址可能需要一段时间才会出现，如果显示“<待定>”，只需重新运行命令，直到可以看到外部 IP 地址。)*

现在我们的 nginx 服务有了一个外部 IP 地址，我们准备向 API 发出一些请求…例如，调用/tags/golang 端点:

```
$ curl http://35.189.117.234/tags/golang?token=public
{"Projects":[{"id":1,"name":"Gigfinder bot (Facebook Messenger bot)","summary":"Wondering when and where your favourite band is playing live next? Or just want to see what gigs are on tonight in your city? Message me and I'll tell you!","url":"https://www.facebook.com/gigfinderbot/","tags":["songkick","facebook","golang","supervisord","nginx","amazon","simpledb","redis"]},{"id":2,"name":"Songkick Alert bot (Facebook Messenger bot)","summary":"Say hello to the Songkick Alerts bot on Facebook Messenger to set up your Songkick alerts. Once set up and activated, the bot will send you instant notifications on Messenger for any newly announced shows by the Songkick artists you're tracking.","url":"https://www.facebook.com/songkickalerts/","tags":["songkick","last.fm","facebook","golang","supervisord","nginx","amazon","simpledb"]},{"id":4,"name":"Slack gig attendance bot (Slack integration)","summary":"A Slack bot that checks users' Songkick event attendance and automatically shares these to a Slack channel of your choice. As an example, we have a specific #gig Slack channel where notifications of the shows/gigs that users are planning to go to are being posted as soon as a user marks their attendance on Songkick. Includes artist images retrieved via the last.fm API.","url":"https://github.com/mmenti/songslack","tags":["songkick","slack","last.fm","golang","supervisord","nginx","simpledb"]},{"id":5,"name":"Slack Gigfinder bot (Slack bot)","summary":"Directly search for upcoming shows of an artist from within Slack, powered by the Songkick API.","url":"http://blog.songkick.com/2015/11/09/slack-magic/","tags":["songkick","slack","golang","supervisord","nginx"]}],"Experience":[{"id":1,"name":"Awne","dates":"2016 - 2017","location":"London, UK","job_title":"Technical co-founder","summary":"Technical co-founder (with two others) of Awne, a personal relationship assistant.  Built a platform and API that lets us create content and deliver that content to users from one unified back-end to multiple potential delivery channels (messaging, bots, web, apps etc).\n Developed a Facebook Messenger bot to ask users daily questions, record user answers, and deliver tips and updates into their personal Awne home.\n Created prototypes/ proof-of-concepts for delivering Awne content (and collecting user data) via different channels (e.g. messaging, SMS, web, Amazon Alexa) via the same API-powered back-end/platform.\n Both API/platform and Messenger bots built using golang, running through supervisord and nginx.","tags":["golang","api","redis","wit.ai","awne","facebook","supervisord","nginx","mysql","amazonec2","amazonrds"]}]
```

发布到/contact 端点以向我发送电子邮件:

```
$ curl -X POST -F 'channel=email' -F 'message=test email message dude!!' -F "from=test@test.net" -F "token=public" http://35.189.117.234/contact/ 
{"success_code":202,"success_text":"Email successfully sent, thanks so much!"}
```

就是这样！(我觉得。)我们的 API 现在已经完全容器化了，耶！

一旦一切都在运行，并且您想要扩大或缩小某个特定的微服务，只需运行 kubectl scale 命令即可:

```
$ kubectl scale deployment contact-endpoint --replicas=5
```

这将创建运行/contact API 端点的 pod 的 5 个副本。缩小是一样的，只是用一个更小的数字替换 5(如果指定 replicas=0，将没有 pods 运行应用程序)。

您甚至可以根据 CPU 需求自动扩展部署:

```
$ kubectl autoscale deployment contact-endpoint --min=1 --max=5 
--cpu-percent=80
```

要更新服务，无需任何停机，只需对您的程序、内容和/或 Dockerfile 进行更改，然后完成构建 docker 容器的步骤，将其推送到 Google Cloud，并使用新容器应用 deployment.yaml(无需对服务做任何事情)。这将使用新版本替换部署中正在运行的 pod。

如果在更新部署时出错，可以快速回滚到给定部署的前一版本:

```
$ kubectl rollout undo deployment/nginx
```

当然，你可以用 Kubernetes 部署做更多的事情，我只是触及了表面——如果你想了解更多，请参见这里的文档:[https://Kubernetes . io/docs/concepts/workloads/controllers/deployment/](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)。

哎哟，这已经变成了一个有点像怪物的帖子——如果你还和我在一起，我希望它在给出 Docker 和 Kubernetes 中的一些概念的实际演练中证明是有用的。正如我之前说过的，我对这个很陌生，所以毫无疑问这里有一些错误——如果你发现了什么，请告诉我，我会更新帖子的！