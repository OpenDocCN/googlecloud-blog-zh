# 如何通过 Firebase 的云函数启动数据流管道

> 原文：<https://medium.com/google-cloud/how-to-kick-off-a-dataflow-pipeline-via-cloud-functions-696927975d4e?source=collection_archive---------0----------------------->

我们在 Soru 的数据管道依赖于谷歌数据流。我选择它而不是 Dataproc，因为在我们的小团队中，无服务器架构的便利性非常重要。Apache Beam 与我在谷歌经常使用的内部工具 Flume 的相似性也在这个选择中发挥了作用。

我们的数据流管道是用 Python 编写的。我通常从我的本地计算机开始一次性工作，直到它准备好集成到我们的生产管道中。我将在另一篇博文中讲述如何为各种用例构建数据流管道。

以编程方式从另一个系统调用数据流管道的最简单方法是

*   创建您的自定义数据流模板。这里是[指令](https://cloud.google.com/dataflow/docs/templates/creating-templates)。

请务必仔细阅读文档。SDK 中可能有一些限制。

例如，在 Python SDK 中，只有基于文件的 IOs 支持 ValueProvider。当我对管道进行模板化时，我必须将输入参数从 BigQuery 更改为 TextIO。参见 https://issues.apache.org/jira/browse/BEAM-1440 的

*   将其部署到 GCS 存储桶。我写了一个简单的部署脚本，因为我们有测试、试运行和生产项目，我们需要将我们的系统部署到其中的每一个项目。

```
def deploy(project, bucket, template_name):
    command = '''
        python -m {template_name} \
            --runner DataflowRunner \
            --project {project} \
            --requirements_file requirements.txt \
            --staging_location gs://{bucket}/staging \
            --temp_location gs://{bucket}/temp \
            --template_location \
                gs://{bucket}/templates/{template_name}
    '''.format(project=project, bucket=bucket,
               template_name=template_name)
    os.system(command)
    os.system('''
        gsutil cp {template_name}_metadata \
            gs://{bucket}/templates/
    '''.format(bucket=bucket, template_name=template_name))
```

这个脚本假设您有<template></template>

*   通过 REST API 调用管道。

最近，我们越来越依赖无服务器架构。我们在 Node.js 6 运行时使用 Firebase 云函数，所以我们需要从那里开始处理数据流模板。不幸的是，客户端库支持有点挑剔。我必须说，做这件事一点也不好玩，所以我不想麻烦你。顺便说一下，这段代码很可能也适用于 GCP 的云函数，尽管我还没有测试过。

下面是我们的代码大致的样子。我们依赖“Google APIs”:“33 . 0 . 0”。

```
const { google } = require('googleapis');
const dataflow = google.dataflow('v1b3');const TEMPLATE_BUCKET = `your template bucket on GCS`;const kickOffDataflow = (input, output) => {
  var jobName = `your job name`;
  var tmpLocation = `gs://${TEMPLATE_BUCKET}/tmp`;
  var templatePath = `gs://${TEMPLATE_BUCKET}/templates/` +
    `your_template_name`;
  var request = {
    projectId: process.env.GCLOUD_PROJECT,
    requestBody: {
      jobName: jobName,
      parameters: {
        input: input,
        output: output
      },
      environment: {
        tempLocation: tmpLocation
      }
    },
    gcsPath: templatePath
  }
  return
    google.auth.getClient({
      scopes: ['[https://www.googleapis.com/auth/cloud-platform'](https://www.googleapis.com/auth/cloud-platform')]
    })
    .then(auth => {
      request.auth = auth;
      return dataflow.projects.templates.launch(request);
    })
    .catch(error => {
      console.error(error);
      throw error;
    });
}
```

让我知道这是否有帮助，并随时提出改进建议。编码快乐！