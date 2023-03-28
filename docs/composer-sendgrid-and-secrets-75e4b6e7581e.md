# 作曲家、发送网格和秘密

> 原文：<https://medium.com/google-cloud/composer-sendgrid-and-secrets-75e4b6e7581e?source=collection_archive---------0----------------------->

![](img/d9d1f1ae7d7055843d26e834fd181e43.png)

一位客户希望使用 GCP 作曲家(气流)在工作流失败时通过 Sendgrid 发送通知电子邮件。当一个人希望使用 Sendgrid 时，他必须提供一个 ApiKey 来证明你被授权发送电子邮件。客户端希望将 Sendgrid API 密钥存储在 GCP 机密中。本文向我们展示了实现这一目标的秘诀。

首先，为了让工作流在失败时自动发送通知电子邮件，必须在 DAG 中声明这一点。

下面是一个这样做的 DAG 示例:

```
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.exceptions import AirflowFailException# These args will get passed on to each operator
# You can override them on a per-task basis during operator initialization
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'email': ['[jones@example.com](mailto:jones@example.com)'],
    'email_on_failure': True,
    'email_on_retry': True,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}
with DAG(
    'tutorial',
    default_args=default_args,
    description='A simple DAG',
    schedule_interval=timedelta(days=1),
    start_date=datetime(2021, 1, 1),
    catchup=False,
    tags=['example'],
) as dag:t1 = BashOperator(
      task_id='print_date',
      bash_command='date',
    )def task_to_fail():
        raise AirflowFailException("Forced Exception!")t2 = PythonOperator(dag=dag,
      task_id='throw_exception',
      python_callable=task_to_fail
    )t1 >> t2
```

我想让你看的重点是:

*   DAG 会抛出异常，因此整个工作流会失败。
*   “`email_on_failure`”标志被设置为真。
*   在“`email`”列表中提供了一个电子邮件地址，通知电子邮件将发送到该地址。

如果我们运行此 DAG，我们会发现它会按预期失败，但是还有更多工作要做。我们还没有告诉作曲家如何发送电子邮件。发送电子邮件有多种方式，包括 SMTP、Sendgrid 等。

气流主要由名为`airflow.cfg`的配置文件驱动。这个文件基本上是一个名称/值对集，分成相关的部分。其中一个部分专门针对内部气流产生的电子邮件。两个参数是我们最感兴趣的。第一种叫做`email_backend`。这指定了发送电子邮件时要调用的逻辑。这使我们能够定制如何发送电子邮件。Apache 已经提供了这样一个插件，它提供了到 Sendgrid 的钩子。第二个感兴趣的参数称为“`email_conn_id`”，用于提供气流连接名称。这个连接应该保存由`email_backend`引用的电子邮件子系统所需的参数。此连接 id 将用于获取允许气流使用 Sendgrid 所需的信息。出于我们的目的，我们希望`airflow.cfg`现在包括:

```
[email]
email_backend=airflow.providers.sendgrid.utils.emailer.send_email
email_conn_id=sendgrid_default
```

在菜谱的后面，我们将展示如何在 Composer 环境中设置这些值。

当通过 Airflow 发送电子邮件时，该电子邮件将被视为源自某个源电子邮件地址，并且具有与该地址相关联的名称。如果这些参数是连接 id 的一部分就更好了，但遗憾的是事实并非如此。相反，使用两个环境变量来提供这些值:

*   SENDGRID_MAIL_FROM=mycomposer@example.com
*   SENDGRID _ MAIL _ SENDER =作曲家

可能需要配置 Sendgrid 来授权来自源地址的电子邮件。

此时，我们可以创建气流连接定义，我们的解决方案将运行；然而，我们想利用 GCP 秘密作为我们的安全信息的金库。为此，我们需要在我们的`airflow.cfg`文件中增加一个条目。

为了能够使用 Secret manager，我们需要修改`airflow.cfg`以在 secrets 部分包含一个名为`secrets`的条目。这是用来命名一些插件逻辑，提供了一个存储秘密的钩子。气流为 GCP 秘密经理提供了一个插件。因此，`airflow.cfg`中的条目应为:

```
[secrets]
backend=airflow.providers.google.cloud.secrets.secret_manager.CloudSecretManagerBackend
```

当这被配置时，当需要秘密时，气流将咨询 GCP 秘密管理器。连接定义也被视为机密。如果我们在 Secret Manager 中创建一个适当命名的条目，Airflow 将使用该机密的值作为连接 Id。该值必须以连接 Id URL 格式编码。

对于 Sendgrid，我们可以创建以下格式的值:

```
sendgrid://username:[SOME_API_KEY]@host
```

值中只检查 API 键。秘密的名字也是极其重要的。连接 Id 的机密名称的结构是:

```
airflow-connections-[CONNECTION_ID]
```

就我们的目的而言，这意味着:

```
airflow-connections-sendgrid_default
```

就这样…所有的部分都连接在一起了。

现在，我们将演示所有部件。

1.  创建新的 Composer 环境

创建一个新的 GCP 作曲家环境。接受大多数默认值。在我的测试中，我使用的是气流版本 2.1.4。在确认之前，请确保指定气流配置覆盖和环境变量的值:

气流配置覆盖

```
[email]
email_backend=airflow.providers.sendgrid.utils.emailer.send_email
email_conn_id=sendgrid_default
[secrets]
backend=airflow.providers.google.cloud.secrets.secret_manager.CloudSecretManagerBackend
[logging]
logging_level=DEBUG
```

(注意，日志条目不是必需的，但是根据我的经验，它在开发/测试期间非常有价值)

环境变量

*   SENDGRID_MAIL_FROM=mycomposer@example.com
*   SENDGRID _ MAIL _ SENDER =作曲家

![](img/e1ea30d7a07bfb08ae490503a286257b.png)

创建 Composer 环境可能需要 10-20 分钟。

2.在 GCP 秘密管理器中创建一个秘密

在 GCP 秘密管理器中，创建一个名为`airflow-connections-sendgrid_default`的新秘密。将其值设置为

```
sendgrid://username:[SENDGRID_API_KEY]@host
```

3.授权作曲家阅读秘密

在运行时，Composer 需要能够从秘密管理器中读取秘密的值。确保添加了 IAM 权限以允许访问。

4.运行失败的 DAG

如果我们现在运行一个配置为在失败时发送电子邮件的失败 DAG，我们会发现已经发送了一封电子邮件。

另请参见:

*   [气流:邮件配置](https://airflow.apache.org/docs/apache-airflow/stable/howto/email-config.html)
*   [气流:配置—电子邮件](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#email)
*   [Docs:为您的环境配置 Secret Manager】](https://cloud.google.com/composer/docs/composer-2/configure-secret-manager)