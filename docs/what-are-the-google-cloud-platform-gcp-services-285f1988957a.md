# 什么是谷歌云平台(GCP)服务？

> 原文：<https://medium.com/google-cloud/what-are-the-google-cloud-platform-gcp-services-285f1988957a?source=collection_archive---------0----------------------->

## 描述时没有……营销的帮助

> *参见* : [谷歌的云平台是什么(用 1000 个最常用的词描述)？](/@retomeier/how-the-big-search-company-lets-you-use-their-computers-to-do-your-own-stuff-b393a3aa0cc3#.x7y23ngyj)

[谷歌云平台](https://cloud.google.com/products/) (GCP)提供数十种 [IaaS](https://en.wikipedia.org/wiki/Cloud_computing#Infrastructure_as_a_service_.28IaaS.29) 、 [PaaS](https://en.wikipedia.org/wiki/Cloud_computing#Platform_as_a_service_.28PaaS.29) 、 [SaaS](https://en.wikipedia.org/wiki/Cloud_computing#Software_as_a_service_.28SaaS.29) 服务。可悲的是，维基百科上关于 GCP 的条目是垃圾，虽然官方文件很好，但洒在上面的营销灰尘让我牙痛。作为我自己的参考，我对 GCP 上的每一项服务都做了客观的描述。

向 [Greg Wilson](https://medium.com/u/f1669de10ebd?source=post_page-----285f1988957a--------------------------------) 、Google Cloud 开发者关系团队和 Google Cloud 开发者专家致敬，他们给出了四个字的描述。

## **计算**

*   [**计算引擎**](https://cloud.google.com/compute/)*虚拟机、磁盘和网络*一种 [IaaS](https://en.wikipedia.org/wiki/Cloud_computing#Infrastructure_as_a_service_.28IaaS.29) 服务，提供托管在谷歌基础设施上的[虚拟机](https://en.wikipedia.org/wiki/Virtual_machine) (VMs)。竞争对手的服务包括[亚马逊弹性计算云](https://en.wikipedia.org/wiki/Amazon_Elastic_Compute_Cloud)，以及 [OpenStack](https://en.wikipedia.org/wiki/OpenStack) 等本地对等服务。
*   [**App Engine**](https://cloud.google.com/appengine/)【*托管应用平台*】一种 [PaaS](https://en.wikipedia.org/wiki/Cloud_computing#Platform_as_a_service_.28PaaS.29) 服务，用于使用容器实例构建 web 应用和移动后端，这些容器实例预先配置有[几个可用运行时](https://cloud.google.com/appengine/docs/about-the-standard-environment)中的一个，每个运行时都包括一组标准的 App Engine 库。竞争对手的服务包括亚马逊弹性豆茎和微软 Azure 网站。
*   [**容器引擎**](https://cloud.google.com/container-engine/)*托管库/容器*[码头工人](https://www.docker.com/)容器的集群管理和编排系统。它基于开源的 Kubernetes 项目。
*   [**容器注册表**](https://cloud.google.com/container-registry)*私有容器注册表&存储*私有 [Docker](https://en.wikipedia.org/wiki/Docker_(software)) 存储库托管在谷歌的基础设施上。
*   [**云功能**](https://cloud.google.com/functions/) [ *无服务器微服务* ]一种基于事件的异步计算解决方案，允许您创建[微服务](https://en.wikipedia.org/wiki/Microservices)(小型专用功能)，这些微服务可以响应云事件，而无需明确管理的服务器或运行时环境。谷歌云功能自 2016 年 2 月开始进入 Alpha [。](http://venturebeat.com/2016/02/09/google-has-quietly-launched-its-answer-to-aws-lambda/)
*   [**Cloud Pub/Sub**](https://cloud.google.com/pubsub)【*分布式实时消息*】一种完全托管的实时消息服务，用于在独立的应用程序之间发送和接收消息。
*   用于应用引擎*云 API 网关*的 [**云端点框架，用于从您的代码中创建**](https://cloud.google.com/appengine/docs/java/endpoints/) **[RESTful](https://en.wikipedia.org/wiki/Representational_state_transfer) 服务，并使其可供使用[应用引擎](https://en.wikipedia.org/wiki/Google_App_Engine)的 [iOS](https://en.wikipedia.org/wiki/IOS) 、 [Android](https://en.wikipedia.org/wiki/Android_(operating_system)) 和 [Javascript](https://en.wikipedia.org/wiki/JavaScript) 客户端访问。原*云端点*。**

## **存储和数据库**

*   [**云存储**](https://cloud.google.com/storage/)**对象&文件存储和服务**统一的对象存储服务，提供[系列存储选项](https://cloud.google.com/storage/docs/storage-classes)，包括地理冗余(低延迟、高 [QPS](https://en.wikipedia.org/wiki/Queries_per_second) 内容服务于跨地理区域的用户)、区域性(针对特定区域的工作负载)、[近线](https://cloud.google.com/storage-nearline/nearline-whitepaper)(针对每月访问不到一次的数据)，以及)竞争对手的服务包括[亚马逊简单存储服务](https://en.wikipedia.org/wiki/Amazon_S3)(地理冗余/区域)和[亚马逊冰川](https://en.wikipedia.org/wiki/Amazon_Glacier) (coldline)。**
*   **[**云 SQL**](https://cloud.google.com/sql)【*托管 MySQL* 】一种完全托管的 [MySQL](https://en.wikipedia.org/wiki/MySQL) 数据库服务，用于在谷歌的基础设施上托管关系型 MySQL 数据库。**
*   **[**Bigtable**](https://cloud.google.com/bigtable/)[*h base Compatible NoSQL*]高性能 [NoSQL](https://en.wikipedia.org/wiki/NoSQL) [大数据](https://en.wikipedia.org/wiki/Big_data)数据库服务，旨在以一致的低延迟和高吞吐率支持超大型工作负载。Google [在内部使用 Bigtable](http://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)来支持包括搜索和 Gmail 在内的服务。**
*   **[**云数据存储**](https://cloud.google.com/datastore/)*分布式分层键/值存储*[NoSQL](https://en.wikipedia.org/wiki/NoSQL)无模式数据库，用于存储非关系数据。当需要 [ACID](https://en.wikipedia.org/wiki/ACID) 事务或者存储的数据高度结构化时，它是 [Bigtable](https://cloud.google.com/bigtable/) 的替代方案。**
*   **[**云扳手**](https://cloud.google.com/spanner/) 一个托管的全球分布式关系数据库，具有 ACID 事务、强一致性、SQL 语义、水平伸缩、高可用性。**
*   **[**持久磁盘**](https://cloud.google.com/persistent-disk/) [ *可连接虚拟机的磁盘* ]一种提供 SSD 和 HDD 存储的服务，可以连接到在[计算引擎](https://cloud.google.com/compute/)或[容器引擎](https://cloud.google.com/container-engine/)中运行的实例。**
*   **[**云源库**](https://cloud.google.com/source-repositories/)*托管私有 Git 库*[托管在 GCP 的私有 Git](https://en.wikipedia.org/wiki/Git) 库；他们目前正处于测试阶段。**

## ****大数据****

*   **[**Big query**](http://cloud.google.com/bigquery)*无服务器数据仓储&分析*S[无服务器](https://en.wikipedia.org/wiki/Serverless_computing)，全托管，Pb 级[数据仓库](https://en.wikipedia.org/wiki/Data_warehouse)和[分析](https://en.wikipedia.org/wiki/Analytics)平台，用于存储和查询[大数据](https://en.wikipedia.org/wiki/Big_data)使用 [SQL](https://en.wikipedia.org/wiki/SQL) 。**
*   **[**云数据流**](https://cloud.google.com/dataflow/) [ *托管数据处理* ]全托管实时数据处理服务，用于批量和流式[大数据](https://en.wikipedia.org/wiki/Big_data)处理，支持 [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load) 、批量计算、连续计算。**
*   **[**data proc**](https://cloud.google.com/dataproc/)*Managed Spark&Hadoop***Managed[Apache Hadoop](https://en.wikipedia.org/wiki/Apache_Hadoop)， [Apache Spark](https://en.wikipedia.org/wiki/Apache_Spark) ， [Apache Pig](https://en.wikipedia.org/wiki/Pig_(programming_tool)) ，以及 [Apache Hive](https://en.wikipedia.org/wiki/Apache_Hive) 服务用于处理大型数据集。****
*   ****[**云数据实验室**](https://cloud.google.com/datalab/)【*数据可视化与探索*】一款基于 [Jupyter](https://en.wikipedia.org/wiki/IPython#Project_Jupyter) (前身为 IPython)构建的大规模数据探索、分析、可视化的交互工具。它支持使用 [BigQuery](http://cloud.google.com/bigquery) 、[计算引擎](https://cloud.google.com/compute/)、使用 [Python](https://en.wikipedia.org/wiki/Python_(programming_language)) 、 [SQL](https://en.wikipedia.org/wiki/SQL) 、 [JavaScript](https://en.wikipedia.org/wiki/JavaScript) 的[云存储](https://cloud.google.com/storage)。云数据实验室正在[公测](https://techcrunch.com/2015/10/13/google-launches-cloud-datalab-an-interactive-tool-for-exploring-and-visualizing-data/)。****
*   ****[**Google Genomics**](https://cloud.google.com/genomics/overview)[*Managed Genomics Platform*]使用[全球基因组与健康联盟](https://www.genomeweb.com/informatics/google-joins-global-alliance-genomics-and-health)定义的标准存储、处理、探索和共享[基因组](https://en.wikipedia.org/wiki/Genomics)数据的 API。这包括对管理数据集、读数和变量的支持；搜索和切片；以及为共享设置访问控制。****

## ******机器学习******

*   ****[**云机器学习**](https://cloud.google.com/products/machine-learning/) [ *带 TensorFlow 的机器学习* ]一个托管服务，用于使用 [TensorFlow](https://www.tensorflow.org/) 框架构建[机器学习](https://en.wikipedia.org/wiki/Machine_learning)模型。****
*   ****[**云视觉 API**](https://cloud.google.com/vision/)【*图像识别和分类*】一个 [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) API，可以用来将图像的内容理解成类别，检测图像中的单个对象和人脸，并查找和读取图像中包含的印刷文字 [](https://techcrunch.com/2016/02/18/google-opens-its-cloud-vision-api-to-all-developers/) 。****
*   ****[**云语音 API**](https://cloud.google.com/speech/)*将语音转换为文本*[REST](https://en.wikipedia.org/wiki/Representational_state_transfer)API，可用于将音频转换为文本。API 可以识别超过 80 种语言和变体。谷歌云语音 API[目前正在公测](https://techcrunch.com/2016/03/23/google-opens-access-to-its-speech-recognition-api-going-head-to-head-with-nuance/)。****
*   ****[**自然语言 API**](https://cloud.google.com/natural-language/)【*文本解析和分析*】一个 [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) API，可以用来解析文本的结构和含义。它可以在提供的文本中提取包括人物、地点、事件和情感在内的信息。谷歌云自然语言 API[目前正在公测](https://techcrunch.com/2016/07/20/google-launches-new-api-to-help-you-parse-natural-language/)。****
*   ****[**翻译 API**](https://cloud.google.com/translate/) [ *语言检测和翻译* ]一个 [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) API，可用于将任意语言字符串翻译成任何支持的语言。[语言识别](https://cloud.google.com/translate/docs/detecting-language)可用于未知源语言的情况。****

## ******联网******

*   ****[**谷歌云虚拟网络**](https://cloud.google.com/compute/docs/networking)【*软件定义网络*】一套由谷歌管理的网络功能，包括粒度 IP 地址范围选择、[路由](https://en.wikipedia.org/wiki/Routing_table)、[防火墙](https://en.wikipedia.org/wiki/Firewall_(computing))、[虚拟专用网](https://en.wikipedia.org/wiki/Virtual_private_network) (VPN)和云路由器，用于配置您的 GCP 资源，在[虚拟私有云](https://gigaom.com/2014/11/04/google-cloud-goes-corporate-with-peering-carrier-interconnects-vpn/) (VPC)中将它们彼此连接和隔离。****
*   ****[**云负载均衡**](https://cloud.google.com/compute/docs/load-balancing-and-autoscaling)*多区域负载分配*[负载均衡](https://en.wikipedia.org/wiki/Load_balancing_(computing))和[自动扩展](https://en.wikipedia.org/wiki/Autoscaling) GCP 计算资源在单个 [anycast](https://en.wikipedia.org/wiki/Anycast) IP [](http://research.google.com/pubs/pub44824.html)之后的单个或多个区域中的服务。****
*   ****[**云 CDN**](https://cloud.google.com/cdn/) [ *内容交付网络* ]利用 Google 全球分布的 edge [存在点](https://peering.google.com/#/infrastructure)缓存 HTTP(S) [负载均衡](https://en.wikipedia.org/wiki/Load_balancing_(computing))内容靠近用户。****
*   ****[**谷歌云互联**](https://cloud.google.com/interconnect/docs)*与* *GCP* 对等允许 GCP 客户通过比他们现有的互联网连接更高可用性和/或更低延迟的连接来连接到谷歌。****
*   ****[**云 DNS**](https://cloud.google.com/dns/docs/)【*可编程域名服务*】一种托管的权威[域名系统](https://en.wikipedia.org/wiki/Domain_Name_System) (DNS)服务，运行在与 Google 相同的基础设施上。云 DNS 将对[域名](https://en.wikipedia.org/wiki/Domain_name)的请求翻译成 [IP 地址](https://en.wikipedia.org/wiki/IP_address)，并提供 UI、命令行界面和 API，用于发布和管理数百万个 [DNS 区域](https://en.wikipedia.org/wiki/DNS_zone)和[资源记录](https://en.wikipedia.org/wiki/Domain_Name_System#DNS_resource_records)。****

## ******身份和安全******

*   ****[**Google Cloud IAM**](https://cloud.google.com/iam/)[*身份&访问管理* ]让[管理员](https://en.wikipedia.org/wiki/System_administrator)授权谁可以对特定资源采取行动，以及内置的[审计](https://en.wikipedia.org/wiki/Information_technology_audit) [](http://venturebeat.com/2016/03/23/google-cloud-platform-now-offers-identity-and-access-management-roles-for-users/)。****
*   ****[**云资源管理器**](https://cloud.google.com/resource-manager/)【*云项目元数据管理*】一种以编程方式管理资源容器(如组织和项目)的服务，用于分组和分层组织 GCP 资源 [⁴](http://venturebeat.com/2015/08/06/google-launches-cloud-deployment-manager-out-of-beta/) 。****
*   ****[**云安全扫描器**](https://cloud.google.com/security-scanner/)*App Engine 安全扫描器*[web 安全扫描器](https://en.wikipedia.org/wiki/Web_application_security_scanner)针对 [App Engine](http://cloud.google.com/appengine) 应用中的[常见漏洞](https://techcrunch.com/2015/02/19/google-launches-security-scanner-to-help-find-vulnerabilities-in-app-engine-sites/)，包括[跨站点脚本](https://en.wikipedia.org/wiki/Cross-site_scripting) (XSS)、Flash 注入、混合内容(HTTPS HTTP)、过时/不安全的库。****

## ******谷歌云平台管理服务******

*   ****[**stack driver**](http://thenewstack.io/closer-look-google-stackdriver/)[*云监控、日志记录&诊断*为构建在云基础设施上的应用提供监控、日志记录和诊断，包括 GCP 和 AWS。Stackdriver 提供了度量、仪表板、警报、日志管理、报告和[跟踪](https://en.wikipedia.org/wiki/Tracing_(software))功能。****
*   ****[**部署管理器**](https://cloud.google.com/deployment-manager/) [ *基于模板的基础设施部署* ]一种基础设施自动化和管理服务，允许您定义模板来部署各种 GCP 服务，包括[云存储](http://cloud.google.com/storage)、[计算引擎](http://cloud.google.com/compute)和[云 SQL](http://cloud.google.com/sql) 。 [⁵](http://venturebeat.com/2015/08/06/google-launches-cloud-deployment-manager-out-of-beta/)****
*   ****[**云壳**](https://cloud.google.com/shell/)***基于浏览器的终端/CLI*[命令行](https://en.wikipedia.org/wiki/Command-line_interface)从[浏览器](https://en.wikipedia.org/wiki/Web_browser)内访问云资源，无需在您的系统上安装谷歌云 SDK 或其他工具 [⁶](http://www.programmableweb.com/news/google-cloud-platform-updates-include-cloud-shell/2015/10/16) 。******
*   ******[**Google cloud billing API**](https://cloud.google.com/billing/)*programatic GCP billing management*以编程方式为您的 GCP 项目管理账单 [⁷](https://techcrunch.com/2013/12/23/googles-cloud-platform-gets-billing-api-to-help-developers-monitor-and-analyze-cost/) 。******

> ******有什么遗漏或错误吗？补充评论！******

## ******进一步阅读******

*   ******[谷歌的云平台是什么？](/@retomeier/what-is-the-google-cloud-platform-d92a9c9e5e89)******
*   ******或者大型搜索公司如何让你用他们的电脑做你自己的事情******
*   ******[谷歌云平台的注释历史](/@retomeier/an-annotated-history-of-the-google-cloud-platform-90b90f948920)******