# Go 中的云运行服务调用工作流回调端点

> 原文：<https://medium.com/google-cloud/a-cloud-run-service-in-go-calling-a-workflows-callback-endpoint-7827ce5b77bc?source=collection_archive---------0----------------------->

都是理查德·塞罗特的错，我最终和[戈朗](https://go.dev/)有了瓜葛！我们讨论了使用谷歌云[工作流](https://cloud.google.com/workflows)和在 Go 中实现的[云运行](https://cloud.run/)服务的用例。所以这是一个玩围棋的机会。嗯，我还是不喜欢错误处理…但是让我们倒回一下故事吧！

Workflows 是 Google Cloud 上的一个完全托管的服务/API orchestrator。您可以使用 YAML 语法创建一些高级业务工作流。我用它构建了许多小项目，并且[在博客](https://cloud.google.com/blog/topics/developers-practitioners/introducing-workflows-callbacks)上写了关于它的内容。我特别喜欢它暂停工作流执行的能力，创建一个[回调端点](https://cloud.google.com/workflows/docs/creating-callback-endpoints)，您可以从外部系统调用它来恢复工作流的执行。通过回调，您能够实现人工验证步骤，例如在一个费用报告应用程序中，经理验证或拒绝团队中某个人的费用(这是我在这篇文章[中实现的)。](https://cloud.google.com/blog/topics/developers-practitioners/smarter-applications-document-ai-workflows-and-cloud-functions)

对于我和 Richard 的用例，我们有一个创建这样一个回调端点的工作流。这个端点是从 Go 中实现的云运行服务调用的。让我们看看如何实现工作流:

```
main:
    params: [input]
    steps: - create_callback:
        call: events.create_callback_endpoint
        args:
            http_callback_method: "POST"
        result: callback_details - log_callback_creation:
        call: sys.log
        args:
            text: ${"Callback created, awaiting calls on " + callback_details.url} - await_callback:
        call: events.await_callback
        args:
            callback: ${callback_details}
            timeout: 86400
        result: callback_request - log_callback_received:
        call: sys.log
        args:
            json: ${callback_request.http_request} - return_callback_request:
        return: ${callback_request.http_request}
```

上面的工作流定义创建了一个回调端点。第一步返回回调端点的 URL。那么工作流正在等待从外部调用回调端点。然后继续执行，记录一些关于传入调用的信息并返回。

我用一个服务帐户部署了该工作流，该服务帐户具有工作流编辑者角色、日志写入者角色(记录信息)和服务帐户令牌创建者角色(创建 OAuth2 令牌)，正如文档中的[所解释的那样。](https://cloud.google.com/workflows/docs/creating-callback-endpoints#oauth-token)

现在我们来看看 Go 服务。我做了一个 go mod init 来创建一个新项目。我创建了一个 main.go 源文件，内容如下:

```
package mainimport (
    metadata "cloud.google.com/go/compute/metadata"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "os"
    "strings"
)
```

元数据模块用于从[云运行元数据服务器](https://cloud.google.com/run/docs/container-contract#metadata-server)获取 OAuth2 令牌。

```
// OAuth2 JSON struct
type OAuth2TokenInfo struct {
     // defining struct variables
     Token      string `json:"access_token"`
     TokenType  string `json:"token_type"`
     Expiration uint32 `json:"expires_in"`
}
```

instance/service-accounts/default/token 中的元数据信息返回一个 JSON 文档，我们用上面的 struct 映射它。我们对 access_token 字段感兴趣，我们使用它对工作流回调端点进行身份验证调用。

```
func main() {
    log.Print("Starting server...")
    http.HandleFunc("/", handler) // Determine port for HTTP service.
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
        log.Printf("Defaulting to port %s", port)
    } // Start HTTP server.
    log.Printf("Listening on port %s", port)
    if err := http.ListenAndServe(":"+port, nil); err != nil {
        log.Fatal(err)
   }
}
```

main()函数启动我们的 Go 服务。现在让我们更详细地看看 handler()函数:

```
func handler(w http.ResponseWriter, r *http.Request) {
    callbackUrl := r.URL.Query().Get("callback_url")
    log.Printf("Callback URL: %s", callbackUrl)
```

我们找回了。将包含回调端点 url 的 callback_url 查询参数。

```
 // Fetch an OAuth2 access token from the metadata server
    oauthToken, errAuth := metadata.Get("instance/service-accounts/default/token")
    if errAuth != nil {
        log.Fatal(errAuth)
    }
```

上面，我们通过 metadata Go 模块调用了元数据服务器。然后，我们在之前定义的结构中解组返回的 JSON 文档，代码如下:

```
 data := OAuth2TokenInfo{}
    errJson := json.Unmarshal([]byte(oauthToken), &data)
    if errJson != nil {
        fmt.Println(errJson.Error())
    }
    log.Printf("OAuth2 token: %s", data.Token)
```

现在是准备调用我们的工作流回调端点的时候了，有一个 POST 请求:

```
 workflowReq, errWorkflowReq := http.NewRequest("POST", callbackUrl, strings.NewReader("{}"))
    if errWorkflowReq != nil {
        fmt.Println(errWorkflowReq.Error())
    }
```

我们通过报头添加 OAuth2 令牌作为载体授权:

```
 workflowReq.Header.Add("authorization", "Bearer "+data.Token)
    workflowReq.Header.Add("accept", "application/json")
    workflowReq.Header.Add("content-type", "application/json") client := &http.Client{}
    workflowResp, workflowErr := client.Do(workflowReq)
    if workflowErr != nil {
        fmt.Printf("Error making callback request: %s\n", workflowErr)
    }
    log.Printf("Status code: %d", workflowResp.StatusCode) fmt.Fprintf(w, "Workflow callback called. Status code: %d", workflowResp.StatusCode)
}
```

我们只是在 Go 服务结束时返回状态代码。

为了部署 Go 服务，我简单地使用了源部署方法，运行 gcloud run deploy，并回答一些问题(服务名、区域部署等。)几分钟后，服务启动并运行。

我从 Google Cloud 控制台创建了一个新的工作流执行。一旦启动，它就记录回调端点 URL。我复制它的值，然后用？callback_url=指向该 url 的查询字符串。瞧，服务恢复工作流的执行，工作流结束。

*最初发表于*[T5【https://glaforge.appspot.com】](https://glaforge.appspot.com/article/a-cloud-run-service-in-go-calling-a-workflows-callback-endpoint)*。*