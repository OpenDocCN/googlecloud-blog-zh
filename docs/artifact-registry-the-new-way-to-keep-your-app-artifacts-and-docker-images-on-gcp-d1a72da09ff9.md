# 工件注册:在 GCP 上保存你的应用工件和 Docker 图片的新方法

> 原文：<https://medium.com/google-cloud/artifact-registry-the-new-way-to-keep-your-app-artifacts-and-docker-images-on-gcp-d1a72da09ff9?source=collection_archive---------0----------------------->

![](img/d3a4d1d7a3b31a5bae6dd126017bef36.png)

2020 年 11 月更新

[神器注册表现在 GA](https://cloud.google.com/blog/products/devops-sre/artifact-registry-is-ga) ！

* 2021 年 10 月更新:

[语言包的工件注册表现已正式发布](https://cloud.google.com/blog/products/application-development/node-python-and-javarepos-are-generally-available)

— :: —

今天，当你需要在 GCP 上存储你的 Docker 图片时，你可以使用 GCR(谷歌容器注册)。这非常方便，因为您将在 GCP 生态系统中使用完全集成的服务，这已经很容易由 GCP IAM 管理。

通常，Doker 图像并不是您需要保存的唯一工件，使用 Jfrog Artifactory 或 Sonatype Nexus 这样的应用程序来管理 jar 文件、npm 等工件是很常见的。你也可以使用 GCS 来维护那些工件，正如这里显示的。

谷歌最近推出了**工件注册表**(̵c̵u̵r̵r̵e̵n̵t̵l̵y̵̵i̵n̵̵b̵e̵t̵a̵)̵，2020 年 11 月正式发布)，使你能够集中存储工件并建立依赖关系，作为集成的谷歌云体验的一部分。

AR 是 Container Registry 的发展，当它成为 GA 时，它将取代 GCR，所以了解一下它的特性会很有好处。

Artifact Registry 为管理包和 Docker 容器映像提供了一个单一的位置。您可以:

*   管理整个工件生命周期—存储、管理和保护您的工件。
*   集成到您的环境中—使用您现有的 CI/CD 工具来上传和下载工件。利用与云身份和访问管理以及 Google 云产品的紧密集成，或者将其他工具与 Artifact Registry 集成。
*   在单个 Google Cloud 项目中创建多个区域性存储库。
*   在项目或存储库级别控制访问。
*   VPC 服务中的安全库控制着安全边界。

Artifact Registry 与 Cloud Build 和其他持续交付和持续集成系统集成，以存储来自您的构建的包。您还可以存储用于生成和部署的可信依赖项。

下面我将向您展示如何为 Docker 容器和 Java 工件创建 AR。

# 码头工人

第一步:

向工件注册表验证:

```
gcloud beta auth configure-docker us-central1-docker.pkg.dev
```

第二步:

Docker 需要访问工件注册表来推和拉图像。您可以使用独立的 Docker 凭证助手工具`docker-credential-gcr`来配置您的工件注册表凭证，以便与 Docker 一起使用，而不需要使用`gcloud`。

```
VERSION=2.0.0
OS=linux  # or "darwin" for OSX, "windows" for Windows.
ARCH=amd64  # or "386" for 32-bit OSs

curl -fsSL "https://github.com/GoogleCloudPlatform/docker-credential-gcr/releases/download/v${VERSION}/docker-credential-gcr_${OS}_${ARCH}-${VERSION}.tar.gz" \
  | tar xz --to-stdout ./docker-credential-gcr \
  > /usr/bin/docker-credential-gcr && chmod +x /usr/bin/docker-credential-gcr
```

您可以在此找到关于如何安装此工具[的更多信息](https://github.com/GoogleCloudPlatform/docker-credential-gcr)

第三步:

将 Docker 配置为在与工件注册中心交互时使用您的工件注册中心凭证

```
docker-credential-gcr configure-docker --registries=us-central1-docker.pkg.dev 
```

现在你应该可以走了，用你的`docker push` & `docker tag` & `docker pull`命令；)

# Java 语言(一种计算机语言，尤用于创建网站)

下面的步骤将向您展示如何创建 maven repo 并推送您的工件。要知道 maven 还在 ALPHA，随时可以换。

第一步:

授予 Cloud SDK 访问 Google Cloud 的权限

```
gcloud auth application-default login
```

第二步:

创建 maven 存储库

```
gcloud beta artifacts repositories create quickstart-maven-repo --repository-format=maven \
--location=us-central1 [--description="Maven repository"]
```

对于 docker 容器，您只需更改`--repository-format=docker`

您可以使用验证它是否是正确创建的

```
gcloud beta artifacts repositories list
```

要简化 gcloud 命令，您可以使用以下命令设置默认 repo 和位置:

```
gcloud config set artifacts/repository quickstart-maven-repo
```

和

```
gcloud config set artifacts/location us-central1
```

第三步:

将配置添加到 pom.xml 中

```
<distributionManagement>
  <snapshotRepository>
    <id>artifact-registry</id>
    <url>artifactregistry://us-central1-maven.pkg.dev/**PROJECT**/quickstart-maven-repo</url>
  </snapshotRepository>
  <repository>
    <id>artifact-registry</id>
    <url>artifactregistry://us-central1-maven.pkg.dev/**PROJECT**/quickstart-maven-repo</url>
  </repository>
</distributionManagement>

<repositories>
  <repository>
    <id>artifact-registry</id>
    <url>artifactregistry://us-central1-maven.pkg.dev/**PROJECT**/quickstart-maven-repo</url>
    <releases>
      <enabled>true</enabled>
    </releases>
    <snapshots>
      <enabled>true</enabled>
    </snapshots>
  </repository>
</repositories>

<build>
  <extensions>
    <extension>
      <groupId>com.google.cloud.artifactregistry</groupId>
      <artifactId>artifactregistry-maven-wagon</artifactId>
      <version>2.0.1</version>
    </extension>
  </extensions>
</build>
```

您可以使用以下命令轻松生成此配置

```
gcloud beta artifacts print-settings mvn
```

第四步:

现在，您将能够使用`mvn deploy`和`mvn release`命令来推动工件

就是这样！

## 摘要

在本文中，您将看到如何使用新的 GCP 工件注册中心轻松地集成和管理您的工件，因此您将不需要任何其他外部工件管理器工具。

GCP AR 也将在未来取代我们目前的 GCR，所以即使在这个早期阶段的 Beta/Alpha，也要密切关注它。

我希望你喜欢它！

请让我知道你的想法！

来源:

[https://cloud.google.com/artifact-registry/docs/overview](https://cloud.google.com/artifact-registry/docs/overview)

[https://cloud . Google . com/artifact-registry/docs/Java/quick start](https://cloud.google.com/artifact-registry/docs/java/quickstart)