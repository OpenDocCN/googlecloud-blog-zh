# 通过使用 Cloud Composer 关闭 GCE 实例来节省资金

> 原文：<https://medium.com/google-cloud/save-money-by-shuting-down-gce-instances-using-cloud-composer-e70d7a3f6c98?source=collection_archive---------1----------------------->

大多数 GCP 用户使用计算引擎资源，这使得在 GCP 上设置不同大小和风格的虚拟机变得很容易。这些资源是根据它们的运行时间计费的。通过明智地减少这些 GCE 实例的运行时间，可以节省大量资金。

我们可以通过多种方式优化 GCE 成本。其中之一是在 GCE 实例不被使用时关闭它们。例如，我们可以在周末关闭“开发”环境中的所有实例。这可以为我们节省很多钱。最好的方法是使用 GCP 上可用的工具和服务实现自动化。有一个非常好的[指南](https://cloud.google.com/scheduler/docs/start-and-stop-compute-engine-instances-on-a-schedule)使用云调度器、发布/订阅和云函数来按照给定的时间表停止和启动实例。

本文重点介绍如何使用 Cloud Composer 来调度 Dag 以停止和启动实例。Cloud Scheduler+Pub/Sub+Cloud Function 组合和 Cloud Composer 选择哪一个取决于很多因素，我们不打算在本文中详细讨论。一个因素可能是，如果您的环境中已经在使用 Cloud Composer，那么对这种自动化也使用相同的组件是有意义的。

# 先决条件

1.  用户已经部署了 Cloud Composer 环境，并且能够计划他/她的 Dag。
2.  获取 composer 工作节点使用的服务帐户。用户可以执行下面的 gcloud 命令来获得相同的结果:

```
gcloud composer environments describe <composer_env_name> --location <region> --project <composer_env_project_id> --format json | jq '.config.nodeConfig.serviceAccount'
```

3.为 composer worker 服务帐户分配计算实例管理员角色，以便 Dag 停止/启动实例。

```
gcloud projects add-iam-policy-binding <compute_instance_project_id> --member='serviceAccount:<composer_worker_sa_email>' --role="roles/compute.instanceAdmin.v1"
```

4.获取 composer 存储其 Dag 的 GCS 位置。用户可以执行下面的 gcloud 命令来获得相同的结果。我们将使用这个桶来上传 Dag。

```
gcloud composer environments describe <composer_env_name> --location <region> --project <composer_env_project_id> --format json | jq '.config.dagGcsPrefix'
```

# Dag 详细信息

使用以下代码创建 DAG 以停止实例。根据您的 env 替换 PROJECT_ID 和 ZONE 变量的值。这将在每周星期六上午 12 点停止给定项目和区域中带有标签“env:dev”的所有实例。

```
**import** **logging**

**from** **airflow** **import** models, utils **as** airflow_utils
**from** **airflow.models** **import** BaseOperator
**from** **airflow.utils.decorators** **import** apply_defaults
**from** **airflow.operators.dummy_operator** **import** DummyOperator
**import** **googleapiclient.discovery**
**from** **oauth2client.client** **import** GoogleCredentials

ENVIRONMENT = "dev"
PROJECT_ID = <project_id>
ZONE = <zone>

DEFAULT_ARGS = {
    'owner': 'airflow',
    'start_date': airflow_utils.dates.days_ago(7)
}

**class** **StopInstanceOperator**(BaseOperator):
    *"""Stops the virtual machine instances."""*
    @apply_defaults
    **def** __init__(self, project, zone, *args, **kwargs):
        self._compute = **None**
        self.project = project
        self.zone = zone
        super(StopInstanceOperator, self).__init__(*args, **kwargs)

    **def** get_compute_api_client(self):
        **if** self._compute **is** **None**:
            credentials=GoogleCredentials.get_application_default()
            self._compute = googleapiclient.discovery.build(
                'compute', 'v1', cache_discovery=**False**,
                credentials=credentials)
        **return** self._compute

    **def** list_instances(self):
        instance_res=self.get_compute_api_client().instances().list(
            project=self.project, zone=self.zone,
            filter=f"labels.env={ENVIRONMENT}").execute()
        **return** [instance['name'] 
                **for** instance **in** instance_res.get('items', [])]

    **def** execute(self, context):
        **for** instance **in** self.list_instances():
            logging.info(
                'Stopping instance %s in project %s and zone %s',
                instance, self.project, self.zone)
            self.get_compute_api_client().instances().stop(
                project=self.project, zone=self.zone,   
                instance=instance).execute()

**with** models.DAG(
    "stop_gce_instances",
    default_args=DEFAULT_ARGS,
    schedule_interval='0 0 * * 6',
    tags=["stop_gce_instances"],
) **as** dag:
    begin = DummyOperator(task_id='begin')
    end = DummyOperator(task_id='end')
    stop_gce_instances = StopInstanceOperator(
        project=PROJECT_ID, zone=ZONE, task_id='stop_gce_instances')
    begin >> stop_gce_instances >> end
```

使用以下代码创建 DAG 以启动实例。根据您的 env 替换 PROJECT_ID 和 ZONE 变量的值。这将在每周一上午 5 点启动给定项目和区域中标签为“env:dev”的所有实例。

```
**import** **logging**

**from** **airflow** **import** models, utils **as** airflow_utils
**from** **airflow.models** **import** BaseOperator
**from** **airflow.utils.decorators** **import** apply_defaults
**from** **airflow.operators.dummy_operator** **import** DummyOperator
**import** **googleapiclient.discovery**
**from** **oauth2client.client** **import** GoogleCredentials

ENVIRONMENT = "dev"
PROJECT_ID = <project_id>
ZONE = <zone>

DEFAULT_ARGS = {
    'owner': 'airflow',
    'start_date': airflow_utils.dates.days_ago(7)
}

**class** **StartInstanceOperator**(BaseOperator):
    *"""Start the virtual machine instances."""*
    @apply_defaults
    **def** __init__(self, project, zone, *args, **kwargs):
        self._compute = **None**
        self.project = project
        self.zone = zone
        super(StartInstanceOperator, self).__init__(*args, **kwargs)

    **def** get_compute_api_client(self):
        **if** self._compute **is** **None**:
            credentials= GoogleCredentials.get_application_default()
            self._compute = googleapiclient.discovery.build(
                'compute', 'v1', cache_discovery=**False**, 
                credentials=credentials)
        **return** self._compute

    **def** list_instances(self):
        instance_res=self.get_compute_api_client().instances().list(
            project=self.project, zone=self.zone, 
            filter=f"labels.env={ENVIRONMENT}").execute()
        **return** [instance['name'] 
                **for** instance **in** instance_res.get('items', [])]

    **def** execute(self, context):
        **for** instance **in** self.list_instances():
            logging.info(
                'Starting instance %s in project %s and zone %s',
                instance, self.project, self.zone)
            self.get_compute_api_client().instances().start(
                project=self.project, zone=self.zone, 
                instance=instance).execute()

**with** models.DAG(
    "start_gce_instances",
    default_args=DEFAULT_ARGS,
    schedule_interval='0 5 * * 1',
    tags=["start_gce_instances"],
) **as** dag:
    begin = DummyOperator(task_id='begin')
    end = DummyOperator(task_id='end')
    start_gce_instances = StartInstanceOperator(
        project=PROJECT_ID, zone=ZONE,  
        task_id='start_gce_instances')
    begin >> start_gce_instances >> end
```

将 dag 上传到 composer dags 存储桶文件夹。为了进行测试，从 composer UI 中触发 Dag。DAG 应该成功完成。

我借用了这篇[文章](https://cloud.google.com/architecture/automating-infrastructure-using-cloud-composer)的大部分代码，做了非常小的修改。

省钱！！快乐阅读！！！欢迎评论和建议。