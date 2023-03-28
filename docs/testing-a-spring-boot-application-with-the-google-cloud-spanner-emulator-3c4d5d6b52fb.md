# 使用 Google Cloud Spanner 模拟器测试 Spring Boot 应用程序

> 原文：<https://medium.com/google-cloud/testing-a-spring-boot-application-with-the-google-cloud-spanner-emulator-3c4d5d6b52fb?source=collection_archive---------2----------------------->

Cloud Spanner 已经出现了一段时间，但是根据真实实例测试您的应用程序可能会很困难:

*   使用真实实例进行测试是有成本的——“不要测试太多，这是在浪费我们的钱！”
*   你必须在线，并且有像样的连接——如果你在通勤时工作，这可不太好

最近，谷歌发布了云扳手模拟器。仿真器可以在您自己的机器上运行，并且与真实的扳手实例兼容——它可以运行与真实的扳手完全相同的 SQL 语句。

在本指南中，我们将介绍如何设置模拟器，使其作为小型 Spring Boot Java 应用程序的一部分运行。

# 使用模拟器

Cloud Spanner 模拟器可以在 Linux 下作为正常进程运行，也可以在其他平台上与 Docker 一起运行。我们将在本指南中使用 docker 映像，因为它简单且独立，而且我喜欢我的构建在任何地方运行。

模拟器项目在 Github 上的[这里。这里有关于如何在 Docker 下工作的说明，但是让我们把所有这些逻辑放在我们的构建中，这样其他开发人员(和 CI 服务器)就不必担心它了。](https://github.com/GoogleCloudPlatform/cloud-spanner-emulator)

# 一个简单的 Spring Boot 应用

我们的应用程序将非常简单，使用 Spring 数据进行数据访问。用于演示的几个实体:

和它们的储存库:

# 测试配置

对于集成测试，需要进行一些设置。这个 Spring 配置将用于设置测试环境:

我们将使用不同的扳手配置进行测试，从系统属性中读取项目、实例和数据库 id，如果没有指定，则使用默认值。这将允许用户点击一个真正的扳手实例，如果他们想要手动运行测试，或者在特殊的 CI 构建上。也许我们希望将模拟器用于日常 CI，但是对于发布前的构建，我们将使用真正的 Spanner 实例？

环境变量 SPANNER_EMULATOR_HOST 将使用 SPANNER 模拟器的位置进行配置。如果未指定，将尝试使用真实的扳手实例。

当 Spanner 模拟器启动时，有一个没有配置实例的项目可用。此代码:

查找仿真程序中唯一可用的实例配置，并查找唯一可用的项目。根据这些信息，可以创建一个新的实例，然后创建一个新的数据库。创建数据库 ID 提供者 bean 时(应用程序只创建一次)，这是设置所有这些的好时机。无论如何，在模拟器中这样做是很便宜的，所以即使你决定对每个测试方法都这样做，也不会增加太多的开销。

# 集成测试

现在我们已经设置好了所有这些，我们可以编写一个测试:

我们将使用存储库类删除所有数据，并在每次测试之间重新创建数据，因此我们总是在已知的状态下进行测试。然后测试几个库，只是为了证明模拟器在工作。

# 用一个 Maven 项目将所有东西连接在一起

我们将从一个非常简单的带有扳手相关性的 Spring Boot 项目开始:

到目前为止，我们还没有运行集成测试的东西。接下来，我们将添加一个 [Docker 插件](http://dmp.fabric8.io)，它将启动一个 Docker 容器来运行扳手模拟器:

第一部分是指定 Docker 图像。Docker 图像的名称是 gcr.io/cloud-spanner-emulator/emulator:1.0.0，我们从[扳手模拟器文档](https://cloud.google.com/spanner/docs/emulator)中获得。因为我们想要一个一致的版本，我们将选择一个具体的版本号，1.0.0。可用版本列表可以通过使用 gcloud 命令行工具列出标签来确定:

```
gcloud container images list-tags 
gcr.io/cloud-spanner-emulator/emulator
```

模拟器公开了一个我们将使用的 TCP 端口。配置:

```
<**port**>+spanner.emulator.host:spanner.emulator.port:9010</**port**>
```

会将容器中的端口 9010 映射到主机上随机选择的空闲端口，还会保存一些包含主机端口和 Docker 主机本身的系统属性。通常 docker 主机是 localhost，但现在总是——这取决于您的平台——例如，使用 docker-machine 启动 Docker 的用户将拥有一个对应于虚拟机的主机。

接下来，我们指示 Docker 插件在 TCP 端口 9010 响应之前不要继续构建——我们不想在模拟器完全启动之前就开始测试。

剩下的部分是使用故障保护插件配置集成测试本身的执行:

SPANNER_EMULATOR_HOST 环境变量正在使用由 Docker 启动的模拟器的主机和端口进行配置。剩下的就是标准的故障安全集成测试配置。

运行 mvn verify，Spanner 模拟器将在 Docker 中启动，然后测试将运行，最后模拟器容器被停止并再次移除。第一次运行时，您需要网络连接来下载依赖项和 Spanner Docker 映像，但之后，即使您没有任何网络/互联网访问，也可以运行构建。

# 使用真实的扳手实例

如果需要对一个真实的扳手实例进行测试，我们可以稍微改变一下项目来实现这一点。

让我们定义 3 个可能从命令行配置的构建属性:test.projectId、test.instanceId 和 test.databaseId。之前编写的 JUnit 测试代码已经包含了这些属性，所以让我们通过重新定义故障保护插件配置，在构建文件中贯穿这些属性:

但是模拟器呢？让我们将所有的仿真器代码移到一个 Maven 概要文件中:

这里我们有一个概要文件，它将总是被激活，除非 test.projectId 被配置，所以它将像以前一样工作。只有在此配置文件处于活动状态时，才会在故障保护中设置 SPANNER_EMULATOR_HOST 环境变量。

所以现在，在没有设置任何属性的情况下，模拟器将运行，构建将使用模拟器运行测试。但是，如果需要运行真实的 Spanner 实例，请设置这三个属性:

```
mvn verify -Dtest.projectId=helix-sydney -Dtest.instanceId=myinstance -Dtest.databaseId=cats
```

在运行之前，您需要[为您的实例配置凭证](https://cloud.google.com/sdk/gcloud/reference/auth/login)。