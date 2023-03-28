# 使用 Cloud Composer 计划 GCP STS 传输作业

> 原文：<https://medium.com/google-cloud/schedule-gcp-sts-transfer-job-with-cloud-composer-2aa947fb1948?source=collection_archive---------1----------------------->

存储传输服务可用于将数据从本地存储系统传输到 GCS 存储桶。它提供了许多有用的功能，如一次性和定期传输，数据完整性检查，元数据保存等。

存储传输服务目前支持一小时作为最小计划频率。如果需要以分钟级别的频率进行调度，那么我们需要使用云调度器和云合成器等调度工具。我的同事[bam bang Satrijotomo](/@bambang20)**写了一篇非常好的[文章](/@bambang20/schedule-google-cloud-sts-transfer-job-with-cloud-scheduler-7e25e9943e4e)介绍如何使用云调度器调度 STS 作业。云调度程序使用起来更简单，但是目前它还不支持像 VPC SC 这样的功能。如果用户环境中已经使用了 Cloud Composer，则建议使用 Cloud Composer。**

**在本文中，我们将实现一个 cloud composer dag，它将运行 STS 传输作业。我们将安排此 dag 每十分钟运行一次。**

# ****先决条件****

1.  **用户已经部署了 Cloud Composer 环境，并且能够计划他/她的 Dag。**
2.  **用户有一个现有的正在运行的 STS 作业，需要定期运行。**
3.  **获取 composer 工作节点使用的服务帐户。用户可以执行下面的 gcloud 命令来获得相同的结果:**

```
gcloud composer environments describe <composer_env_name> --location <region> --project <composer_env_project_id> --format json | jq '.config.nodeConfig.serviceAccount'
```

**4.获取 composer 存储其 Dag 的 GCS 位置。用户可以执行下面的 gcloud 命令来获得相同的结果:**

```
gcloud composer environments describe <composer_env_name> --location <region> --project <composer_env_project_id> --format json | jq '.config.dagGcsPrefix'
```

**5.将存储传输用户角色分配给 composer worker 服务帐户。**

```
gcloud projects add-iam-policy-binding <sts_job_project_id> --member='serviceAccount:<composer_worker_sa_email>' --role="roles/storagetransfer.user"
```

# **Dag 详细信息**

**使用下面的代码创建一个 DAG。根据您的环境替换项目 ID 和 TSOP 作业名称变量的值。我计划每 10 分钟运行一次 dag，用户可以根据自己的需要选择这个时间间隔。**

```
import datetime
import logging
import time

from airflow import models, utils as airflow_utils
from airflow.models import BaseOperator
from airflow.utils.decorators import apply_defaults
import googleapiclient.discovery
from oauth2client.client import GoogleCredentials

PROJECT_ID = <sts_job_project_id>
TSOP_JOB_NAME = <transfer_job_name>

DEFAULT_ARGS = {
    'owner': 'airflow',
    'start_date': airflow_utils.dates.days_ago(7),
}

class TSOPJobRunOperator(BaseOperator):
  """Runs the TSOP job."""

  @apply_defaults
  def __init__(self, project, job_name,  *args, **kwargs):
    self._tsop = None
    self.project = project
    self.job_name = job_name
    super(TSOPJobRunOperator, self).__init__(*args, **kwargs)

  def get_tsop_api_client(self):
    if self._tsop is None:
      credentials = GoogleCredentials.get_application_default()
      self._tsop = googleapiclient.discovery.build(
        'storagetransfer', 'v1', cache_discovery=False,       
        credentials=credentials)
    return self._tsop

  def execute(self, context):
    logging.info('Running Job %s in project %s',
                 self.job_name, self.project)
    self.get_tsop_api_client().transferJobs().run(
      jobName=self.job_name,
      body={'project_id': self.project}).execute()

with models.DAG(
    "run_tsop_transfer_job",
    default_args=DEFAULT_ARGS,
    schedule_interval='*/10 * * * *'
    tags=["tsop_job"],
    user_defined_macros={"TSOP_JOB_NAME": TSOP_JOB_NAME},
) as dag:
    run_tsop_job = TSOPJobRunOperator(
        project=PROJECT_ID, job_name=TSOP_JOB_NAME, 
        task_id='run_tsop_job')
```

**将 dag 上传到 composer dags 存储桶文件夹。为了进行测试，从 composer UI 中触发 DAG。DAG 应该成功完成。可以从云控制台验证 STS 作业运行。**

**欢迎建议和评论！！！快乐阅读。**