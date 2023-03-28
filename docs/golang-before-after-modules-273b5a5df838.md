# Golang 前| |后模块

> 原文：<https://medium.com/google-cloud/golang-before-after-modules-273b5a5df838?source=collection_archive---------2----------------------->

虽然我很熟悉(也很喜欢)旧的`${GOPATH}`方法，但为了保持最新，也因为它有很多好处，我已经开始使用 Go 模块。

和其他人一样，我发现这种转换令人困惑。所以…

## 模块之前

```
WORKDIR=[[PATH-TO-YOUR-WORKING-DIRECTORY]]mkdir -p ${WORKDIR}/go
export GOPATH=${WORKDIR}/go
export PATH=${GOPATH}/bin:${PATH}mkdir -p {${WORKDIR}/go/src/foo, ${WORKDIR}/go/src/foo/bar}
```

然后创建`${WORKDIR}/go/src/foo/bar/library.go`:

```
package barfunc Something() (string) {
    return "Hello Freddie"
}
```

然后创造`${WORKDIR}/go/src/foo/main.go`:

```
package mainimport (
    "fmt"
    "foo/bar"
)func main() {
    fmt.Printf("%s", bar.Something())
}
```

你会有这样一个结构:

```
.
└── go
    └── src
        └── foo
            ├── bar
            │   └── library.go
            └── main.go
```

然后，您可以通过以下任何一种方式运行它:

```
GO111MODULE=**off** go run ${WORKDIR}/go/src/foo/main.go
Hello Freddie!cd ${WORKDIR}/go/src/foo
GO111MODULE=**off** go run main.go
Hello Freddie!GO111MODULE=**off** go run foo
Hello Freddie!
```

> **注意**对于上述内容，您可以省略`GO111MODULE=off`，因为当源在`${GOPATH}`内时，这是隐含的。我将它包含在内是为了明确其意图。

## 模块后

假设你做到了以上几点！

您不需要执行此步骤，但这是新的最佳实践。我们正在将我们的资源转移到`${GOPATH}`之外。`${GOPATH}`仍然用于存储我们的版本化包。

> **注意**在这个简单的例子中，我们没有使用外部包，所以`${GOPATH}`保持为空。参见结尾部分的示例。

```
mv ${WORKDIR}/go/src/foo ${WORKDIR}
rm ${WORKDIR}/go/src
```

您应该有这样的结构:

```
.
├── **foo**
│   ├── bar
│   │   └── library.go
│   └── main.go
└── go
```

> **NB** 我们植根于`**foo**`的资源现在在`${GOPATH}`之外

```
cd ${WORKDIR}/fooGO111MODULE=**on** go mod init foo
more go.mod
module foogo 1.12GO111MODULE=**on** go run foo
Hello Freddie!GO111MODULE=**on** go run main.go
Hello Freddie!
```

## 为了完整性

以显示与外部包装的区别。不使用模块，`go get`将包的最新版本拉入`${GOPATH}/src`:

```
GO111MODULE=**off** go get github.com/golang/glog.
├── foo
│   ├── bar
│   │   └── library.go
│   ├── go.mod
│   ├── go.sum
│   └── main.go
└── **go**
    └── **src**
        └── github.com
            └── golang
                └── glog
                    ├── glog_file.go
                    ├── glog.go
                    ├── glog_test.go
                    ├── LICENSE
                    └── README
```

与使用模块相比，将包的一个特定版本(或多个版本)放入`${GOPATH}/pkg`:

```
GO111MODULE=**on** go get github.com/golang/glog.
├── foo
│   ├── bar
│   │   └── library.go
│   ├── go.mod
│   ├── go.sum
│   └── main.go
└── go
    └── **pkg**
        ├── linux_amd64
        │   └── github.com
        │       └── golang
        │           └── glog.a
        └── mod
            ├── cache
            │   ├── download
            │   │   └── github.com
            │   │       └── golang
            │   │           └── glog
            └── github.com
                └── golang
                    └── glog@v0.0.0-20160126235308-23def4e6c14b
                        ├── glog_file.go
                        ├── glog.go
                        ├── glog_test.go
                        ├── LICENSE
                        └── README
```

## Go 模块镜像

参见`[https://proxy.golang.org/](https://proxy.golang.org/)`

您将`PROXY=https://proxy.golang.org`包含在所有上述命令中，以利用 Golang 团队的(基于 Google Trillian 的)镜像:

```
GO111MODULE=on \
GOPROXY=https://proxy.golang.org \
go get github.com/golang/glog
```

## 欺骗

多亏了这个[链接](https://www.reddit.com/r/golang/comments/9blgn0/using_go_modules_with_vendorprovided_protobuf/)这个有用的技巧，当使用模块时，可以找到将要安装在`go/pkg/mod`中某处的包:

```
GO111MODULE=on \
go list -f '{{ .Dir }}' github.com/golang/gloggo: finding github.com/golang/glog latest
/path/to/go/pkg/mod/github.com/golang/glog@v0.0.0-20160126235308-23def4e6c14b
```

## Visual Studio 代码

一些人询问我对 Visual Studio 代码 w/ Go 模块的配置设置。以下是我所知道的:

```
{
    "go.autocompleteUnimportedPackages": true,
    "go.useLanguageServer": true,
    "[go]": {
        "editor.snippetSuggestions": "none",
        "editor.formatOnSave": true,
        "editor.codeActionsOnSave": {
            "source.organizeImports": true
        }
    },
    "gopls": {
        "usePlaceholders": true // add parameter placeholders when completing a function
    },
    "files.eol": "\n", // formatting only supports LF line endings    
    "go.toolsEnvVars": {
        "GO111MODULE": "on",
        "GOPROXY": "[https://proxy.golang.org](https://proxy.golang.org)"
    },
}
```

这个[页面](https://github.com/Microsoft/vscode-go/wiki/Go-modules-support-in-Visual-Studio-Code)的帮助和这个[来自谷歌工程师的回复](https://github.com/golang/go/issues/32903#issuecomment-507817147)的组合。我试着跟上潮流，但是 Golang 团队的 gopls 完全忽略了我:(

`[https://github.com/golang/go/wiki/gopls](https://github.com/golang/go/wiki/gopls)`

## 结论

模块是 Golang 包管理的[未来](https://github.com/golang/go/wiki/Modules)。现在开始暴跌，我主要看到了好处。即使使用我简单的 Golang repos，也可以定义导入的包的特定版本。

最初，我不理解(也许现在仍然不理解)不把我的源代码包含在`${GOPATH}`下的好处，但是，一个好处是，它变得更加明显，哪些源代码是我的源代码，哪些是其他的包。当一切都在`${GOPATH}`之下时，这是令人困惑的。

在某种程度上，语义版本化(又名 SemVer)的使用也意味着，一旦您已经`go get useful-package@v1.0.0`您应该(！)再也不需要重新`get`它了。我以前总是按照项目来区分我的`${GOPATH}`目录，以保持代码隔离。但是，通过将我的源代码从`${GOPATH}`中取出来，因为我只需要一份`useful-package@v1.0.0`的副本。)因为我知道这个包是不可变的，所以我现在将开始为我的所有项目使用一个单一的中央`${GOPATH}`。

仅此而已！