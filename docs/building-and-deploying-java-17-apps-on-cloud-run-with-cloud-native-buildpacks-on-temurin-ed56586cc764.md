# 在云上构建和部署 Java 17 应用使用 Temurin 上的云原生构建包运行

> 原文：<https://medium.com/google-cloud/building-and-deploying-java-17-apps-on-cloud-run-with-cloud-native-buildpacks-on-temurin-ed56586cc764?source=collection_archive---------1----------------------->

在本文中，让我们重温一下在[云运行](https://cloud.run/)上部署 Java 应用的主题。特别是我会部署一个 [Micronaut](https://micronaut.io/) app，用 [Java 17](https://jdk.java.net/17/) (Adoptium 的 Temurin)编写，用 [Gradle](https://gradle.org/) 构建。

**带自定义 Dockerfile 文件**

在 Cloud Run 上，您部署容器化的应用程序，因此您必须决定您想要为您的应用程序构建容器的方式。在[以前的文章](https://glaforge.appspot.com/article/start-the-fun-with-java-14-and-micronaut-inside-serverless-containers-on-cloud-run)中，我展示了一个使用您自己的 Dockerfile 的例子，对于 OpenJDK 17 看起来如下，并启用了该语言的预览功能:

```
FROM openjdk:17
WORKDIR /app
COPY ./ ./
RUN ./gradlew shadowJar
EXPOSE 8080
CMD ["java", " - enable-preview", "-jar", "build/libs/app-0.1-all.jar"]
```

为了进一步改进 Docker 文件，您可以使用多级 Docker 构建器，首先用 Gradle 在一个步骤中构建应用程序，然后在第二个步骤中运行它。此外，由于 JAR 文件名是硬编码的，您可能希望将命令参数化。

要构建映像，您可以使用 Docker 在本地构建它，然后将其推送到容器注册表，然后部署它:

```
# gcloud auth configure-docker
# gcloud components install docker-credential-gcrdocker build . - tag gcr.io/YOUR_PROJECT_ID/IMAGE_NAME
docker push gcr.io/YOUR_PROJECT_ID/IMAGE_NAMEgcloud run deploy myservice \
       -image gcr.io/YOUR_PROJECT_ID/IMAGE_NAME
```

除了使用 Docker 在本地构建，您还可以让[云构建](https://cloud.google.com/build)为您完成:

```
gcloud builds submit . - tag gcr.io/YOUR_PROJECT_ID/SERVICE_NAME
```

**带起重臂**

除了使用 Dockerfiles，您还可以让 [JIB](https://github.com/GoogleContainerTools/jib) 为您创建容器，就像我在[另一篇文章](https://glaforge.appspot.com/article/running-micronaut-serverlessly-on-google-cloud-platform)中写的那样。您配置 Gradle 使用 JIB 插件:

```
plugins {
    …
    id “com.google.cloud.tools.jib” version “2.8.0”
}
…
tasks {
    jib {
       from {
          image = “gcr.io/distroless/java17-debian11”
       }
       to {
          image = “gcr.io/YOUR_PROJECT_ID/SERVICE_NAME”
       }
    }
}
```

您指定了插件的版本，但是您也通过选择具有相同版本的基础映像来表明您想要使用 Java 17。请确保更改项目 ID 和服务名称的占位符。请随意查阅关于 JIB Gradle 插件的[文档](https://github.com/GoogleContainerTools/jib/tree/master/jib-gradle-plugin)。然后，您可以让 Gradle 用。/gradlew jib，或与。/gradlew jibDockerBuild，如果您希望使用本地 Docker 守护程序。

**采用云原生构建包**

现在我们已经讨论了其他方法，让我们来关注使用[云本地构建包](https://buildpacks.io/)来代替，特别是[谷歌云本地构建包](https://github.com/GoogleCloudPlatform/buildpacks)。有了 buildpacks，您就不必在部署服务之前为 docker 文件或构建容器而烦恼了。您让 Cloud Run 使用 buildpacks 从源代码构建、封装和部署您的应用程序。

开箱即用，buildpack 实际上是针对 Java 8 或 Java 11 的。但我对用 Java 17 运行 Java 的最新 LTS 版本感兴趣，以便利用一些预览功能，如记录、密封类、开关表达式等。

在我的 Gradle build 中，我指定我使用的是 Java 17，但也启用了预览功能:

```
java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(17))
    }
}
```

就像塞德里克·尚皮乌斯在[的博客文章](https://melix.github.io/blog/2020/06/java-feature-previews-gradle.html)中所说的那样，要启用预览功能，你还应该告诉格雷尔你想让它们执行编译、测试和执行任务:

```
tasks.withType(JavaCompile).configureEach {
    options.compilerArgs.add(“--enable-preview”)
}tasks.withType(Test).configureEach {
    useJUnitPlatform()
    jvmArgs(“--enable-preview”)
}tasks.withType(JavaExec).configureEach {
    jvmArgs(“ — enable-preview”)
}
```

到目前为止还不错，但是正如我所说的，默认的本机 buildpack 没有使用 Java 17，我想指定我使用预览功能。因此，当我试图使用 buildpack 从源代码中部署我的云运行应用程序时，只需运行 gcloud deploy 命令，我就会得到一个错误。

```
gcloud beta run deploy SERVICE_NAME
```

为了解决这个问题，我必须添加一个配置文件，以指示 buildpack 使用 Java 17。我在项目的根目录下创建了一个 project.toml 文件:

```
[[build.env]]
name = “GOOGLE_RUNTIME_VERSION”
value = “17”
[[build.env]]
name = “GOOGLE_ENTRYPOINT”
value = “java — enable-preview -jar /workspace/build/libs/app-0.1-all.jar”
```

我指定运行时版本必须使用 Java 17。但是我还添加了— enable-preview 标志，以便在运行时启用预览功能。

**采用终端打开 JDK 17**

锦上添花的是，这个版本使用的是 OpenJDK 17 的 Adoptium 版本，正如我们最近宣布的！如果您查看云构建中的构建日志，您应该会看到一些提到它的输出，比如:

```
{
    "link": "[https://github.com/adoptium/temurin17-binaries/releases/download/jdk-17.0.4.1%2B1/OpenJDK17U-jdk-sources_17.0.4.1_1.tar.gz](https://github.com/adoptium/temurin17-binaries/releases/download/jdk-17.0.4.1%2B1/OpenJDK17U-jdk-sources_17.0.4.1_1.tar.gz)",
    "name": "OpenJDK17U-jdk-sources_17.0.4.1_1.tar.gz",
    "size": 105784017
}
```

好样的。Java 17 Micronaut app，部署在云 Run 上的 Temurin 多亏了云原生 buildpacks！我赢了时髦的宾果游戏🙂