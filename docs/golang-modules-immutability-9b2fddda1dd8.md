# Golang 模和不变性

> 原文：<https://medium.com/google-cloud/golang-modules-immutability-9b2fddda1dd8?source=collection_archive---------0----------------------->

昨天写了一篇[总结](/google-cloud/golang-before-after-modules-273b5a5df838)我最近切换到 Go 的模块。在结论中，我写道我将在我的项目中转向单一的`${GOPATH}`。模块的优点之一是，包的版本应该是不可变的。这意味着，一旦你拉了一次包，你就不应该(必须)再拉一次包。

但是，当然，如果你只使用一台机器，那也是可行的。但是，举例来说，当你使用 Docker 时会发生什么呢？有没有办法把这个扩展到 Google Cloud Build？

## 码头工人建造

让我们在示例中添加一个 distroless Docker 构建。在`${WORKDIR}`中创建这个 Dockerfile:

```
FROM golang:1.12 as buildRUN printf "[timer] start\t%s\n" $(date +%s%N)COPY ./foo /foo
WORKDIR /foo
RUN GO111MODULE=on GOPROXY=[https://proxy.golang.org](https://proxy.golang.org) go build fooRUN printf "[timer] end\t%s\n" $(date +%s%N)FROM gcr.io/distroless/baseCOPY --from=build /foo /
CMD ["/foo"]
```

以下是运行 10 次的结果，确保 Docker 不会在使用`--no-cache`的构建之间缓存层:

```
for t in {1..10}
do
  docker build \
  **--no-cache** \
  --tag=foo:latest . \
  | grep "^\[timer\]"
done[timer] start 1562692830352570907
[timer] end   1562692833701779047
[timer] start 1562692837743692624
[timer] end   1562692841088710635
[timer] start 1562692845149766319
[timer] end   1562692848465842721
[timer] start 1562692852464160691
[timer] end   1562692855829662065
[timer] start 1562692859765082861
[timer] end   1562692863149075221
[timer] start 1562692867096511753
[timer] end   1562692870563710328
[timer] start 1562692874552237719
[timer] end   1562692877931096022
[timer] start 1562692881991442336
[timer] end   1562692885298613761
[timer] start 1562692889252781564
[timer] end   1562692892541429803
[timer] start 1562692896460210629
[timer] end   1562692899837684581
```

其根据 Sheets 具有 3357913011 的平均值和 50407471 纳秒的标准偏差。

## Docker 副本

这不是一个很好的解决方案，但是不可能在 Docker 构建期间挂载 Docker 卷(这样会更好)。修改(或创建另一个 docker 文件):

```
FROM golang:1.12 as buildRUN printf "[timer] start\t%s\n" $(date +%s%N)**COPY ./go/pkg /go** COPY ./foo /fooWORKDIR /foo
RUN GO111MODULE=on GOPROXY=[https://proxy.golang.org](https://proxy.golang.org) go build fooRUN printf "[timer] end\t%s\n" $(date +%s%N)FROM gcr.io/distroless/baseCOPY --from=build /foo /
CMD ["/foo"]
```

这一次，我们将通过将`golang-module-mirror`作为`/go`安装到容器中来运行测试。在我们将丢弃的第一次运行中，将创建我们的镜像，后续运行将使用它:

```
[timer] start 1562694281225041674
[timer] end   1562694285811156939
[timer] start 1562694289319807596
[timer] end   1562694293776883985
[timer] start 1562694297206823489
[timer] end   1562694301530111138
[timer] start 1562694304936211074
[timer] end   1562694309252913601
[timer] start 1562694312650657096
[timer] end   1562694316880599577
[timer] start 1562694320360184122
[timer] end   1562694324700460041
[timer] start 1562694328110722373
[timer] end   1562694332475305957
[timer] start 1562694335900959477
[timer] end   1562694340500560831
[timer] start 1562694343962632837
[timer] end   1562694348394060653
[timer] start 1562694351868504357
[timer] end   1562694356285662159
```

大概是因为我的本地缓存如此之小(只有`glog`)，所以到达`COPY`的时间大大超过了这里的好处。平均值为 4406618035，标准偏差为 117921035.6。

## 谷歌云构建

我写过几次关于 Google Cloud Build 的文章。这是一个低调而引人注目的服务，它提供了一种机制来运行一系列容器图像，将结果从一个传递到下一个。最(！)的时候，该服务用于构建容器映像，但是它可以用于构建更多的资产。

我意识到，默认情况下，使用一个`library/Golang`映像而不是`gcr.io/cloud-builders/go`映像，需要在构建步骤之间持久化，例如`/go`，以便两个不同的`golang`步骤可以共享一个`${GOPATH}`。

解决方案是使用云构建卷(见下文)。在本例中，名为`go-modules`的卷被创建并安装在`/go`上，用于需要它的第一个步骤，并且该卷一直存在，直到没有其他步骤引用它。对于随后引用它的每个步骤，例如`go get`包的缓存被保留。

> **2019–07–10 更新**:我了解到云构建允许一个`options`节([链接](https://cloud.google.com/cloud-build/docs/build-config#options))，允许`env`和`volumes`节被定义一次并应用于每一步，而不是像我下面这样在每一步重复。

```
steps:- name: golang:1.12
  env:
  - GOPATH=/go
  - GO111MODULE=on
  - GOPROXY=[https://proxy.golang.org](https://proxy.golang.org)
  **dir**: foo
  args:
  - go
  - build
  - -o
  - /go/bin/**test1**
  - foo
  **volumes**:
  - name: go-modules
    path: /go- name: golang:1.12
  env:
  - GOPATH=/go
  - GO111MODULE=on
  - GOPROXY=[https://proxy.golang.org](https://proxy.golang.org)
  **dir**: foo
  args:
  - go
  - build
  - -o
  - /go/bin/**test2**
  - foo
  **volumes**:
  - name: go-modules
    path: /go- name: golang:1.12
  env:
  - GOPATH=/go
  - GO111MODULE=on
  - GOPROXY=[https://proxy.golang.org](https://proxy.golang.org)
  **dir**: foo
  args:
  - go
  - build
  - -o
  - /go/bin/**test3**
  - foo
  **volumes**:
  - name: go-modules
    path: /go- name: busybox
  args:
  - ls
  - -l
  - /go/bin
  **volumes**:
  - name: go-modules
    path: /go
```

> 我认为`GOPATH=/go`在这里是多余的，因为这是`Golang`图像的默认工作目录。
> 
> 因为云构建使用`/workspace`作为工作目录，我们的源代码将位于`/workspace/foo`而不是我们的包等。被持久保存在`/go`中。
> 
> 因为我们的源代码在`/workspace`的子目录中，所以我使用云构建的`dir`修饰符使`/workspace/foo`成为构建的工作目录。
> 
> **注意**我正在使用`GOPROXY`来利用 Golang 团队的围棋模块镜像。

因此，在这个不可否认的小例子中，每次构建(相同的东西)都会创建一个不同名称的二进制文件(`testX`)并将它放在共享的`/go/bin`目录中。

这样做的目的是展示在第一次构建时，包是如何缓存在`/go/pkg`中的:

```
gcloud builds submit \
--config=./cloudbuild.yaml \
--project=[[YOUR-PROJECT]]...
starting build "22137c6c-df17-418b-8177-c231dd34b719"FETCHSOURCE
...
BUILD
Starting Step #0
Step #0: Pulling image: golang:1.12
Step #0: 1.12: Pulling from library/golang
Step #0: Digest: sha256:3fee5835...
Step #0: Status: Downloaded newer image for golang:1.12
**Step #0: go: finding github.com/golang/glog v0.0.0**
Finished Step #0
Starting Step #1
Step #1: Already have image: golang:1.12
Finished Step #1
Starting Step #2
Step #2: Already have image: golang:1.12
Finished Step #2
Starting Step #3
Step #3: Pulling image: busybox
Step #3: Using default tag: latest
Step #3: latest: Pulling from library/busybox
Step #3: Digest: sha256:bf510723...
Step #3: Status: Downloaded newer image for busybox:latest
Step #3: total 5892
Step #3: **test1**
Step #3: **test2**
Step #3: **test3**
Finished Step #3
PUSH
DONE
```

但是，如果我们删除|注释掉对`volumes`的引用，并删除`busybox`步骤，因为在`go/bin`中没有创建任何文件，然后重新运行构建:

```
starting build "3837eb7d-dae6-476e-9e72-67ee2f57de86"FETCHSOURCE
...
BUILD
Starting Step #0
Step #0: Pulling image: golang:1.12
Step #0: 1.12: Pulling from library/golang
Step #0: Digest: sha256:3fee5835...
Step #0: Status: Downloaded newer image for golang:1.12
**Step #0: go: finding github.com/golang/glog v0.0.0**
Finished Step #0
Starting Step #1
Step #1: Already have image: golang:1.12
**Step #1: go: finding github.com/golang/glog v0.0.0**
Finished Step #1
Starting Step #2
Step #2: Already have image: golang:1.12
**Step #2: go: finding github.com/golang/glog v0.0.0**
Finished Step #2
PUSH
DONE
```

这一次，您将看到每次运行构建时都提取了`glog`包。这是因为我们没有在构建步骤中共享`go/pkg`。

例如，如果您有一个现有的多阶段 docker 文件，或者您有一个包含几个二进制文件的 repo，并且您希望为每个二进制文件发出不同的容器映像，那么我怀疑这种机制会节省时间。