# Yosh 在基础设施用地

> 原文：<https://medium.com/google-cloud/yosh-in-infrastructure-land-36c54e1bb71f?source=collection_archive---------0----------------------->

过去一周左右，我一直在做基础设施方面的工作。不仅仅是因为我认为这很有趣(我真的这么认为)，这也是我正在做的这个客户项目的要求。我们的团队中没有专门的运营人员，所以我想我最好在回到 JS 领域之前做一些准备。这是我正在做的事情的一个简要概述。

## 站台

docker(1) 非常酷，因为它允许你创建微小的操作系统来运行你的应用。我非常喜欢为我的应用程序创建小的[阿尔卑斯](https://www.alpinelinux.org/)盒子。但是所有这些机器都需要在一台电脑上运行，所以我们得安装那台电脑。

我现在选择的电脑是[谷歌云](https://cloud.google.com/)。这是因为与 AWS 不同，UI 并不可怕。和它一起工作真的很好。我知道一些更“严肃”的操作人员可能会说 UI 不重要，重要的是功能。但是我跑题了；如果我不能理解用户界面，我将不会使用任何功能，所以嘿，谷歌云是我现在喜欢用的。

## 容器主机

所以在谷歌云上有一个叫做“谷歌容器引擎”(GKE)的东西。它允许您只需点击一下鼠标就可以设置一个 [kubernetes](http://kubernetes.io/) 集群。它和广告宣传的差不多。不过我喜欢把事情自动化到一个命令，所以我用了 [terraform(1)](https://www.terraform.io/) 和 [gcloud(1)](https://cloud.google.com/sdk/gcloud/) 来实现。

Kubernetes 很酷，因为它非常擅长管理容器。如果您曾经使用过 docker，您可能会记得写出了一大堆标志来挂载卷、端口、安全性等等。使用 kubernetes，您可以将这些标志写在一个(可读性很强的)文件中。yml 格式，然后在其上运行 *$ kubectl apply -f <文件>* 来将其部署到您的集群中。感觉真的很好。

Kubernetes 可以装各种东西。在没有任何停机时间的情况下部署东西也非常有效。甚至管理秘密也很有效。秘密也只是。包含一些字符串的 yml 文件。它们都存储在内存中，所以攻击者需要更加努力才能读取它们。

总而言之:我对 kubernetes 很满意。它并不完美，但感觉比手工管理容器/机器更上一层楼；而且肯定比 heroku 少很多魔法。

## 连续一切

所以能够运行容器很酷，但是我们要在上面运行什么呢？首先要做的是自动化。为了让我们的系统为我们工作，如果我们能够创建连续部署就好了。

我对持续部署的看法是这样的:

*   写了一段代码
*   代码被上传到 git 主机上的一个特性分支
*   测试运行，如果测试通过，我们可以将其合并到我们的开发分支
*   我们的开发分支被自动部署到我们的登台环境中
*   如果我们对我们的登台环境满意，我们将 dev 合并到 master
*   主服务器自动部署到生产环境中

我选择的工具是 [buildkite](https://buildkite.com) 。这是一个由了不起的人构建的半自托管自动化提供商。他们只是三个人在做一些超级好用而且看起来非常漂亮的东西。周末我一直穿着他们的[休闲装](https://chat.buildkite.com/)，他们给了我超级大的帮助。

无论如何，我现在已经为我的容器建立了一个完整的自动化设置，其中我创建了一个包含我的开发依赖项的基础构建，以及一个为生产做好准备的优化构建。我的目录中的 docker 文件现在具有这样的结构:

```
.buildkite
docker
  build/
    Dockerfile
  optimize/
    Dockerfile
```

我这样嵌套 docker 文件的原因是，不要给 docker 文件起不同的名字，这应该是最佳实践。啊，在这种情况下，我不介意过多地嵌套目录。

## 坏事

到目前为止，我在这个设置中遇到的唯一问题是 npm(1)为构建占用了大量内存。我的小机器一直在崩溃。幸运的是，我已经能够通过在 Node 中设置一些标志来解决这个问题。我的 docker 文件现在看起来像这样:

```
FROM mhart/alpine-node:4

# Create app directory
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

# Install img dependencies from: [https://pkgs.alpinelinux.org/packages](https://pkgs.alpinelinux.org/packages)
RUN apk update
RUN apk add jq

# Install app dependencies
COPY package.json /usr/src/app/
RUN /usr/bin/node \
  --max_semi_space_size=1 \
  --max_old_space_size=198 \
  --max_executable_size=148 \
  /usr/bin/npm install

# Add installed binaries to env
ENV PATH /app/node_modules/.bin:$PATH

# Bundle app source
COPY . /usr/src/app

EXPOSE 8080
CMD ["npm", "start"]
```

你在这里看到的所有节点标志防止内存使用失控，防止构建崩溃。唯一的缺点是，现在构建需要很长时间，但至少他们完成了。也许我应该尽快转移到内存超过 0.6GB 的节点，但谁知道呢？也许吧。

## 包扎

仅此而已。我现在在自己的服务器上运行 kubernetes 上的一大堆容器。由于我还没有为我的客户完成基础设施的设置，我可能会在未来几天/几周内做更多的工作。我热衷于为自己的基础设施添加更多的机器人。像[提及机器人](https://github.com/facebook/mention-bot)和 [ghb0t](https://github.com/jfrazelle/ghb0t) 这样的东西看起来是不错的补充；但我可以想象把管理机器人军团的想法推进一点。哈。

如果你感兴趣，这里是我一直在做的东西的笔记；也许会有用。

*   [https://github.com/yoshuawuyts/infrastructure](https://github.com/yoshuawuyts/infrastructure)
*   [https://github.com/yoshuawuyts/playground-docker-buildkite](https://github.com/yoshuawuyts/playground-docker-buildkite)
*   [https://github.com/yoshuawuyts/knowledge/blob/master/bin/docker.md](https://github.com/yoshuawuyts/knowledge/blob/master/bin/docker.md)
*   [https://github.com/yoshuawuyts/knowledge/blob/master/bin/gcloud.md](https://github.com/yoshuawuyts/knowledge/blob/master/bin/gcloud.md)
*   [https://github.com/yoshuawuyts/knowledge/blob/master/bin/kubectl.md](https://github.com/yoshuawuyts/knowledge/blob/master/bin/kubectl.md)

Cheers!⚠️

*Cross-posted from* [*https://github.com/yoshuawuyts/writing*](http://yoshuawuyts.com/writing/./2016-08-15-yosh-in-infrastructure-land)