# Google 云平台上的基础设施代码:初始模板

> 原文：<https://medium.com/google-cloud/infrastructure-as-code-on-google-cloud-platform-beginning-templates-68882e68d666?source=collection_archive---------0----------------------->

这是我使用谷歌部署管理器(GDM)所学到的知识的系列文章的第 4 部分

**更新 2017.09.01** :上周收到通知，下面提到的 bug 已经修复。我还没有机会测试它。

**更新 2017.04.04** :无法通过引用将服务账号权限分配给发布/订阅主题/订阅(见本文最后一个代码块)已经被 Google 承认为 bug。目前还没有修复的预计时间:[https://issuetracker.google.com/issues/35903722](https://issuetracker.google.com/issues/35903722)

我上次提到我正在开始使用模板。我使用模板的目的是允许开发人员提供一个简化的配置，概述他们需要的基础设施，以便他们可以根据需要创建和修改它。如果你读了我的上一篇文章，你会发现开发人员要学习创建一个部署配置所需的所有东西需要做多少工作——我是说真的，开发人员有编程之类的事情要做，你知道吗？如果我能提供一个简化的配置，并在一个模板中处理所有潜在的复杂性，哇哦！

现在我还没有自大到认为自己足够聪明，可以用负载平衡器开始我的模板之旅。我从更简单的东西开始——我是服务客户。到目前为止，我已经将所有的配置文件存储在一个 config 文件夹中。为了便于组织，我决定在子目录中创建模板，并在其中创建助手:

```
[dbayer: ~/tmp]$ tree -d config
config
└── templates
    └── helpers
```

让我们从服务帐户的现有配置开始:

```
resources:
- name: dav-svcaccount
  type: iam.v1.serviceAccount
  properties:
    accountId: dav-svcaccount
    displayName: dav-svcaccountoutputs:
- name: serviceAccountEmail
  value: $(ref.dav-svcaccount.email)
```

这很简单，但肯定有一些重复可以消除。模板可以用 Python 或者 Jinja 编写。因为它更像原始配置中的 YAML(也更像我顺便熟悉的 ERB 模板)，我选择了 Jinja。目前有 4 个可用的环境变量:project(GCP 项目名称)、deployment(部署名称)、name(资源名称)和 type(资源类型)。这是我想到的:

```
resources:
- name: {{ env['name'] }}-serviceaccount
  type: iam.v1.serviceAccount
  properties:
    accountId: {{ env['name'] }}
    displayName: {{ env['name'] }}outputs:
- name: email
  value: $(ref.{{ env['name'] }}-serviceaccount.email)
```

记住要在模板中输出电子邮件地址——这就是它稍后显示在您的配置文件中的方式。现在，我可以重写我的配置以使用该模板—您必须导入该模板以使其可用:

```
imports:
- path: templates/svcaccount.jinjaresources:
- name: dav-svcaccount
  type: templates/svcaccount.jinjaoutputs:
- name: serviceAccountEmail
  value: $(ref.dav-svcaccount.email)
```

嘿——现在我的开发人员只需要指定模板和服务帐户名！(最终，我的目标是简化流程并将其添加到 Jenkins 基础设施中，但这是以后的事了。)

现在让我们转到稍微难一点的话题——发布/订阅主题和订阅。让我们从基本配置开始:

```
resources:
- name: dav-pubsub-topic
  type: pubsub.v1.topic
  properties:
    topic: dav-pubsub-topic
  accessControl:
    gcpIamPolicy:
      bindings:
      - role: roles/pubsub.publisher
        members:
        - "serviceAccount:dav-svcaccount@myproject.iam.gserviceaccount.com"
      - role: roles/pubsub.viewer
        members:
        - "serviceAccount:dav-svcaccount@myproject.iam.gserviceaccount.com"
- name: dav-pubsub-subscription
  type: pubsub.v1.subscription
  properties:
    subscription: dav-pubsub-subscription
    topic: $(ref.dav-pubsub-topic.name)
    ackDeadlineSeconds: 600
  accessControl:
    gcpIamPolicy:
      bindings:
      - role: roles/pubsub.subscriber
        members:
        - "serviceAccount:dav-svcaccount@myproject.iam.gserviceaccount.com"
      - role: roles/pubsub.viewer
        members:
        - "serviceAccount:dav-svcaccount@myproject.iam.gserviceaccount.com"
```

同样，这里有很多重复，所以它是一个很好的模板候选。以下是 Jinja 中的模板:

```
resources:
- name: {{ env['name'] }}-topic
  type: pubsub.v1.topic
  properties:
    topic: {{ env['name'] }}
  accessControl:
    gcpIamPolicy:
      bindings:
      - role: roles/pubsub.publisher
        members:
        - "serviceAccount:{{ properties['serviceAccountEmail'] }}"
      - role: roles/pubsub.viewer
        members:
        - "serviceAccount:{{ properties['serviceAccountEmail'] }}"
- name: {{ env['name'] }}-subscription
  type: pubsub.v1.subscription
  properties:
    subscription: {{ env['name'] }}-sub
    topic: $(ref.{{ env['name'] }}-topic.name)
    ackDeadlineSeconds: {{ properties['ackDeadlineSeconds'] }}
  accessControl:
    gcpIamPolicy:
      bindings:
      - role: roles/pubsub.subscriber
        members:
        - "serviceAccount:{{ properties['serviceAccountEmail'] }}"
      - role: roles/pubsub.viewer
        members:
        - "serviceAccount:{{ properties['serviceAccountEmail'] }}"
```

这允许以下更简单的配置:

```
imports:
- path: templates/pubsub.jinjaresources:
- name: dav-pubsub
  type: templates/vst-pubsub.jinja
  properties:
    serviceAccountEmail: "dav-svcaccount@myproject.iam.gserviceaccount.com"
    ackDeadlineSeconds: 600
```

现在让我们对存储桶做同样的尝试。采用以下配置:

```
resources:
- name: dav-bucket
  type: storage.v1.bucket
  properties:
    storageClass: MULTI_REGIONAL
    acl:
    - role: OWNER
      entity: "project-owners-1234567890"
    - role: READER
      entity: "project-viewers-1234567890"
    - role: OWNER
      entity: "project-editors-1234567890"
    - role: OWNER
      entity: "user-dav-svcaccount@myproject.iam.gserviceaccount.com"
```

这一次，模板看起来有点不同——我添加了一些逻辑，以允许多个用户添加到所有者、作者和读者角色，以及一些逻辑，以便在配置中不需要这些:

```
resources:
- name: {{ env['name'] }}
  type: storage.v1.bucket
  properties:
    storageClass: {{ properties['storageClass'] }}
    acl:
    - role: OWNER
      entity: "project-owners-1234567890"
    - role: READER
      entity: "project-viewers-1234567890"
    - role: OWNER
      entity: "project-editors-1234567890"
{% if properties['owners'] %}
{% for user in properties['owners'] %}
    - role: OWNER
      entity: "user-{{ user }}"
{% endfor %}
{% endif %}
{% if properties['editors'] %}
{% for user in properties['editors'] %}
    - role: WRITER
      entity: "user-{{ user }}"
{% endfor %}
{% endif %}
{% if properties['readers'] %}
{% for user in properties['readers'] %}
    - role: READER
      entity: "user-{{ user }}"
{% endfor %}
{% endif %}
```

这进而导致以下配置:

```
imports:
- path: templates/bucket.jinjaresources:
- name: dav-bucket
  type: templates/svcaccount.jinja
  properties:
    storageClass: MULTI_REGIONAL
    owners:
    - dav-svcaccount1@myproject.iam.gserviceaccount.com
```

很遗憾，此模板有一个问题。它工作得很好，但是需要将项目编号硬编码到模板中。您必须为您处理的每个 GCP 项目创建单独的模板副本。

在开始这次旅程之前，我对金贾一无所知，为此我做了大量的搜索。您应该记得，从这篇文章的开始，Jinja 模板可以使用项目名称，但不能使用项目编号。

进入模板助手模块。这不是一个非常好的解决方案，但是比维护模板的多个副本要好得多。如果您还记得，前面我说过模板可以是 Jinja 或 Python 格式。通过切换到 Python，我可以在我的模板中包含助手脚本。对于这种情况，我用字典编写了一个简单的函数，其中包含我使用的项目名称和编号。由于项目名称可用于我的模板，所以我可以将名称传递给助手脚本并取回编号。

```
def GetProjectNumber(projectId):
    project = {'myproject': '1234567890',
            'myproject2': '0987654321',
            'myproject3': '6789012345'
            }
    return project[projectId]
```

现在我可以简单地在这个地方维护项目列表，或者有一天可能写一个脚本来更新它。我对助手本身没有太多的了解——默认情况下，可用的库非常有限，您必须为想要导入的任何其他库提供完整的源代码。系统调用是明确禁止的，如果模板进行系统调用，将会失败。

“但是大卫，”你说。"现在我已经有了我的助手函数，我该如何使用它？"

我很高兴你问了。现在我们必须用 Python 重写我们的 Jinja 模板，结果是:

```
from templates.helpers import projectNumberdef GenerateConfig(context):
    PROJECT_NUMBER = projectNumber.GetProjectNumber(context.env['project'])
    resources = [{
        'name': context.env['name'],
        'type': 'storage.v1.bucket',
        'properties': {
            'storageClass': context.properties['storageClass'],
            'acl': [{
                'role': 'OWNER',
                'entity': 'project-owners-' + PROJECT_NUMBER
                },{
                    'role': 'OWNER',
                    'entity': 'project-editors-' + PROJECT_NUMBER
                },{
                    'role': 'READER',
                    'entity': 'project-viewers-' + PROJECT_NUMBER
                }]
            }
        }]if 'owners' in context.properties:
        for owner in context.properties['owners']:
            resources.properties.acl.append({
                'role': 'OWNER',
                'entity': 'user-' + owner
                })if 'editors' in context.properties:
        for editor in context.properties['editors']:
            resources[0]['properties']['acl'].append({
                'role': 'WRITER',
                'entity': 'user-' + editor
                })if 'readers' in context.properties:
        for reader in context.properties['readers']:
            resources[0]['properties']['acl'].append({
                'role': 'READER',
                'entity': 'user-' + reader
                })return {'resources': resources}
```

您还必须记住在您的 YAML 文件中包含助手模块，以便一切正常工作:

```
imports:
- path: templates/bucket.py
- path: templates/helpers/projectNumber.pyresources:
- name: dav-bucket
  type: templates/svcaccount.jinja
  properties:
    storageClass: MULTI_REGIONAL
    owners:
    - dav-svcaccount@myproject.iam.gserviceaccount.com
```

您不能组合 Python 和 Jinja 模板——部署管理器会抱怨很多，并给出一些关于 Python 编译错误和 Jinja 文件的神秘错误。然而，一旦所有的模板都被转换了，它允许开发人员提供一个类似于下面的配置，并且允许运营人员随着情况的变化更新底层的模板。开发人员可以提供如下内容:

```
imports:
- path: templates/svcaccount.jinja
- path: templates/pubsub.jinja
- path: templates/bucket.py
- path: templates/helpers/projectNumber.pyresources:
- name: dav-svcaccount
  type: templates/svcaccount.jinja- name: dav-pubsub
  type: templates/vst-pubsub.jinja
  properties:
    serviceAccountEmail: "dav-svcaccount@myproject.iam.gserviceaccount.com"
                         # deployment still fails if email provided by reference :-(
    ackDeadlineSeconds: 600- name: dav-bucket
  type: templates/svcaccount.jinja
  properties:
    storageClass: MULTI_REGIONAL
    owners:
    - $(ref.dav-svcaccount.email)outputs:
- name: serviceAccountEmail
  value: $(ref.dav-svcaccount.email)
```

现在，我们真正开始了部署管理器之旅！我当然希望我的模板会随着我获得更多的经验而改进。我对这方面的了解很少，但是下一次我希望使用模板来简化负载平衡器的构建。