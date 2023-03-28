# 使用 Kubernetes、Nginx Ingress 和 Let's Encrypt 设置 Google Cloud(cert manager)

> 原文：<https://medium.com/google-cloud/setting-up-google-cloud-with-kubernetes-nginx-ingress-and-lets-encrypt-certmanager-bf134b7e406e?source=collection_archive---------0----------------------->

![](img/c31c9be4d76cbda4f84f7e9f0bd704fd.png)

在这篇文章中，我将尝试分享我在 kubernetes 中设置集群的步骤。请注意，我对 Google Cloud & Kubernetes 的了解仅限于一周的研究，所以我愿意接受改进它的建议。

这是针对那些刚刚开始探索谷歌云/ Kubernetes 的人，并试图巩固一周的(有时令人困惑的)研究

**假想知识**(一周内可习得！；))

*   kubernetes 工作原理的基本概念(pod、部署、服务)
*   谷歌云的基本概念

**最后会部署什么:**

*   单节点 kubernetes 集群，不支持扩展。
*   包含应用程序的 docker 图像(Nginx/Angular2)
*   包含 api 的 docker 映像(带有 mongodb 的连接细节)(NodeJS)
*   一个 docker 图像与 nginx 反向代理谷歌存储
*   Nginx 入口路由到 docker 后端
*   由加密验证的证书，由入口使用

**未包含:**

*   实际的 docker 图像，这个指南更多的是作为如何设置这个“架构”的指南。

## 设置 GCloud CLI(如果可用，请跳过)

有关如何设置 Google Cloud CLI，请参见附录。从这一点出发，假设您有一个工作的 gcloud，登录并指向您的项目/计算区域

# 创建集群

`gcloud container clusters create <cluster_name> --num-nodes=1`

这一步将为您创建一个具有一个节点(一台计算引擎机器)的集群，所有的 pods 服务都将在该集群上运行。

`gcloud container clusters get-credentials <cluster_name>`

这个命令配置 **kubectl** ，它是与 kubernetes 交互的 cli 工具，针对我们刚刚创建的集群工作

# 创建名称空间(可选)

出于我们的目的，名称空间旨在将环境相互分离。例如，我们将在同一个集群/节点上运行测试/QA 环境。这将使 pod 和服务相互隔离。

`kubectl create namespace <name>`

一个集群在刚创建时通常有两个名称空间，**默认**和 **kube-system** ，如果不在命令中指定名称空间，您的资源将会在**默认**名称空间中结束。 **kube-system** 是为 kubernetes 服务保留的。

# 设置舵，安装舵杆

Helm 可以被描述为 kubernetes 的一个包管理器，您将使用这个工具安装 nginx-ingress 和 certmanager。它作为客户机/服务器架构运行，helm 是您使用的 cli 工具，tiller 是服务器(我们稍后将在集群上安装)。

```
*kubectl* create serviceaccount --namespace kube-system tiller
*kubectl* create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller*helm* init --service-account tiller*kubectl* patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'*helm* init --service-account tiller --upgrade
```

这里我们不确定是否需要补丁。但是它将在 kube-system 名称空间中创建一个服务帐户，创建必要的 rolebindings，并用这个帐户初始化 helm。

有关这些步骤的更多详细信息，请参考[https://cloud . Google . com/solutions/continuous-integration-helm-concourse](https://cloud.google.com/solutions/continuous-integration-helm-concourse)。

# 部署应用 docker

在此步骤之前，我们已经在注册表中提供了图像。对于我们的应用程序来说，这是一个 nginx，它公开了端口 80 并服务于 angular 2 构建。

我们有一个在启动 nginx 之前运行的脚本，它替换了源代码中的 api 和静态主机变量

```
*echo* "Configuring environment variable"
*sed* -i -e 's#SET_API_HOST_VARIABLE#'"$API_HOST"'#g' /www/main.*.js
*sed* -i -e 's#SET_STATIC_HOST#'"$STATIC_HOST"'#g'  /www/main.*.js
```

这确保了我们有一个通用的 docker 映像构建，但是我们仍然可以在部署之后使用环境变量来调整主机。

这也是我们在 kubernetes 中做的第一步，为我们的应用程序创建一个**配置图**。

```
*kubectl* create configmap AppNameConfig --from-literal API_HOST=http://api-host --from-literal STATIC_HOST=$STATIC_HOST
```

然后在我们的应用程序 kubernetes 清单文件中使用这个 configmap，以指定要注入的环境变量，以及它需要从哪里获取值，在本例中是 map。

```
**apiVersion:** apps/v1beta2
**kind:** Deployment
**metadata:
  name:** AppName
**spec:
  replicas:** 1
  **selector:
    matchLabels:
      app:** AppName
  **strategy:
    rollingUpdate:
      maxSurge:** 1
      **maxUnavailable:** 1
    **type:** RollingUpdate
  **template:
    metadata:
      labels:
        app:** AppName
    **spec:
      containers:** - **env:** - **name:** API_HOST
          **valueFrom:
            configMapKeyRef:
              key:** API_HOST
              **name:** AppNameConfig
        - **name:** STATIC_HOST
          **valueFrom:
            configMapKeyRef:
              key:** STATIC_HOST
              **name:** AppNameConfig
        **name:** AppName
        **image:** AppDockerImageName
        **ports:** - **containerPort:** 80
```

此清单适用于:

```
*kubectl* apply -f AppManifest.yml
```

这将在 kubernetes 上创建部署，这将启动包含运行您的映像的容器的 pod。

此时，无法从外部访问该部署。为了让我们的负载平衡器工作，我们需要将其公开为一个集群 ip 服务。这只是确保这个部署有一个集群 ip。

```
*kubectl* expose deployment AppName
```

# 部署 API docker

这个的流程和 app docker 完全一样。除了我们需要一个 mongo 连接。这最好存储在 kubernetes secret 中，这样它们就不像放在 configmap 中那样可见。

创建一个秘密:

```
*kubectl* create secret generic ApiMongoSecret --from-literal MONGO_HOST="<host>" --from-literal MONGO_REPLICA_SET="<replica>" --from-literal MONGO_GET_VARIABLES="<variables>"
```

像以前一样创建配置图:

```
*kubectl* create configmap ApiConfigMap --from-literal API_HOST=http://apihost --from-literal STATIC_HOST=http://statichost --from-literal APP_HOST=http://apphost
```

manifest 文件略有不同，因为它现在也需要注入环境变量，但是用秘密的内容填充它们。(我只包含了每种方法的一个例子)。

```
**apiVersion:** apps/v1beta2
**kind:** Deployment
**metadata:
  name:** ApiName
**spec:
  replicas:** 1
  **selector:
    matchLabels:
      app:** ApiName
  **strategy:
    rollingUpdate:
      maxSurge:** 1
      **maxUnavailable:** 1
    **type:** RollingUpdate
  **template:
    metadata:
      labels:
        app:** ApiName
    **spec:
      containers:** - **env:** - **name:** API_HOST
          **valueFrom:
            configMapKeyRef:
              key:** API_HOST
              **name:** ApiConfigMap
        - **name:** MONGO_HOST
          **valueFrom:
            secretKeyRef:
              key:** MONGO_HOST
              **name:** ApiMongoSecret
        **name:** ApiName
        **image:** ApiDockerImageName
        **livenessProbe:
          failureThreshold:** 3
          **httpGet:
            path:** /customendpoint
            **port:** 6000
            **scheme:** HTTP
          **initialDelaySeconds:** 60
          **periodSeconds:** 60
          **successThreshold:** 1
          **timeoutSeconds:** 2
        **readinessProbe:
          failureThreshold:** 3
          **httpGet:
            path:** /customendpoint
            **port:** 6000
            **scheme:** HTTP
          **initialDelaySeconds:** 60
          **periodSeconds:** 60
          **successThreshold:** 1
          **timeoutSeconds:** 2
        **ports:** - **containerPort:** 6000
```

注意，这个清单还有一个 livenessProbe 和 readinessProbe 配置。默认情况下，ingress 将查询后端的“/”，并期望得到 200 的结果，以了解实例是否“正常”。因为在我们的例子中，我们没有任何来自/用于 api 的服务，我们在这里覆盖这个检查。

据我所知，这也将是部署将用来了解 pod 是否处于错误状态的端点，以便它可以停止和重新启动它们。

和以前一样，应用清单文件并将其公开为集群 ip 服务:

```
*kubectl* apply -f ApiManifest.yml
*kubectl* expose deployment ApiName
```

# 创建 Google 存储反向代理

同样，部署这个代理的步骤是相同的。这可能也可以配置一个入口，但是我们没有时间研究这个。我们的 docker 是一个 nginx，配置如下:

```
user  nginx;
worker_processes  2;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

load_module /usr/lib/nginx/modules/ngx_http_perl_module.so;

env GS_BUCKET;
env INDEX;

events {
    worker_connections  10240;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;

    keepalive_timeout  65;

    resolver                   8.8.8.8 valid=300s ipv6=off;
    resolver_timeout           10s;

    upstream gs {
        server                   storage.googleapis.com:443;
        keepalive                128;
    }

    perl_set $bucket_name 'sub { return $ENV{"GS_BUCKET"}; }';
    perl_set $index_name  'sub { return $ENV{"INDEX"} || "index.html"; }';

     gzip on;
      gzip_static on;
      gzip_disable "msie6";
      gzip_vary on;
      gzip_proxied any;
      gzip_comp_level 6;
      gzip_buffers 16 8k;
      gzip_http_version 1.0;
      gzip_min_length 256;
      gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/vnd.ms-fontobject application/x-font-ttf font/opentype image/svg+xml image/x-icon application/octet-stream;

    server_tokens off;

    server {
        if ( $request_method !~ "GET|HEAD" ) {
            return 405;
        }

        location ~ /(.*) {
            set $query $1;
            proxy_set_header    Host storage.googleapis.com;
            proxy_pass          https://gs/$bucket_name/$query;
            proxy_http_version  1.1;
            proxy_set_header    Connection "";

            proxy_intercept_errors on;
            proxy_hide_header       alt-svc;
            proxy_hide_header       X-GUploader-UploadID;
            proxy_hide_header       alternate-protocol;
            proxy_hide_header       x-goog-hash;
            proxy_hide_header       x-goog-generation;
            proxy_hide_header       x-goog-metageneration;
            proxy_hide_header       x-goog-stored-content-encoding;
            proxy_hide_header       x-goog-stored-content-length;
            proxy_hide_header       x-goog-storage-class;
            proxy_hide_header       x-xss-protection;
            proxy_hide_header       accept-ranges;
            proxy_hide_header       Set-Cookie;
            proxy_ignore_headers    Set-Cookie;
        }
    }
}
```

这里的环境变量需要指定我们将代理哪个桶。同样，这保持了图像的一般性。

我们创建配置图:

```
*kubectl* create configmap StorageProxyConfigMap --from-literal GS_BUCKET=<NameOfTheStorageBucket>
```

将启动部署的清单文件:

```
**apiVersion:** apps/v1beta2
**kind:** Deployment
**metadata:
  name:** StorageProxyName
**spec:
  replicas:** 1
  **selector:
    matchLabels:
      app:** StorageProxyName
  **strategy:
    rollingUpdate:
      maxSurge:** 1
      **maxUnavailable:** 1
    **type:** RollingUpdate
  **template:
    metadata:
      labels:
        app:** StorageProxyName
    **spec:
      containers:** - **env:** - **name:** GS_BUCKET
          **valueFrom:
            configMapKeyRef:
              key:** GS_BUCKET
              **name:** StorageProxyConfigMap
        **name:** StorageProxyName
        **image:** StorageProxyImageName
        **ports:** - **containerPort:** 80
```

告诉 kubernetes 该怎么做:

```
*kubectl* apply -f StorageProxyManifest.yml
*kubectl* expose deployment StorageProxyName
```

# 安装 Nginx 入口

此时，我们有三个映像在它们的容器/pod/部署中运行，它们都作为服务公开，但是它们还不能从外部访问。为此，我们将安装一个入口负载平衡器。

Ingress 也分两部分运行，通过安装它，您设置了“nginx-ingress-controller”部署，它运行实际的 nginx。

之后，您可以创建入口资源，这些资源由控制器获取并配置 nginx。

(默认情况下，gcloud 为此提供了自己的控制器，但我们选择了 nginx，因为我们可以更好地匹配现有的 nginx 配置。)

```
*helm* install --name nginx-ingress stable/nginx-ingress --set rbac.create=true
*kubectl* apply -f /tmp/manifests-generated/ingress-resource.yml
```

*(RBAC 选项用于基于角色的访问控制，我们没有进一步探讨)。*

这将创建两个控制器，nginx-ingress-controller 和 nginx-ingress-default-backend，所有不匹配的 URL 都将路由到这两个控制器。

为了配置入口，我们创建以下资源:

```
**apiVersion:** extensions/v1beta1
**kind:** Ingress
**metadata:
  annotations:
    ingress.kubernetes.io/ssl-redirect:** "true"
    **kubernetes.io/ingress.class:** nginx
    **nginx.ingress.kubernetes.io/force-ssl-redirect:** "true"
    **nginx.ingress.kubernetes.io/rewrite-target:** /
  **name:** IngressName
**spec:
  rules:** - **host:** sub.host.com
    **http:
      paths:** - **backend:
          serviceName:** AppServiceName
          **servicePort:** 80
        **path:** /
      - **backend:
          serviceName:** ApiServiceName
          **servicePort:** 6000
        **path:** /api/
      - **backend:
          serviceName:** StorageProxyServiceName
          **servicePort:** 80
        **path:** /static/
  **tls:** - **hosts:** - sub.host.com
    **secretName:** NameOfCertificateSecret
```

此配置将/定向到应用程序，将请求/api/定向到 api，将/static/定向到存储代理。重写目标注释将确保/api/ & /static/不被发送到它们的后端，因此对于调用/api/todos/list，api 后端将接收它作为/todos/list。

注意，我们在这里还通过指定从哪个秘密中检索证书细节来配置 SSL。这是我们在设置证书管理器时配置的。基本上，入口期望从这个 secretname 得到一个有效的证书，否则你将得到一个假的自签名证书。

另请注意，虽然该入口现在有一个外部 IP，但该 IP 不是静态的，为此，必须将其提升为静态 IP。这可以通过谷歌云控制台(VPC 网络->外部 IP ->变短暂为静态，或通过控制台:

```
IP_ADDRESS=**$***(kubectl describe service nginx-ingress-controller --namespace=$NAMESPACE | grep 'LoadBalancer Ingress' | rev | cut -d: -f1 | rev | xargs)* 
*gcloud* compute addresses create NameOfStaticIp --addresses $IP_ADDRESS --region europe-west1
```

此命令将查找入口负载平衡器的 IP 地址(注意，对我们有效，因为我们只有一个，里程可能会有所不同:-))

然后我们将其提升为静态 IP。

# 安装证书管理器和证书

到目前为止，我们有一个负载平衡器，它有一个外部 IP，将请求路由到不同的后端，此时，您可以将 DNS 指向您的 IP。

用 helm 安装证书管理器:

```
*helm* install stable/cert-manager
```

创建一个颁发者(提供证书的实例，在我们的例子中，让我们加密):

```
**apiVersion:** certmanager.k8s.io/v1alpha1
**kind:** Issuer
**metadata:
  name:** NameForIssuer
**spec:
  acme:** *# The ACME server URL* **server:** https://acme-v02.api.letsencrypt.org/directory
    *# Email address used for ACME registration* **email:** "yourmail@domain.com"
    *# Name of a secret used to store the ACME account private key* **privateKeySecretRef:
      name:** IssuerPrivateKeyName
    *# Enable the HTTP-01 challenge provider* **http01:** {}
```

创建证书:

```
**apiVersion:** certmanager.k8s.io/v1alpha1
**kind:** Certificate
**metadata:
  name:** CertificateName
**spec:
  secretName:** NameOfCertificateSecret
  **commonName:** sub.host.com
  **dnsNames:** - sub.host.com
  **issuerRef:
    name:** NameForIssuer
    **kind:** Issuer
  **acme:
    config:** - **http01:
        ingressClass:** nginx
      **domains:** - sub.host.com
```

最后，应用它们:

```
*kubectl* apply -f issuer.yml
*kubectl* apply -f certificate.yml
```

一旦应用了这些资源，cert-manager 就会发现证书需要验证。它将创建一个新的入口来托管 ACME challenge，它还将启动一个执行加密请求的 pod。

一旦验证完成，它将在 secret 中存储详细信息(我们之前也在入口中使用了它)。

验证可能会有一些延迟，但通常情况下，几分钟后，您应该能够访问您的主机，并看到它是安全的！

**注意:**发现一些关于设置证书管理器的混乱信息。我认为可以通过两种方式来设置。如上所述，您可以手动创建颁发者/证书，并让 cert-manager 完成它的工作。

另一个是用注释配置入口，这样 cert-manager 就可以获取注释。在我们的配置中，这不需要特定的注释。

# 附录

## 设置 GCloud CLI

这里有两个选项，谷歌云外壳，或者从你最喜欢的本地终端。或者，您可以用 docker 包装 gcloud 命令:

```
**FROM** google**/**cloud-sdk:latest
**RUN** curl **-**o get_helm.sh https:**//**raw.githubusercontent.com**/**kubernetes**/**helm**/**master**/**scripts**/**get
**RUN** chmod **+**x get_helm.sh
**RUN** .**/**get_helm.sh
```

Helm 可以被描述为 kubernetes 的一个包管理器，您将使用这个工具安装 nginx-ingress 和 certmanager。它作为客户机/服务器架构运行，helm 是您使用的 cli 工具，tiller 是服务器(我们稍后将在集群上安装)。

如果从这个 docker 创建一个容器，并运行该容器，那么登录会话应该是持久的。在这种情况下，卷是在主机/docker 之间共享的脚本文件夹，因此我们可以添加文件/脚本以在云上运行。

```
*docker* create -v **$***(pwd)*/scripts:/scripts -w="/scripts" \
 --name=<container-name> -it gcloud-platform:latest /bin/bash*docker* start -ia <container-name>
```

按照您喜欢的风格安装 gcloud 后，您可以使用以下方式登录:

`gcloud auth login --brief`

它应该向您显示一个令牌的链接，您必须在终端中复制该令牌。登录后，使用
`gcloud config set project <PROJECT_ID>`设置您的项目 ID 和计算区域

`gcloud config set compute/zone <ZONE>`