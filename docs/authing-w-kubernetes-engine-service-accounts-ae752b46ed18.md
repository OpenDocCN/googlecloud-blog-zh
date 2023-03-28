# 授权 Kubernetes 引擎服务帐户

> 原文：<https://medium.com/google-cloud/authing-w-kubernetes-engine-service-accounts-ae752b46ed18?source=collection_archive---------0----------------------->

## 警告开发者

您可以使用 Kubernetes(！idspnonenote)来验证 Kubernetes 引擎资源。)服务帐户。我在 Kubernetes(引擎)文档中找不到这种方法的例子，所以请谨慎对待，如果您对这种方法有任何安全顾虑，**不要**使用它。

我们的目标是使用 Kubernetes 引擎服务帐户来认证和授权一些软件过程，例如 CI|CD 管道。

> **注意**当我最初研究这个问题时，我假设服务帐户需要是 Google 云平台服务帐户，而不是 Kubernetes 引擎服务帐户。

## 串

我假设您有一个(牺牲性的)集群(称为`cluster`)愿意用来测试这个集群和一个称为`root`的上下文。

```
kubectl config get-clusters
NAME
clusterkubectl config get-contexts
CURRENT   NAME   CLUSTER   AUTHINFO    NAMESPACE
          root   cluster   [[YOU]]
```

## 语境

为了确保我们明确使用哪个上下文，让我们取消默认上下文；这将要求我们为每个`kubectl`命令明确引用我们想要的上下文:

```
kubectl config unset current-contextkubectl get nodes
The connection to the server localhost:8080 was refused - did you specify the right host or port?kubectl get nodes --context=root
NAME                                        STATUS   ROLES    AGE   VERSION
gke-cluster-01-default-pool-b0fa792d-lt2v   Ready    <none>   80m
gke-cluster-01-default-pool-b2ff416e-vtbt   Ready    <none>   80m
gke-cluster-01-default-pool-b457df6d-6n2s   Ready    <none>   80m
```

## 命名空间

为了确定我们的工作范围，让我们为此创建并使用一个名称空间:

```
NAMESPACE=testkubectl create namespace ${NAMESPACE} --context=root
namespace/test created
```

## 服务帐户

让我们创建一个 Kubernetes 引擎服务帐户。为简单起见，我们将在`${NAMESPACE}`中创建服务帐户，但它不需要在也不限于这个名称空间中创建:

```
NAME=testecho "
apiVersion: v1
kind: ServiceAccount
metadata:
 name: ${NAME}
 namespace: ${NAMESPACE}
" | kubectl apply --filename=- --context=root
serviceaccount/${NAME} created
```

> 如果你不喜欢我重复使用`test`这个名字，我很抱歉。因为每个资源都是不同的，所以这不是问题。使用你喜欢的价值观。

## 证明

Kubernetes 引擎利用 Google[云平台] [OAuth2]认证。如果您检查您的 Kubernetes 配置文件，您会看到您的凭证是使用`gcloud config config-helper`获得的:

```
users:
- name: [[YOUR-GOOGLE-ID]]
  user:
    auth-provider:
      config:
        access-token: ya29.[[REDACTED]]
        cmd-args: **config config-helper** --format=json
        cmd-path: /usr/lib/google-cloud-sdk/bin/**gcloud**
        expiry: "2019-03-07T23:59:59Z"
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      name: gcp
```

如果您运行这个命令，您将会看到由`gcloud`命令获得的`access_token`，并由`kubectl`用来向集群验证您的身份。

这篇文章中概述的方法的一个副作用是，它不要求`gcloud`对软件重复验证集群可用。

我们将创建一个新用户(名为`${NAME}`)来代表服务帐户:

```
SECRET=$(kubectl get serviceaccount/${NAME} \
--namespace=${NAMESPACE} \
--context=root \
--output=jsonpath="{.secrets[0].name}")TOKEN=$(kubectl get secret/${SECRET} \
--namespace=${NAMESPACE} \
--context=root \
--output=jsonpath="{.data.token}" | base64 --decode)kubectl config set-credentials ${NAME} \
--token=${TOKEN}kubectl config set-context ${NAME} \
--user=${NAME} \
--cluster=cluster
```

> **NB**`set-credentials`命令正在向您的 Kubernetes 配置文件添加一个**载体**令牌。您必须控制对此令牌和文件的访问。如果你失去了其中任何一个的控制权，撤销秘密(！)使用`kubectl delete secret/${SECRET} --namespace=${NAMESPACE} --context=root`。
> 
> 在最后一个命令中，我们将创建一个以`${NAME}`命名的上下文。您可能更喜欢将上下文命名为与用户名不同的名称。
> 
> 如果你得到 base64 解码值。但是`kubectl get secret/${SECRET} ...`返回 base64 编码的值。我们想要解码后的值。

现在，如果你回顾~/。kube/config 文件中，您应该看到为`${NAME}`添加了一个`user`条目和一个`context`条目:

```
contexts:
**- context:
    cluster: cluster
    user: ${NAME}
  name: ${NAME}**
- context:
    cluster: cluster
    user: [[YOU]]
  name: root
current-context: ""
kind: Config
preferences: {}
users:
- name: [[YOU]]
  user:
    auth-provider:
      config:
        access-token: ya29.[[REDACTED]]
        cmd-args: config config-helper --format=json
        cmd-path: /usr/lib/google-cloud-sdk/bin/gcloud
        expiry: "2019-03-07T23:59:59Z"
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      name: gcp
**- name: ${NAME}
  user:
    token: ${TOKEN}**
```

但是:

```
kubectl get nodes --context=${NAME}
Error from server (Forbidden): nodes is forbidden: User "system:serviceaccount:**${NAMESPACE}:${NAME}**" cannot list resource "nodes" in API group "" at the cluster scope
```

已验证但未授权。

## 批准

此帐户在群集上没有权限。

我们将授予`${NAME}`使用部署的能力。为此，我们需要创建一个角色(为了更具体，我使用了`Role`而不是`ClusterRole`，然后在`${NAME}`和`Role`之间创建一个`RoleBinding`。

```
echo "
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ${NAME}
  namespace: ${NAMESPACE}
rules:
- apiGroups:
  - apps
  - extensions
  resources:
  - deployments
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
...
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ${NAME}
  namespace: ${NAMESPACE}
subjects:
- kind: ServiceAccount
  name: ${NAME}
  namespace: ${NAMESPACE}
roleRef:
  kind: Role
  name: ${NAME}
  apiGroup: rbac.authorization.k8s.io
---
" | kubectl apply --filename=- --context=root
role.rbac.authorization.k8s.io/${NAME} created
rolebinding.rbac.authorization.k8s.io/${NAME} created
```

## 试验

让我们在名称空间`default`和`${NAMESPACE}`中创建测试部署:

```
kubectl run nginx \
--image=nginx \
--replicas=1 \
--namespace=default \
--context=rootkubectl run nginx \
--image=nginx \
--replicas=1 \
--namespace=${NAMESPACE} \
--context=root
```

我们的`root`上下文拥有完全授权，但是我们的`${NAME}`上下文(代表我们的`${NAME}`用户)应该只能枚举`${NAMESPACE}`中的部署:

```
kubectl get deployments \
--context=${NAME} \
--namespace=**default**
Error from server (Forbidden): deployments.extensions is forbidden: User "system:serviceaccount:${NAMESPACE}:${NAME}" cannot list resource "deployments" in API group "extensions" in the namespace "default"kubectl get deployments \
--context=${NAME} \
--namespace=**${NAMESPACE}**
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx   1         1         1            1           25s
```

> **NB** `kubectl run`使用`extensions`而不是新的`apps` API 组创建部署。

## 结论

由 Google OAuth 支持的 Kubernetes Engine auth 是非常好的，也是一个不错的选择。

正如这里所展示的，如果您有一个集群外部的软件(而不是人)并且需要与集群交互，那么您可以使用 Kubernetes 服务帐户来验证和授予帐户精确的权限(授权)。

直接将`gcloud`包含在外部软件的配置中就省去了。尽管——请记住——添加到`~/.kube/config`文件中的令牌是一个**不记名**令牌。拥有令牌的任何人都可以使用服务帐户的权限访问群集。如果您失去了对令牌或引用它的 Kubernetes 配置文件的控制，**删除秘密**:

```
kubectl delete secret/${SECRET} \
--namespace=${NAMESPACE} \
--context=root
```

这将禁止使用令牌:

```
kubectl delete secret/${SECRET} \
--namespace=${NAMESPACE} \
--context=**root**
secret "fred-token-nbqw9" deletedkubectl get deployment/nginx \
--namespace=${NAMESPACE} \
--context=${NAME}
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx   1         1         1            1           59mkubectl get deployment/nginx \
--namespace=${NAMESPACE} \
--context=${NAME}
error: You must be logged in to the server (Unauthorized)
```

结果不是即时的，但应该是迅速的。

## 整理一下！

大锤式的整理方法是:

```
kubectl delete namespace/${NAMESPACE} --context=root
```

因为`Service Account`、`Role`、`RoleBinding`和一个`Deployment`是在这个名称空间中创建的，所以它们将被一起删除。

要删除其他部署:

```
kubectl delete deployment/nginx --namespace=**default** --context=root
```

要删除配置文件更改，请执行以下操作:

```
kubectl config unset users.${NAME}
kubectl config unset contexts.${NAME}
```

仅此而已！