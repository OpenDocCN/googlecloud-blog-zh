# 从应用程序引擎访问 BigQuery

> 原文：<https://medium.com/google-cloud/accessing-bigquery-from-app-engine-d01823de81ee?source=collection_archive---------0----------------------->

## 使用 OAuth 2.0 服务帐户验证 API 调用

一位谷歌云平台客户今天问我如何列出谷歌应用引擎应用程序中所有可用的 BigQuery 数据集。

如果你不知道什么是 BigQuery 或 App Engine，这篇文章可能不适合你…现在还不适合！相反，你应该看看 [BigQuery](https://cloud.google.com/bigquery/what-is-bigquery) 和 [App Engine](https://cloud.google.com/appengine/docs) 的文档。

![](img/1d6d1211229d7c0fe47c03f1446c1674.png)

BigQuery 和 App Engine 是我最喜欢的谷歌云平台的两个产品。

这个问题的解决方案很简单，但是我认为有足够多的移动内容，比一篇博文所需要的要多。让我们假设列表将作为请求处理的一部分来显示，如下所示:

```
func **handle**(w http.ResponseWriter, r *http.Request) {
    // create a new App Engine context from the request.
    c := appengine.**NewContext**(r) // obtain the list of dataset names.
    **names**, err := **datasets**(c)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    } w.Header().Set("Content-Type", "text") // print it to the output.
    if len(names) == 0 {
        fmt.Fprintf(w, "no datasets visible")
    } else {
        fmt.Fprintf(w, "datasets:\n\t" + strings.Join(**names,** "\n"))
    }}
```

每当一个新的 HTTP 请求到来时，上面定义的处理程序就会被执行。它将创建一个新的 App Engine 上下文，然后由*数据集*函数使用，返回一个列表，其中包含 App Engine 应用程序可见的所有 BigQuery 数据集的名称。最后，列表将被打印到输出端 *w* 。

为了注册处理程序，以便它在每个 HTTP 请求上执行，我们添加了一个 *init* func。

```
func **init**() { // all requests are handled by handler.
    http.HandleFunc(“/”, **handle**)}
```

现在到了有趣的部分，我们如何实现*数据集*函数？

```
func datasets(c context.Context) ([]string, error) { // some awesome code ...}
```

让我们分三部分实现该函数的主体:

## **1。创建经过身份验证的 HTTP 客户端**

给定一个上下文( [*)语境。Context*](https://godoc.org/golang.org/x/net/context#Context) )命名为 *c* 我们可以使用这段代码创建一个经过身份验证的客户端。

```
// create a new HTTP client.
client := &http.Client{
    Transport: &oauth2.Transport{
        Source: **google.AppEngineTokenSource**(c,
            bigquery.BigqueryScope),
        Base: &**urlfetch.Transport**{Context: c},
    },
}
```

在 Go 中，HTTP 客户端使用传输进行通信，传输使用…传输！一路都是运输机！但是什么是 HTTP 传输呢？

HTTP 客户端中的传输字段属于类型 [*往返者*](http://golang.org/pkg/net/http#RoundTripper) :

```
type RoundTripper interface {
        RoundTrip(*[Request](https://golang.org/pkg/net/http/#Request)) (*[Response](https://golang.org/pkg/net/http/#Response), [error](https://golang.org/pkg/builtin/#error))
}
```

一个 [*往返器*](http://golang.org/pkg/net/http#RoundTripper) 负责生成一个给定请求的响应，但是它也有能力改变请求和响应的设置。这对于监控、检测和身份验证非常有用。

[*oauth2。传输*](https://godoc.org/golang.org/x/oauth2#Transport) 类型，给定一个请求，添加认证头并通过其基本传输转发请求。

```
&oauth2.Transport{
    Source: google.**AppEngineTokenSource**(c, **bigquery.BigqueryScope**),
    Base: &urlfetch.Transport{Context: c},
}
```

上面的代码片段创建了一个新的 [*oauth2。Transport*](https://godoc.org/golang.org/x/oauth2#Transport) 使用 App Engine 项目的默认服务帐户对请求进行身份验证，然后使用由[*URL fetch*](https://cloud.google.com/appengine/docs/go/urlfetch/)*—*提供的 HTTP 传输方式从 App Engine 应用程序访问外部资源。

## **2。创建一个大查询服务并列出所有数据集**

用“[google.golang.org/api/bigquery/v2](http://google.golang.org/api/bigquery/v2)”创建一个大查询服务非常简单。只需导入包，并使用 [*bigquery 为 HTTP 客户端创建一个新服务。新增*](https://godoc.org/google.golang.org/api/bigquery/v2#New) 。

```
bq, err := **bigquery.New**(client)
if err != nil {
    return nil, fmt.Errorf("create service: %v", err)
}
```

检查错误是否不为零很重要，因为该操作可能会像网络上的任何其他操作一样失败。一旦我们有了服务，我们就可以按照文档列出该项目中可见的所有数据集:

```
// obtain the current application id, the BigQuery id is the same
appID := appengine.AppID(ctx)datasets, err := **bq.Datasets.List(appID).Do()**
if err != nil {
    return nil, fmt.Errorf("could not list datasets: %v", err)
}
```

## 3.创建包含项目名称的列表

我们得到了*列表*，一个 [*bigquery。DatasetList*](https://godoc.org/google.golang.org/api/bigquery/v2#DatasetList) *，*这样我们可以迭代*项目*中的所有项目，并在返回之前将它们的 *Id* 添加到我们的列表中。

```
var ids []stringfor _, d:= range datasets.Datasets {
    ids = append(ids, d.Id)
}return id, nil
```

结果代码可在 [GitHub](https://github.com/GoogleCloudPlatform/golang-samples/tree/master/appengine/bigquery) 上获得。

希望这对你们很多人有帮助！欢迎在 [Twitter](http://twitter.com/francesc) 上提问或发送建议！