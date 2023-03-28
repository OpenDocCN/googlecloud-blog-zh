# 谷歌云平台搜索—云 SQL

> 原文：<https://medium.com/google-cloud/search-on-google-cloud-platform-cloud-sql-65a0f75b3af8?source=collection_archive---------4----------------------->

作为主题“Google 云平台上的搜索”系列的继续，在本文中，我将探索作为存储的[云 SQL](https://cloud.google.com/sql/) ，即用于文本搜索的 PostgreSQL 内置功能。本系列之前的文章/解决方案包括:

*   [在谷歌云平台上搜索——应用引擎和搜索 API](/google-cloud/search-on-google-cloud-platform-app-engine-and-search-api-31cda6917bbf)
*   [搜索谷歌云平台——云数据存储](/google-cloud/search-on-google-cloud-platform-cloud-datastore-615c14cb1bb)

我将重复以前文章中的任务描述:“在 Google 云平台上为 eshop 创建自动完成功能，这样当用户搜索一些文本时，它会返回包含该文本的产品名称”。在现实生活中，我使用的是百思买的产品数据。呈现搜索页面和结果的 web 服务器是在 Flask framework (Python)中完成的，并部署在 Google App Engine Flexible 上。我使用灵活而非标准的原因是因为目前无法从 GAE 标准连接到云 SQL PostgreSQL。

该项目的代码在 [Github](https://github.com/zdenulo/gcp-search/tree/master/cloud_sql) 库中。

# 谷歌云 SQL

云 SQL 于 2011 年推出，最初支持 MySQL，2017 年推出对 PostgreSQL 的支持，从现在起，我将在 PostgreSQL 的上下文中使用云 SQL 的用法和属性。云 SQL 是完全托管的数据库，这意味着服务器和磁盘存储是自动调配的。它还有其他特点，如:

*   自动备份，可以定义每天的备份窗口。
*   自动存储增加，所以不应该发生你用完磁盘空间
*   通过不同区域中的副本实现高可用性
*   目前支持 PostgreSQL 9.6，目前官方版本是 10，11 是测试版
*   自定义 CPU 和内存配置，从 1 个共享 CPU 和 0.6GB RAM 开始，最多 64 个 vcpu 和 416 GB RAM 内存
*   存储容量从 10GB 到 10TB 不等
*   最低配置从每月 10 美元起，其他服务如自动备份、副本当然需要额外费用
*   从 onpremise 或其他云提供商迁移数据库
*   网络访问规则，可以定义访问数据库实例的 IP 地址(范围),也可以使用云 sql 代理[https://cloud.google.com/sql/docs/postgres/sql-proxy](https://cloud.google.com/sql/docs/postgres/sql-proxy)

# 数据库设置

我将使用 [Google Cloud SDK](https://cloud.google.com/sdk/) 来执行命令行操作，因此首先我将创建名为“myinstance”的数据库实例:

```
gcloud sql instances create myinstance --database-version=POSTGRES_9_6 --cpu=1 --memory=3840MiBCreated [https://www.googleapis.com/sql/v1beta4/projects/adventures-on-gcp/instances/myinstance].
NAME DATABASE_VERSION LOCATION TIER ADDRESS STATUS
myinstance POSTGRES_9_6 us-central1-a db-custom-1-3840 35.232.43.115 RUNNABLE
```

注意:如果您的配置中没有设置 zone 和 region，您可以在那里设置它，或者可以使用— zone 和— region 标志来指定数据库实例所在的区域和 zone。

注 2:最好将 web 服务器和数据库实例放在同一个区域中，以获得最佳性能/最小延迟。

下一步是在数据库实例中创建数据库:

```
gcloud sql databases create eshop --instance=myinstance

Creating Cloud SQL database...done. 
Created database [eshop].
instance: myinstance
name: eshop
project: adventures-on-gcp
```

默认用户 **postgres** 是自动创建的，我将在这个例子中使用它，但是我需要为它设置密码。

```
gcloud sql users set-password postgres --instance myinstance --host=% --prompt-for-password
```

# 数据库模型

所有与数据库相关的东西都在 sql_search.py 文件中。值得一提的是，我使用了 [**SQLAlchemy**](https://www.sqlalchemy.org/) 作为 ORM，并将[**SQLAlchemy-Searchable**](https://sqlalchemy-searchable.readthedocs.io/en/latest/index.html)集成到 SQLAlchemy 模型中。sql_search.py 文件的内容:

```
import os
import sqlalchemy
from sqlalchemy import create_engine, Column, String, Float, BigInteger
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy_searchable import make_searchable
from sqlalchemy_utils.types import TSVectorType
from sqlalchemy_searchable import search
from sqlalchemy_searchable import sync_trigger

from db_settings import USERNAME, PASSWORD, DB_NAME

SQLALCHEMY_DATABASE_URI = os.environ['SQLALCHEMY_DATABASE_URI']

if not SQLALCHEMY_DATABASE_URI:
    SQLALCHEMY_DATABASE_URI='postgresql+psycopg2://{USER_NAME}:{PASSWORD}@127.0.0.1:5431/{DATABASE_NAME}'.format(
        USER_NAME=USERNAME, PASSWORD=PASSWORD, DATABASE_NAME=DB_NAME
    )

# these are some necessary lines
Base = declarative_base()
make_searchable(Base.metadata)
engine = create_engine(SQLALCHEMY_DATABASE_URI, echo=True)
sqlalchemy.orm.configure_mappers()

Session = sessionmaker(bind=engine)
session = Session()

class Product(Base):
    """Table for products"""
    __tablename__ = 'products'

    sku = Column(BigInteger, primary_key=True)
    product_name = Column(String)
    price = Column(Float)
    sale_price = Column(Float)
    type = Column(String)
    url = Column(String)
    image = Column(String)
    available = Column(String)
    search_vector = Column(TSVectorType('product_name', ))  # this field is for full text search

    def __repr__(self):
        return self.product_name

class PostgreSQLSearch():
    def init_schema(self):
        """creates table"""
        # configure_mappers()
        session.commit()  # with out this line, Flask hangs for some reason
        Base.metadata.create_all(engine)
        sync_trigger(engine, 'products', 'search_vector', ['product_name'])

    def delete_all(self):
        """deletes table"""
        session.commit()  # with out this line, Flask hangs for some reason
        Product.__table__.drop(engine)

    def insert_bulk(self, product_data):
        """creates multiple row (objects) and commits into database"""
        objects = []
        for product in product_data:
            if product.get('name', ''):
                product_db = Product(sku=product['sku'], product_name=product['name'], price=product['price'],
                                     url=product['url'], type=product['type'], available=product['available'],
                                     sale_price=product['sale_price'], image=product['image']
                                     )
                objects.append(product_db)
        session.bulk_save_objects(objects)
        session.commit()

    def search(self, search_query):
        """making query to database"""
        query = session.query(Product)
        query = search(query, search_query, sort=True)
        results = query.limit(20).all()
        output = []
        for item in results:
            out = {
                'value': item.product_name,
                'label': item.product_name,
                'sku': item.sku
            }
            output.append(out)
        return output

if __name__ == '__main__':
    db_client = PostgreSQLSearch()
    db_client.init_schema()
```

我定义了具有一些字段和特殊字段的类产品是 **search_vector** ，其中是为文本搜索定义的向量和应该被索引的字段。因此，当保存实例(插入行)时，它会自动处理来自 product_name 的文本并填充 search_vector 字段。除此之外，没有什么特别的，我创建了批量上传，查询，创建和删除表的方法。

## 上传数据

我们有 120 万行产品数据，由于 SQLAlchemy searchable 只有在对象通过 SQLAlchemy 保存时才有效，因此很遗憾，直接导入大量 csv 文件是不可能的。这就是为什么我编写脚本来创建对象并保存到数据库中(这不是最有效的方式，但我想我没有选择)。另一件事是，因为我希望它尽可能快，我会在同一个区域创建服务器。(尝试从您的本地计算机上传，您会注意到很大的不同，尤其是如果数据库实例在地球的另一半)。

要创建 Google 计算引擎实例，命令是:

```
gcloud compute instances create instance-1 --scopes=sql-admin --preemtible
Created [https://www.googleapis.com/compute/v1/projects/adventures-on-gcp/zones/us-central1-a/instances/instance-1].
NAME        ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP   STATUS
instance-1  us-central1-a  n1-standard-1               10.128.0.4   35.206.91.84  RUNNING
```

我正在添加计算引擎的特定范围，以便它可以被授权使用云 SQL，我正在使用 preemtible，因为这将花费不到几个小时的时间……每一分都很重要。

我将复制整个存储库的代码，因为其中有我的 csv 文件和产品数据。这可能需要几分钟，因为有许多文件，更简单的是压缩它，只复制一个文件，然后解压缩。

将文件夹从本地计算机复制到服务器:

```
gcloud compute scp --recurse gcp-search instance-1:~/
```

ssh 进入服务器

```
gcloud compute ssh instance-1
```

正在安装 pip，这样我就可以安装 Python 的其余依赖项了:

```
sudo apt-get install python-pip
```

转到服务器上的 cloud_sql/webapp 文件夹并安装 python 依赖项:

```
pip install -r requirements.txt
```

最后，检查存储库根目录下 settings.py 中的变量 PRODUCTS_LOCAL_PATH 是否反映了服务器上的路径，如果没有，则更新它。

另一个重要部分是处理到数据库实例的连接。就像我在开始写的，一种可能性是允许特定的 IP 地址(我们的服务器)访问，或者使用云 SQL 的代理。我将使用第二种选择。第一步是使用命令下载代理的二进制文件:

```
wget [https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64](https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64) -O cloud_sql_proxy
```

将其更改为可执行:

```
chmod +x cloud_sql_proxy
```

我们需要获取云 sql 数据库实例的“实例连接名称”:

```
gcloud sql instances describe myinstance
backendType: SECOND_GEN
connectionName: adventures-on-gcp:us-central1:myinstance
databaseVersion: POSTGRES_9_6
...
```

第二行(connectionName)包含我们需要的信息。连接名通常采用 <project-id>: <region>: <instance-name>的格式。</instance-name></region></project-id>

最好打开另一个终端，通过 ssh 连接到服务器实例，并使用以下命令运行代理:

```
./cloud_sql_proxy -instances=adventures-on-gcp:us-central1:myinstance=tcp:5431

2018/08/08 12:56:58 Listening on 127.0.0.1:5431 for adventures-on-gcp:us-central1:myinstance
2018/08/08 12:56:58 Ready for new connections
2018/08/08 12:58:31 New connection for "adventures-on-gcp:us-central1:myinstance"
2018/08/08 12:58:37 Client closed local connection on 127.0.0.1:5431
```

现在，在另一个终端窗口中，您可以执行第一个 scipt(从文件夹 webapp)在数据库中创建表格，我们将在其中上传数据:

```
python sql_search.py
```

和其他命令开始上传过程(从文件夹 cloud_sql)

```
python upload_data.py
```

希望不会有错误，脚本打印每 10000 次插入的时间戳。整个过程花了我大约 15 分钟。

关于数据库的所有工作都完成了，可以删除 GCE 服务器了

```
gcloud compute instances delete instance-1
The following instances will be deleted. Any attached disks configured
 to be auto-deleted will be deleted unless they are attached to any 
other instances or the `--keep-disks` flag is given and specifies them
 for keeping. Deleting a disk is irreversible and any data on the disk
 will be lost.
 - [instance-1] in [us-central1-a]

Do you want to continue (Y/n)?  

Deleted [https://www.googleapis.com/compute/v1/projects/adventures-on-gcp/zones/us-central1-a/instances/instance-1].
```

# Web 服务器部署

Web 服务器基本上与以前的文章相同，简单的 Flask 应用程序，带有一些端点，我将在 Google App Engine 上部署这些端点。关于云 SQL 连接，需要在 GAE 的 app.yaml 文件中设置具体的配置。这就是 app.yaml 的生活。

```
runtime: python
env: flex
entrypoint: gunicorn -b :$PORT main:app

runtime_config:
  python_version: 3

env_variables:
    # Replace user, password, database, and instance connection name with the values obtained
    # when configuring your Cloud SQL instance.
    # postgresql+psycopg2://<username>:<password>@/<database name>?host=/cloudsql/<instance connection name>
    SQLALCHEMY_DATABASE_URI: postgresql+psycopg2://postgres:mypass@/eshop?host=/cloudsql/adventures-on-gcp:us-central1:myinstance

beta_settings:
    # cloud_sql_instances: <instance connection name>
    cloud_sql_instances: adventures-on-gcp:us-central1:myinstance
```

当我在本地(通过代理)和在 GAE(在 sql_search.py 中使用)使用数据库时，我使用环境变量 SQLALCHEMY_DATABASE_URI 来区分连接配置。

beta_settings 是也需要的特定设置，它包含实例连接名称

如果设置了这些变量，则使用以下命令部署应用程序:

```
gcloud app deploy app.yaml --promote
```

这需要一些时间(有时甚至 10 分钟),因为 Docker 映像是创建和部署的。

您可以在网址[http://<project-id>. appspot . com](http://<project-id>.appspot.com)查看申请

![](img/dc0c0075c4ad889cd8b44615d1eb3b81.png)

# 负载测试

因为一切都正常，就像在以前的案例中一样，我使用 Kubernetes 集群上部署的 Locust 框架进行负载测试。更多详情在这里[https://www . the-swamp . info/blog/search-Google-cloud-platform-app-engine-and-search-API/](https://www.the-swamp.info/blog/search-google-cloud-platform-app-engine-and-search-api/)(负载测试下)以及如何设置集群在这里[https://github . com/zde nulo/GCP-search/tree/master/Load-Testing](https://github.com/zdenulo/gcp-search/tree/master/load-testing)。出于测试目的，我部署了已经运行了 10 个实例的 GAE 应用程序(具有自动缩放功能),因为 GAE Flex 在旋转新实例方面比 GAE 标准慢得多，我不希望自动缩放影响结果。

![](img/98813258ae16702f5e62de1d6846e3b2.png)![](img/77d2157ce9f545069ef3870fdccb5bd3.png)![](img/daeefcacf836a07c421685de72ca7308.png)

我以每秒 5 个用户的孵化率运行了 2000 个用户。从图中可以看出，测试持续了大约 10 分钟，没有服务器错误或类似错误，最后我认为它总共运行了 13 个实例。总共有 218816 个请求，平均响应时间为 16 毫秒，平均响应时间为 20 毫秒，这是迄今为止最快的解决方案。值得注意的是，数据库服务器的 CPU 负载峰值在 30%左右。

![](img/be3a1960ed3f4275a4b073e861814f76.png)

# 结论

由于 PostreSQL 经过了实战测试，是最受欢迎的开源 SQL 数据库之一，并且提供了许多额外的特性，因此它可以成为用于 eshops 数据库存储或类似用途的绝佳选择。在此基础上的云 SQL 提供了可管理的功能，如轻松扩展服务器实例的可能性、自动备份和高可用性。