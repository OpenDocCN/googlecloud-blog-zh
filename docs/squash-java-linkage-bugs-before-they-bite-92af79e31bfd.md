# 在 Java 链接错误咬人之前消灭它们

> 原文：<https://medium.com/google-cloud/squash-java-linkage-bugs-before-they-bite-92af79e31bfd?source=collection_archive---------2----------------------->

一个典型的现代 Java 项目依赖于许多库，每个库又依赖于多个库。有些依赖关系在完全传递依赖图中出现多次，有些只出现一次。有时重复库的版本是相同的。通常它们并不一致，当它们不一致时，程序可能会失败。

![](img/8b9977f498a7ffb1f568781064ab333a.png)

例如，`**org.apache.beam:beam-sdks-java-io-cassandra**`可能调用 Guava 20 中的方法，而`**com.google.cloud:google-cloud-firestore**`调用 Guava 28 中该方法的重命名版本。声明两者的依赖关系会引入隐藏的不兼容性，因为类路径只能包含一个版本的 Guava。即使`**com.google.cloud:google-cloud-firestore**`根本不调用重命名的方法，它仍然可以通过换入 Guava 28 而不是 20(Cassandra 需要的版本)意外地破坏 Cassandra。这种库之间的不兼容被称为*链接错误*，因为其中一个库在运行时链接到了不兼容的 Guava 版本。像这样的问题通常表现为运行时异常，比如`**NoSuchMethodError**`、`**NoClassDefFoundError**`、`**IllegalAccessError**`等等。

Maven Enforcer 插件已经提供了几种策略来减少链接错误，包括要求上限和依赖性收敛。上限检查对于修复在较新版本的库中添加了方法的情况很有用，但当问题是方法已被移除时，上限检查会失败。依赖收敛是避免链接错误的黄金标准，但是在由来自许多独立团队的许多库组成的复杂项目中很难实现。

链接检查器 Enforcer 规则是 Maven Enforcer 插件可以应用于项目的新规则。当完全的依赖收敛是不可能的时候，这是特别有用的。此规则检查项目的整个类路径，并报告所有链接错误。例如，在下面的项目中，gRPC 中的`**JettyNpnSslEngine**`类应该链接到 Jetty 中的`**NextProtoNego**`类，但后者实际上并不在类路径中:

```
$ mvn verify
[INFO] - - maven-enforcer-plugin:3.0.0-M3:enforce (enforce-linkage-checker) @ google-cloud-core-grpc - -
[ERROR] Linkage Checker rule found 21 reachable errors. Linkage error report:
Class org.eclipse.jetty.npn.NextProtoNego is not found; referenced by 1 class file
io.grpc.netty.shaded.io.netty.handler.ssl.JettyNpnSslEngine (grpc-netty-shaded-1.23.0.jar)
…
```

这个特殊问题是由不完整的着色引起的，这是链接错误的一个非常常见的来源。

链接错误是否会在运行时导致问题取决于程序遵循的代码路径。这些问题可能潜伏数月未被发现，然后在生产中意外出现。您的程序可能永远不会调用带有链接错误的方法。另一方面，如果依赖于您的项目的另一个项目调用了这个方法，他们将会得到一个令人讨厌的惊喜。像这样的问题调试起来很困难，很昂贵，也很耗时。链接检查器 Enforcer 规则在开发周期的更早阶段就揭示了这些问题，那时修复它们更容易、更便宜。

要检查 Maven 项目中的链接，将这个`**plugin**`元素添加到 pom.xml:

```
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-enforcer-plugin</artifactId>
  <version>3.0.0-M3</version>
  <dependencies>
    <dependency>
      <groupId>com.google.cloud.tools</groupId>
      <artifactId>linkage-checker-enforcer-rules</artifactId>
      <version>1.1.4</version>
    </dependency>
  </dependencies>
  <executions>
    <execution>
      <id>enforce-linkage-checker</id>
      <phase>verify</phase>
      <goals>
        <goal>enforce</goal>
      </goals>
      <configuration>
        <rules>
          <LinkageCheckerRule implementation= "com.google.cloud.tools.dependencies.enforcer.LinkageCheckerRule">
            <level>WARN</level>
          </LinkageCheckerRule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

现在 **mvn verify** 警告你项目中的所有链接错误:缺少类、从另一个类调用私有方法、带有意外数量参数的方法等等。如果您希望在检测到链接错误时构建完全失败，请将级别设置为 error。

关于链接检查的更多细节可以在[规则的 Wiki](https://github.com/GoogleCloudPlatform/cloud-opensource-java/wiki/Linkage-Checker-Enforcer-Rule) 上找到。您可能还会对我们发布的管理 Java 库依赖关系的[最佳实践](https://jlbp.dev/)感兴趣。

除了最简单的 Java 项目之外，链接错误是所有项目的主要问题。链接检查器 Enforcer 规则提供了一种及早发现这些问题并在需要进行困难的调试之前修复它们的方法。