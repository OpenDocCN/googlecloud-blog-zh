# 使用谷歌云平台的博客商店

> 原文：<https://medium.com/google-cloud/using-google-cloud-platform-s-blobstore-553a01b6f464?source=collection_archive---------3----------------------->

**通过编写一个简单的地理标记照片的 Go web 应用程序**

一个朋友最近在脸书上看到了一张他的朋友上传的照片，他想知道这张照片是在哪里拍的。这张照片是一个人在公园里，他不能准确地指出是哪个公园，但有一个大概的想法。他问我是否有办法让他在地图上看到照片的位置。

在您确定照片的位置之前，必须对照片进行地理标记。 [Geotagged](https://www.wikiwand.com/en/Geotagged_photograph) 基本上是指照片的经纬度需要存储在照片元数据中。元数据是照片的不可见部分，称为 [EXIF](https://www.wikiwand.com/en/Exchangeable_image_file_format) 数据。根据相机的不同，EXIF 数据将存储照片拍摄时相机的当前状态，包括日期和时间、快门速度、焦距、闪光灯、镜头类型、位置数据等。

当然，你能看到照片拍摄地点的唯一方法是相机是否有 GPS 功能。如果您的相机没有任何类型的 GPS 选项，那么在 EXIF 数据中将不会有任何位置数据。大多数单反相机都是这样。但是，如果照片是用智能手机拍摄的，并且启用了定位服务，那么手机的 GPS 坐标将在您拍照时被捕获。

让我们编写一个 Go 程序，从照片的 EXIF 数据中提取经度和纬度信息，并显示照片拍摄地的地图。我们的应用程序将部署在谷歌应用引擎。

> 我是在研究 Blobstore Go API 的时候想到写这个围棋程序的。

**Blobstore Go API**

不用说，我广泛参考了 [Blobstore Go API](https://cloud.google.com/appengine/docs/go/blobstore/) 的官方文档，其中指出:

> Blobstore API 允许您的应用程序提供数据对象，称为**blob***、*，它们比数据存储服务中的对象所允许的大小大得多。Blobs 对于提供大型文件(如视频或图像文件)以及允许用户上传大型数据文件非常有用。Blobs 是通过 HTTP 请求上传文件来创建的。通常，您的应用程序会通过向用户显示一个带有文件上传字段的表单来实现这一点。当提交表单时，Blobstore 从文件的内容中创建一个 blob，并返回一个对 blob 的不透明引用，称为 **blob 键** *，*，稍后您可以使用它来提供 blob。

Blobs 创建后不能修改，但可以删除。我们将使用 Blobstore API 上传一张可能带有地理标记信息的照片。

**上传 blob**

1.  **创建上传网址**

包 [appengine](https://cloud.google.com/appengine/docs/go/reference) 为谷歌应用引擎提供了基本的功能。

函数“NewContext”返回正在进行的 HTTP 请求的上下文。重复调用将返回相同的值。

```
func NewContext(req *http.Request) Context
```

包 [blobstore](https://cloud.google.com/appengine/docs/go/blobstore/reference) 为 App Engine 的持久 blob 存储服务提供了一个客户端。

```
func UploadURL(c appengine.Context, successPath string, opts *UploadURLOptions) (*url.URL, error)
```

打电话给“博客商店”。UploadUrl”来为用户将要填写的表单创建一个上传 Url，当表单的 POST 完成时传递要加载的应用程序路径。这些 URL 过期，不应重复使用。opts 参数可能为零。

```
uploadURL, err := blobstore.UploadURL(context, "/upload", nil)
```

HTTP 处理程序使用 [ResponseWriter](https://golang.org/pkg/net/http/#ResponseWriter) 接口来构造 HTTP 响应。

“write header(int)”—发送带有状态代码的 HTTP 响应标头。如果没有显式调用 WriteHeader，则第一次调用“Write”将触发隐式的“WriteHeader(http。StatusOK)”。因此，对“WriteHeader”的显式调用主要用于发送错误代码。我们在函数“serveError”中使用它。

“func (h Header) Set(key，value string)”—将与“key”关联的标题条目设置为单个元素值。它会替换与“key”关联的任何现有值。在我们的程序中，我们将标题设置为:

```
w.Header().Set("Content-Type", "text/html")
```

**2。创建上传表单**

表单必须包括文件上传字段，并且表单的 **enctype** 必须设置为 **multipart/form-data** 。当用户提交表单时，Blobstore API 处理帖子，创建 blob。API 还为 blob 创建一个 info 记录，并将该记录存储在数据存储中，并将重写的请求作为 blob 键传递给给定路径上的应用程序。

**注意**:您必须为表单页面提供内容类型为“text/html；charset=utf-8”，或者任何包含非 ASCII 字符的文件名都将被错误解释。这是 Go 中的默认内容类型，但是如果您设置了自己的内容类型，您必须记住这样做。

**3。实施上传处理程序**

在这个处理程序中，您可以将 blob 键与应用程序数据模型的其余部分存储在一起。blob 键本身仍然可以从数据存储中的 blob 信息实体访问。请注意，在用户提交表单并调用您的处理程序之后，blob 已经被保存，blob 信息也被添加到数据存储中。如果您的应用程序不想保留 blob，您应该立即删除 blob，以防止它成为孤儿。

**4。使用 goexif**

我们首先使用函数:

```
func NewReader(c appengine.Context, blobKey appengine.BlobKey) Reader
```

“NewReader”返回 blob 的读取器。它总是成功的；如果 blob 不存在，则在第一次读取时会报告错误。blob 键作为 URL 参数“r.FormValue("blobKey ")”传递。

我们将使用 [goexif](https://godoc.org/github.com/rwcarlsen/goexif/exif) 包从图像文件中检索 exif 元数据。

要安装，请在命令窗口中键入:

```
go get github.com/rwcarlsen/goexif/exif
```

我们的函数“getLatLng”使用:

```
func Decode(r io.Reader) (*Exif, error)
```

“Decode”解析来自“Reader”r 的 EXIF 编码数据，并返回一个可查询的 Exif 对象。

```
func (x *Exif) LatLong() (lat, long float64, err error)
```

函数“LatLong”返回照片的纬度和经度(作为“float64”)以及它是否存在。我们的函数将“float64”值转换成一个字符串。

接下来，我们使用“删除”功能来删除 blob。

```
func Delete(c appengine.Context, blobKey appengine.BlobKey) error
```

**5。使用谷歌静态地图 API**

Google 静态地图 API 允许你在网页上嵌入 Google 地图图片，而不需要 JavaScript 或任何动态页面加载。Google 静态地图服务基于通过标准 HTTP 请求发送的 URL 参数创建您的地图，并将地图作为您可以在网页上显示的图像返回。

我们基于上面的 API 创建一个字符串，如下所示。基本 URL 是:

```
http://maps.googleapis.com/maps/api/staticmap?parameters
```

字符串中使用的附加参数有:

*   “zoom”(必需)定义地图的缩放级别，这决定了地图的放大级别。该参数采用与所需区域的缩放级别相对应的数值。在默认的路线图地图视图中，缩放级别可以在 0(最低缩放级别，可以在一张地图上看到整个世界)到 21+(下至单个建筑物)之间。
*   “size”(必需)定义地图图像的矩形尺寸。此参数采用{ horizontal _ value } x { vertical _ value }形式的字符串。例如，600x300 定义 600 像素宽 300 像素高的地图。
*   “maptype”(可选)定义要构建的地图的类型。有几种可能的 maptype 值，包括 roadmap、satellite、hybrid 和 terrain。
*   “center”(必需)定义地图的中心，与地图的所有边等距。此参数将位置作为逗号分隔的{纬度，经度}对(例如“40.714728，-73.998672”)或字符串地址(例如“纽约市政厅”)来标识地球表面上的唯一位置。
*   “标记”(可选)定义一个或多个标记以附加到图像的指定位置。此参数采用单个标记定义，参数由管道字符(|或%7C)分隔。

“标记”参数采用以下格式的赋值集(标记描述符):

```
markers=markerStyles|markerLocation1| markerLocation2|… etc.
```

“markerStyles”集合在标记声明的开头声明，由零个或多个由竖线字符(|)分隔的样式描述符组成，后跟一组由竖线字符(|)分隔的一个或多个位置。

标记样式描述符包含以下键/值分配:

*   " size ":(可选)从集合{tiny，mid，small}中指定标记的大小。如果没有设置大小参数，标记将以默认(正常)大小显示。
*   " color ":(可选)指定 24 位颜色(例如:color=0xFFFFCC)或{黑色、棕色、绿色、紫色、黄色、蓝色、灰色、橙色、红色、白色}集合中的预定义颜色。
*   " label ":(可选)从集合{A-Z，0–9 }中指定一个大写字母数字字符。

请务必阅读文章" [Go Code Organization](/@IndianGuru/go-code-organization-8d185b115c20) "以了解您的文件应该如何组织。

我们的完整程序存储在文件夹“$ GOPATH/src/github . com/satish talim/blobstrex”中

**6。文件 app.yaml**

> 如果您不熟悉 Google App Engine，请务必阅读文章“[下载并安装 Go 的 App Engine SDK](/google-cloud/go-cloud-endpoints-and-app-engine-e3413c01c484)”

上述文件存储在文件夹“$ GOPATH/src/github . com/satish talim/blobstrex”中。

**7。文件 upper.css**

在文件夹“$ GOPATH/src/github . com/satish talim/blobstrex”下创建一个名为“css”的文件夹。这个“css”文件夹包含文件“upper.css”。

您可以在本地测试应用程序，然后部署到 Google App Engine。

最后，[我们的应用程序](http://blobstrex-1081.appspot.com/)显示照片拍摄地点的地图，或者如果它没有 EXIF 数据，则显示错误“抱歉，您的照片没有地理标签信息…”。

**注释**:我希望您能对这些学习笔记提出反馈意见。如果我能让它变得更好，我会喜欢它！