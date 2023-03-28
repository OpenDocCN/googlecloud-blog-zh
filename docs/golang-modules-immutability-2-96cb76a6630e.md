# Golang 模块和不变性#2

> 原文：<https://medium.com/google-cloud/golang-modules-immutability-2-96cb76a6630e?source=collection_archive---------1----------------------->

昨天，我使用 Docker 和 Cloud Build 探索了利用不可变 Golang 包库的一些方法。似乎有一种方法可以利用这种不变性，使用*解构的*多阶段构建。

## 多阶段构建

这是我的 Golang 多阶段构建和 Google 发行版的样板文件

```
FROM golang:1.12 as build
WORKDIR /app
COPY . .
RUN GO111MODULE=on GOPROXY=https://proxy.golang.org go build appFROM gcr.io/distroless/base
COPY --from=build /go/bin/app /
ENTRYPOINT ["/app"]
```

每次运行这个过程时，`golang`映像的`/go/pkg`将被与构建相关的包填充。

> **假设**:临时容器是匿名的，但是必须保存到磁盘上。应该可以发现这些过渡集装箱。但是…
> 
> **假设**:通过解构一个多阶段构建，应该有可能强制持久化临时容器，并且我们可以使用这样的临时容器来将来自多个构建的不同的包提取聚合到单个源中。

## 解构的多阶段构建

我正在设想像这样的种子步骤做一次你的第一个`go.mod`:

```
FROM **golang:1.12** as **origin** WORKDIR /app
COPY go.mod .
RUN GO111MODULE=on GOPROXY=https://proxy.golang.org **go** **mod** **download**
```

并建立图像:

```
docker build --tag=golang:modules --file=Dockerfile.orig .
```

现在我们有了基线`golang:modules`。

并且——随后——**为每个 Go 构建**——将这个构建的包添加到`golang:modules`而不是`golang`:

```
FROM **golang:modules** as **modules** WORKDIR /app
COPY go.mod .
RUN GO111MODULE=on GOPROXY=https://proxy.golang.org **go** **mod download**
```

> **NB** 我发现一个只包含`go.mod`的目录失败了，比如说`go get ./...`但是，有一个`go mod download`命令正是我们所需要的。

并(重新)构建`golang:modules`(包)映像:

```
docker build --tag=**golang:modules** --file=Dockerfile.mods .
```

> **注意**是的，`golang:modules`可能会因为增加了这么多层而变得有问题。Docker 有一个实验性的`docker build … -squash ...`命令可能会有所帮助。

然后，为了建立这个项目:

```
FROM **golang:modules** as **modules**FROM golang:1.12 as **build**
COPY --from=**modules** /go/pkg /go/pkg
WORKDIR /app
COPY . .
RUN GO111MODULE=on GOPROXY=https://proxy.golang.org **go build app**FROM gcr.io/distroless/base
COPY --from=**build** /foo /
ENTRYPOINT ["/foo"]
```

并建立图像:

```
docker build --tag=thisproject --file=Dockerfile.proj .
```

## 结论

使用这种——公认的更复杂的方法——我们(应该)能够在`golang:modules`中累积不同的包，这样，一旦被提取，一个包就被保留，并且即使在不同的构建中也不会被提取第二次。

可通过将两个构建语句绑定在一起来重构解构的多阶段构建 Dockerfile 文件。

仅此而已！