# 无需 gcloud 即可向 GKE 认证

> 原文：<https://medium.com/google-cloud/authenticating-to-gke-without-gcloud-aefd23a840a9?source=collection_archive---------0----------------------->

如果您正在使用 [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine) 并从 CI/CD 之类的无头环境中部署到它，那么您可能正在安装`gcloud`命令行工具(可能每次运行构建时)。有一种方法可以在不使用 `**gcloud**`命令行工具的情况下认证到 GKE 集群**！**

解决方案是使用我们提前制作的静态`[kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)`文件。为此，您仍然需要:

1.  CLI(仅在开发机器上)
2.  用于验证您身份的 Google 凭据(又称服务帐户密钥)。

# 创建静态 kubeconfig 文件

在 bash 终端的变量中设置集群名称和区域:

```
GET_CMD="gcloud container clusters describe [CLUSTER] --zone=[ZONE]"
```

在 bash 中运行以下命令块将通过检索创建一个`kubeconfig.yaml`文件:

```
cat > kubeconfig.yaml <<EOF
apiVersion: v1
kind: Config
current-context: my-cluster
contexts: [{name: my-cluster, context: {cluster: cluster-1, user: user-1}}]
users: [{name: user-1, user: {auth-provider: {name: gcp}}}]
clusters:
- name: cluster-1
  cluster:
    server: "https://$(eval "$GET_CMD --format='value(endpoint)'")"
    certificate-authority-data: "$(eval "$GET_CMD --format='value(masterAuth.clusterCaCertificate)'")"
EOF
```

这个`kubeconfig.yaml`文件不包含诸如您的凭证之类的秘密。它只将 kubectl 指向您的集群。实际上，您可以放心地选择将该文件存储在 git 存储库中。

请注意，您实际上可以通过[触发手动轮换](https://cloud.google.com/kubernetes-engine/docs/how-to/credential-rotation)来轮换这个主 IP 地址*和* CA 证书。如果这样做，您需要重新生成该文件。(这是这种方法的唯一缺点。)

# 为无头身份验证创建服务帐户

1.  您将需要[创建一个服务帐户](https://cloud.google.com/iam/docs/creating-managing-service-accounts)，以便从 headless 环境向 GKE 进行身份验证。
2.  为该服务帐户提供您需要的。(例如，“Kubernetes 引擎开发人员”角色将允许您将工作负载部署到集群。)
3.  然后，创建一个[密钥文件](https://cloud.google.com/iam/docs/creating-managing-service-account-keys)(。json)(该文件是一个**机密**，不要将其签入到您的存储库中)。

# 使用 kubeconfig 文件

现在，您可以转到没有 `**gcloud**`的环境**，获取这个 kubeconfig 文件并将其与您的服务帐户密钥文件结合，通过设置以下环境变量从 headless 环境向您的 GKE 集群进行身份验证:**

```
export GOOGLE_APPLICATION_CREDENTIALS=service-account-key.json
export KUBECONFIG =kubeconfig.yamlkubectl get nodes # You are authenticated if this works!
```

将`GOOGLE_APPLICATION_CREDENTIALS`设置为 kubectl 可以很好地工作，因为 kubectl 中的`gcp` auth 插件使用标准的 Google Cloud Go 客户端库来识别这个环境变量。

希望这个小技巧可以加速您的构建环境，而不必维护安装和配置`gcloud` CLI 的步骤。

这不是在没有`gcloud`的情况下向 GKE 集群进行身份验证的唯一方式。您也可以使用 [Kubernetes 服务账户](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)进行认证，也许我们可以在另一篇文章中探讨这个问题。

*原载于 2019 年 7 月 9 日*[*https://Ahmet . im*](https://ahmet.im/blog/authenticating-to-gke-without-gcloud/)*。*