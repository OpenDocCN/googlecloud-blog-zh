# 谷歌云平台上的基础设施代码:存储桶

> 原文：<https://medium.com/google-cloud/infrastructure-as-code-on-google-cloud-platform-storage-buckets-40f09bfa4328?source=collection_archive---------5----------------------->

关于我使用 Google 的部署管理器所学到的东西的系列文章的第 2 部分

在我的上一篇文章中，我介绍了 Google 的部署管理器，并演示了如何设置基本的发布/订阅主题和订阅。现在来看看更复杂的东西:存储桶。在我体验了 pub/sub 之后，我并不认为它们会有多难，在这种情况下，我是对的。

到目前为止，我的方法是在谷歌云控制台上手动设置我想要的东西，确保它能工作，然后尝试用 GDM (gcloud deployment-manager)重新创建它。为了确定我的配置文件中需要什么设置，我使用了 Google API 文档和 *gcloud * describe* 命令的组合。

存储桶配置可能比发布/订阅稍微复杂一些。您至少需要以下 YAML 节点:

*   `name`:在这种情况下，资源块的名称也将是资源的名称。即`name: my-storage-bucket`将创建一个名为 my-storage-bucket 的存储桶
*   `type`:资源类型，本例中为`storage.v1.bucket`
*   `storageClass`:谷歌最近更新了他们的存储类。截至目前，您可以指定`MULTI-REGIONAL`、`REGIONAL`、`NEARLINE`或`COLDLINE`。自 2016 年 11 月 1 日起，`STANDARD`的默认值根据位置设置自动转换为`MULTI-REGIONAL`或`REGIONAL`。

除此之外，您还可以指定`location`、`versioning`、`lifecycle`、`acl`等设置。请记住，有些设置是不兼容的。例如，您可能没有指定`location`和`storageClass: MULTI-REGIONAL`。关于可用选项的完整列表，可以在[这里](https://cloud.google.com/storage/docs/json_api/v1/buckets)找到 API 文档。

作为一个小小的题外话，我们来谈谈创建服务帐户。这只是一点小小的偏离。真的。服务帐户需要很少的配置:

```
- name: dav-test-serviceaccount
  type: iam.v1.serviceAccount
  properties:
    accountId: dav-test-svc
    displayName: dav-test-svc
```

没有密钥，服务帐户就没什么用了，所以就有了这个:

```
- name: dav-test-svc-key
  type: iam.v1.serviceAccounts.key
  properties:
    parent: projects/my-project/serviceAccounts/dav-test-svc@my-project.iam.gserviceaccount.com
```

你可能会对自己说，“自我，知道`parent:`价值似乎是一种非常糟糕的做事方式。”你是对的。您可以通过引用为部署文件中包含的任何资源指定值。所以把它们放在一起:

```
- name: dav-test-serviceaccount
  type: iam.v1.serviceAccount
  properties:
    accountId: dav-test-svc
    displayName: dav-test-svc
- name: dav-test-svc-key
  type: iam.v1.serviceAccounts.key
  properties:
    parent: $(ref.dav-test-serviceaccount.name)
```

为了与存储桶主题联系起来，我们现在可以将存储桶的权限授予新的服务帐户。在存储桶上设置 ACL 时要记住一个问题—当您在 deployment-manager 中设置 ACL 时，它会替换默认或现有的 ACL，因此请指定您想要设置的所有权限。

最终的配置文件将所有这些放在一起:

```
resources:
- name: dav-test-serviceaccount
  type: iam.v1.serviceAccount
  properties:
    accountId: dav-test-svc
    displayName: dav-test-svc
- name: dav-test-svc-key
  type: iam.v1.serviceAccounts.key
  properties:
    parent: $(ref.dav-test-serviceaccount.name)- name: dav-storagebucket-test-01
  type: storage.v1.bucket
  properties:
    storageClass: MULTI_REGIONAL
    versioning:
      enabled: false
    lifecycle:
      rule:
      - action:
          type: Delete
        condition:
          age: 365
    acl:
    - role: OWNER
      entity: "project-owners-123456789012"
    - role: OWNER
      entity: "project-editors-123456789012"
    - role: READER
      entity: "project-viewers-123456789012"
    - role: WRITER
      entity: "user-$(ref.dav-test-serviceaccount.email)"
# TODO: figure out how to get project-owners et.al. by reference
outputs:
- name: bucket
  value: $(ref.dav-storagebucket-test-01.selfLink)
- name: svcacct
  value: $(ref.dav-test-serviceaccount.email)
- name: svcacct-key
  value: $(ref.dav-test-svc-key.privateKeyData) 
         # this will be base64 encoded
```

我之前没有提到的一件事是`outputs`块。这允许您在运行结束时检索有关部署的信息:

```
[dbayer@LVML201033: ~/tmp]$ gcloud deployment-manager deployments create --config=buckets.yaml dav-test-bucket
Waiting for create operation-1481485797164-543674aae03e1-60b947ad-bcac11a1...done.
Create operation operation-1481485797164-543674aae03e1-60b947ad-bcac11a1 completed successfully.
NAME                       TYPE                        STATE      ERRORS  INTENT
dav-storagebucket-test-01  storage.v1.bucket           COMPLETED  []
dav-test-serviceaccount    iam.v1.serviceAccount       COMPLETED  []
dav-test-svc-key           iam.v1.serviceAccounts.key  COMPLETED  []
OUTPUTS  VALUE
bucket       https://www.googleapis.com/storage/v1/b/dav-storagebucket-test-01
svcacct      dav-test-svc@my-project.iam.gserviceaccount.com
svcacct-key ewogICJ0eXBlIjogInNlcnZpY2VfYWNjb3VudCIsC...
            # svcacct-key will be a long base64 encoded string[dbayer@LVML201033: ~/tmp]$
```

不幸的是，我还没有想出如何检索服务帐户的私钥，这限制了创建密钥的有用性。我还想通过引用检索 ACL 的项目编号和/或实体。

**更新:Marc Vieira-Cardinal 在评论中展示了如何检索私钥。我已经在上面的代码块中加入了他的信息。我还开发了一种在使用模板时检索项目编号的方法，我在这里详细介绍了**[](/google-cloud/infrastructure-as-code-on-google-cloud-platform-beginning-templates-68882e68d666)****。****

**对于存储桶来说差不多就是这样。接下来(假设我没有被闪亮的东西分散注意力):负载平衡器。**