# KubernetesPodOperator 示例

> 原文：<https://medium.com/google-cloud/load-xlsx-data-from-google-drive-and-load-it-to-bigquery-using-airflow-ce9bb6c1d93a?source=collection_archive---------2----------------------->

当试图从 Google Drive 获取数据时，我发现了几个困难，问题主要在于为我们提供的凭据定义正确的范围。我要感谢我的同事[崇杰林](https://medium.com/u/d36e998414a0?source=post_page-----ce9bb6c1d93a--------------------------------)在这个方法上帮助了我。

所以在这里，我们想分享一种利用气流完成工作的方法。

在这篇文章中，我使用了 3 个不同的操作符。

1.  [KubernetesPodOperator](https://airflow.apache.org/docs/stable/kubernetes.html) ，用于从 Google Drive 下载数据，进行数据转换，并将数据以 CSV 的形式上传到 GCS。
2.  [GoogleCloudStorageObjectSensor](https://airflow.apache.org/docs/stable/_modules/airflow/contrib/sensors/gcs_sensor.html)，用于检查文件是否已经存在于 GCS 中。
3.  [GoogleCloudStorageToBigQueryOperator](https://airflow.apache.org/docs/stable/_modules/airflow/contrib/operators/gcs_to_bq.html)，用于将 CSV 格式的数据加载到 Bigquery 中。

在使用 KubernetesPodOperator 之前，应该先创建一个 Dockerimage。在 Dockerimage 内部，我们可以定义需要做的任何事情，并提供可以传递给 dockerized 代码的参数。

**从 Google Drive 下载数据**

基本上，为了能够访问，你可以使用一个服务帐户。此外，我们只能根据文件的 id 来下载文件，因此，如果我们说您只知道文件名，那么您只能在知道 id 后才能下载它，这就是为什么我们需要通过执行搜索查询来找到它。要进一步了解这一点，您可以参考 [Python 快速入门。](https://developers.google.com/drive/api/v3/quickstart/python)

基本上有两种检索 file_id 的有用方法:
(1)如果我们知道 folder_id
，则基于文件名(2)基于查询

我们使用的以下方法是第二种方法。

```
class Constants:
    SERVICE_ACCOUNT_PATH = 'GOOGLE_APPLICATION_CREDENTIALS'
    OAUTH_TOKEN_URI = '[https://www.googleapis.com/oauth2/v4/token'](https://www.googleapis.com/oauth2/v4/token')class NoQueryResultsException(Exception):
    """No Files matching query in Google Drive"""class GoogleSuiteApi:
    def __init__(self):
        self.service_account_credentials = self._service_account_credentials()
        self.service = self._service_build()# [https://stackoverflow.com/questions/49663359/how-to-upload-file-to-google-drive-with-service-account-credential](https://stackoverflow.com/questions/49663359/how-to-upload-file-to-google-drive-with-service-account-credential)
    def _service_build(self):
        service = build('drive', 'v3', credentials=self.service_account_credentials)
        return service[@staticmethod](http://twitter.com/staticmethod)
    def _service_account_credentials():
        service_account_key_path = os.getenv(
            Constants.SERVICE_ACCOUNT_PATH)credentials = service_account.Credentials.from_service_account_file(
            service_account_key_path)
        scoped_credentials = credentials.with_scopes(
            [Constants.OAUTH_TOKEN_URI])
        signer_email = scoped_credentials.service_account_email
        signer = scoped_credentials.signercredentials = google.oauth2.service_account.Credentials(
            signer,
            signer_email,
            token_uri=Constants.OAUTH_TOKEN_URI
        )
        return credentialsdef get_metadata(self, folder_id):
        query = <whatever-your-query-is>
        try:
            get_files = self.service.files().list(q=query,
                          orderBy='modifiedTime desc',
                          pageSize=1).execute().get('files')
        except errors.HttpError as e:
            print ('An error occurred: %s' % e) if len(get_files) == 0:
            raise NoQueryResultsException(
                'No files in Google Drive matching query {query}'.format(query=query))get_files.sort(key=lambda x: x['name'], reverse=True) print(get_files[0])
        return get_files[0]def download_google_document_from_drive(self, file_id,
        export_mime_type=None):
        """
        file_id          :  Google Drive fileId
        export_mime_type :  To be provided if target file is a Google Drive native file.
                            Otherwise, leave it as None
        """
        try:
            if export_mime_type is None:
                request = self.service.files().get_media(fileId=file_id)
            else:
                request = self.service.files().export_media(fileId=file_id,
                                                            mimeType=export_mime_type)
            fh = io.BytesIO()
            downloader = MediaIoBaseDownload(fh, request)
            done = False
            while done is False:
                status, done = downloader.next_chunk()
                print('Download %d%%.' % int(status.progress() * 100))
            return fh
        except Exception as e:
            print('Error downloading file from Google Drive: %s' % e)
```

**数据处理**

获得所需的文件后，您可以通过首先将 xlsx 数据转换为 pandas 来执行处理，并执行必要的 join 或任何其他操作。

```
*import* xlrd
*import* pandas *as* pd*def* convert_xls_to_pandas(file_stream, sheet_name, column_mapping,
    column_types):
    workbook = xlrd.open_workbook(file_contents=file_stream.getvalue())
    df = pd.read_excel(workbook, sheet_name=sheet_name,
                       engine=xlrd)
    df = (df.loc[:, column_mapping.keys()]
          .copy()
          .rename(columns=column_mapping)
          .astype(column_types)
          )
    *return* df
```

注意，在转换为 pandas 时，您还可以为名称和数据类型定义列映射。之后你可以将熊猫转换成 csv。

```
*def* convert_pandas_to_csv(df, csv_name):
    df.to_csv(csv_name, index=*False*, header=*True*)
```

**上传数据到 GCS**

```
*import* google.auth
*from* google.cloud *import* storage*def* upload_file_to_gcs(bucket, input_filepath, output_filename):
    credentials, project = google.auth.default()
    *if* credentials.requires_scopes:
        credentials = credentials.with_scopes(
            ['https://www.googleapis.com/auth/devstorage.read_write'])
    client = storage.Client(credentials=credentials, project=project)

    *try*:
        *try*:
            bucket = client.get_bucket(bucket)
        *except*:
            bucket = client.create_bucket(bucket, project=project)
        blob = bucket.blob(output_filename)
        blob.upload_from_filename(input_filepath)
        *print*('Upload to GCS complete')
    *except* Exception *as* e:
        *print*('An error occurred: %s' % e)

    *return None*
```

为了能够获得访问权，您需要设置`GOOGLE_APPLICATION_CREDENTIALS`环境变量，以及编写必要的[范围](https://cloud.google.com/storage/docs/authentication)。

之后，您可以创建一个带有特定`ENTRYPOINT`的 Dockerimage，使您能够传递一些参数。

**编写气流脚本**

请注意，根据文档，定义一个秘密会自动将该秘密装载到您定义的某个路径中。但是，您也可以将它定义为一个环境变量，而不是将它作为一个卷来挂载。有关更多信息，您可以参考此[示例](https://github.com/GoogleCloudPlatform/python-docs-samples/blob/master/composer/workflows/kubernetes_pod_operator.py)。

```
from airflow.contrib.operators.kubernetes_pod_operator import \
    KubernetesPodOperator
from airflow.contrib.kubernetes.secret import Secretsvc_acc = Secret('volume', '<path-to-mount-your-service-account>', '<secret-where-you-save-your-service-account>',
                         '<your-service-account-name>')
environments = {
    'GOOGLE_APPLICATION_CREDENTIALS': Constants.GOOGLE_APPLICATION_CREDENTIALS}
xlsx_processing = KubernetesPodOperator(
    task_id='<your-task-id-name>',
    image=<your-docker-image>,
    namespace='<your-namespace>',
    name='<your-k8s-job-name>',
    image_pull_policy='Always',
    is_delete_operator_pod=True,
    arguments=[
        <your-args>
    ],
    secrets=[svc_acc],
    env_vars=environments
)
```

您可以将项目的下游设置为`GoogleCloudStorageObjectSensor`。然后把下游设定成`GoogleCloudStorageToBigQueryOperator`。这里我将给出一个例子，如果你想基于给定的模式而不是`autodetect`加载数据。

```
gcs_to_bq_task = GoogleCloudStorageToBigQueryOperator(
    task_id=<your-task-name>,
    bucket=<your-bucket>,
    source_objects=[
        <your-file-path>
    ],
    destination_project_dataset_table=<your-table-name>,
    schema_fields=<your-schema>,
    skip_leading_rows=1, #skip the header
    source_format='CSV',
    write_disposition='WRITE_TRUNCATE',
    google_cloud_storage_conn_id=<your-connection>,
    bigquery_conn_id=<your-connection>,
    autodetect=False #This is because you're using your own schema
)
```

您可以用以下格式定义您的模式:

```
sample_schema = [
        {'name': '<column-name>', 'type': '<bq-data-type>', 'mode': '<mode, can be REQUIRED or NULLABLE>'},
]
```

关于可用数据类型的更多解释，您可以参考第[页](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types)。

最重要的是确保你的气流进入你的集装箱登记处。此外，如果您定义自己的`ConfigMap`或`Secret`，您可以在您的气流中创建它们，如果您在 Kubernetes 上运行您的气流，这也会容易得多。我想就这些了..希望有帮助！