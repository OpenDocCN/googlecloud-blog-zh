# Kubernetes 引擎:kubectl 配置

> 原文：<https://medium.com/google-cloud/kubernetes-engine-kubectl-config-b6270d2b656c?source=collection_archive---------0----------------------->

这已经在其他地方[记录过了](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)但是我认为有必要总结一下`kubectl`和`gcloud`配置的使用，它们的交互，以及当你有多个 Kubernetes 集群时如何在 Kubernetes 配置之间切换。

我正在使用 Kubernetes 引擎，它使配置管理变得更加容易，但是——如果你和我一样——这可能会导致自满，让工具完成它们的工作，而不花时间去研究发生了什么。

## 设置

让我们假设您已经创建了 2 个 Kubernetes 引擎集群，并获得了它们的凭证。您将被配置为使用最近经过身份验证的群集:

```
gcloud container clusters create black ...
gcloud container clusters create white ...gcloud container clusters get-credentials black ...
gcloud container clusters get-credentials **white** ...kubectl get nodesNAME                  STATUS    ROLES     AGE       VERSION
**white**-2378271e-qpd6   Ready     <none>    1h        v1.10.2-gke.3
**white**-5d71e034-njph   Ready     <none>    1h        v1.10.2-gke.3
**white**-f4567e1d-4rwd   Ready     <none>    1h        v1.10.2-gke.3
```

> 为了说明这一点，我在上面重新命名了`NAME`。

我们已经通过了`white`集群的认证。

## kubectl 配置

让我们来看看 Kubernetes(发动机)的配置:

```
kubectl config view
```

底层配置文件很可能是(！)位于:

```
~/.kube/config
```

您的配置将类似于这个文件。为了清楚起见，我修改并重新命名了一些元素。

这里有两个集群、两个上下文和两个用户。还有一个属性`current-context`指向当前上下文。两个名字之一。在这种情况下`white`。

有一点可能不太明显，因为我创建了两个集群(`white`和`black`)并且对它们进行了身份验证，所以我的配置中不需要有两个不同但相同的用户。它们都反映了同一个用户。我建议你**不要**这个**而是**为了清楚起见，这个配置与上面的相同:

> **注意**我已经用我的用户`name`重命名了 2 个`users`中的一个，删除了重复的，并用我的用户`name`重命名了`contexts`中的引用。

让我们深入研究一下这个`auth-provider`。

## 访问令牌

从您的`kubectl config view`中获取访问令牌。它的形式应该是`ya29...`。姑且称之为`${ACCESS_TOKEN}`。

> **NB** 在结果中过期。如果您在该时间后尝试此命令，您的令牌将会过期。如果有，进行任意调用，例如`kubectl get nodes`来强制刷新令牌，重新运行`kubectl config view`并重试。

然后，您可以浏览或卷曲:

```
curl [https://www.googleapis.com/oauth2/v2/tokeninfo?access_token=**$**](https://www.googleapis.com/oauth2/v2/tokeninfo?access_token=ya29.Gl3VBZukOogc8_OjmqUwzttWgILzgnDzDpBadqfr9ggNaz_IpQNxUUDF1fcN7IGnENr29677JS9OnWKJx8YuqICryjdecgtMbJex-W6RISkDtacDLgpwzLuyBS1JvH4)**{ACCESS_TOKEN}**{
 "issued_to": "12345678901.apps.googleusercontent.com",
 "audience": "12345678901.apps.googleusercontent.com",
 "user_id": "123456789012345678901",
 "scope": "[https://www.googleapis.com/auth/userinfo.email](https://www.googleapis.com/auth/userinfo.email) [https://www.googleapis.com/auth/cloud-platform](https://www.googleapis.com/auth/cloud-platform) [https://www.googleapis.com/auth/appengine.admin](https://www.googleapis.com/auth/appengine.admin) [https://www.googleapis.com/auth/compute](https://www.googleapis.com/auth/compute) [https://www.googleapis.com/auth/plus.me](https://www.googleapis.com/auth/plus.me)",
 "expires_in": 1199,
 "email": "[[](mailto:daz.wilkin@gmail.com)[YOUR-EMAIL]]",
 "verified_email": true,
 "access_type": "offline"
}
```

> **注意**邮件反映了你在`gcloud`使用的账户。如果你查看你的`gcloud`配置(或者`gcloud auth list`或者`gcloud config list`，或者更准确地说`gcloud config get-value account`，这应该是你的`ACTIVE` `ACCOUNT`名字。

## g 云配置

`auth-provider`引用了`gcloud`。具体为`cmd-path`、`cmd-args`和`expiry-key`、`token-key`两个属性。`cmd-path`应匹配:

```
which gcloud/.../google-cloud-sdk/bin/gcloud
```

`cmd-args`没有被谷歌很好地记录(提交了一个 bug :-)，但谷歌的一位工程师解释说([链接](https://github.com/GoogleCloudPlatform/google-auth-library-python/issues/146)):

```
gcloud config config-helper
configuration:
  active_configuration: default
  properties:
    core:
      account: [[](mailto:daz.wilkin@gmail.com)[YOUR-EMAIL]]
      disable_usage_reporting: 'False'
credential:
  access_token: [[ACCESS_TOKEN]]
  token_expiry: '2018-06-09T00:00:00Z'
sentinels:
  config_sentinel: /.../.config/gcloud/config_sentinel
```

或者你可以加上`--format=json`和:

```
{
  "configuration": {
    "active_configuration": "default",
    "properties": {
      "core": {
        "account": "[[YOUR-EMAIL]]",
        "disable_usage_reporting": "False"
      }
    }
  },
  "credential": {
    "access_token": "[[ACCESS_TOKEN]]",
    "token_expiry": "2018-06-09T00:00:00Z"
  },
  "sentinels": {
    "config_sentinel": "/.../.config/gcloud/config_sentinel"
  }
}
```

当您使用`kubectl`时，它使用`auth-provider`配置来使用`gcloud config config-helper`为您的帐户获取一个`access_token`，以便它可以访问集群的资源。

我们先简单绕道通过`gcloud` `configurations`来说明这一点。

## gcloud config 配置

你会注意到上面的`gcloud`也有`configurations`。我很少使用这个工具，但是为了完整起见…

```
gcloud config list
[core]
account = [[](mailto:daz.wilkin@gmail.com)[YOUR-EMAIL]]
disable_usage_reporting = FalseYour active configuration is: [**default**]
```

> **NB**

`gcloud`的配置存放在哪里？大概是:

```
ls -la ~/.config/gcloud

active_config
application_default_credentials.json
cache
config_sentinel
configurations
```

> **NB** 编辑以论证观点

如果您:

```
less ~/.config/gcloud/active_config**default**
```

哪一个是配置前缀`config_`中条目的关键字:

```
less ~/.config/gcloud/config_**default**[core]
account = [[](mailto:daz.wilkin@gmail.com)[YOUR-EMAIL]][container][compute]
```

> **NB** 你的配置可能有更多属性。

这应该反映了什么结果`gcloud config list`。

您可以(！)创建多个配置，以反映您希望使用的不同 GCP 配置。例如:

```
gcloud config configurations list
NAME     IS_ACTIVE  ACCOUNT
default  True       [[](mailto:daz.wilkin@gmail.com)[YOUR-EMAIL]]gcloud config configurations create **henry**
Created [**henry**].
Activated [**henry**].gcloud config configurations list
NAME     IS_ACTIVE  ACCOUNT
default  True       [[](mailto:daz.wilkin@gmail.com)[YOUR-EMAIL]]
**henry**    True
```

然后，您可以验证:

```
cat ~/.config/gcloud/active_config
**henry**ls -l ~/.config/gcloud/configurations
config_default
config_**henry**
```

还有，收拾一下:

```
gcloud config configurations activate **default**gcloud config configurations delete henry
The following configurations will be deleted:
 - henry
Do you want to continue (Y/n)?  YDeleted [henry].
```

我发现很少使用 gcloud 配置，而是倾向于针对`default`配置进行多次登录:

```
gcloud auth list
       Credentialed Accounts
ACTIVE  ACCOUNT
        [[](mailto:daz.wilkin@brabantcourt.com)[EMAIL-#1]]
*       [[](mailto:daz.wilkin@gmail.com)[EMAIL-#2]]
        [[](mailto:dazwilkin@cloud-sce.com)[EMAIL-#3]]To set the active account, run:
    $ gcloud config set account `ACCOUNT`
```

如上所示，使用例如`gcloud config set account [[EMAIL-#3]]`在这些选项之间切换。

## kubectl 配置使用上下文

好了，我们现在明白了`kubectl`如何使用`gcloud`来对抗 GCP。但是，我们还没有解决如何管理多个`kubectl`配置的问题。

这是有据可查的([链接](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/))，但这里是我对它的看法。让我们回到我们的`white`和`black`集群。正如我提到的，Kubernetes 引擎将为您的`clusters`、`contexts`和`users`创建唯一的名称。我已经解释了你可以如何(你不需要这样做，而且要小心)重命名|合并你的`users`。你也可以对`clusters`和`contexts`做同样的事情。只是要小心，在你的配置文件中，你要正确地将所有的引用重命名为**。**

如果您破坏了某些东西，您可以重新运行`gcloud container clusters get-credentials`来针对您的集群重新进行身份验证。

那么，我们如何在集群之间切换呢？其实很简单。假设:

```
kubectl config get-contexts
CURRENT   NAME          CLUSTER AUTHINFO           NAMESPACE
*****         **white**         white   dazwilkin
          black         black   dazwilkin
```

你需要使用在你的配置中定义的任何值。

您可以:

```
kubectl config use-context **black**
Switched to context "black"kubectl config use-context white
Switched to context "white"
```

如果你想让自己的生活更简单:

```
alias black="kubectl config use-context black"
black
Switch to context "black"
```

## 结论

【Kubernetes 引擎利用 Google 云平台的`gcloud` CLI 来促进针对集群的身份验证。这篇文章总结了如何在授权集群之间切换，而不需要重复使用`gcloud`来获取凭证。

一路上，这篇文章提供了一些简化`kubectl`配置的**小心操作**说明，以及`kubectl`和`gcloud`如何管理配置和数据存储位置的总结。

随时欢迎反馈！

仅此而已。