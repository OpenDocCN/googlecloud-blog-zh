# Golang |复制到 GCS 并检查存储桶

> 原文：<https://medium.com/google-cloud/golang-copy-to-gcs-check-bucket-58721285788e?source=collection_archive---------0----------------------->

![](img/5d97c93862981713fff9996577b48795.png)

你是否在尝试使用 Go 将文件放入谷歌云存储，而不是将它们拖到运行代码的计算机上，或者打开它们并将内容读入一个新文件？如果是的话，我不久前也是。这里有一个快速的 PSA 来分享我整理的解决这个问题的代码。

# 将文件复制到 GCS

下面的代码示例可以将文件从一个 url 位置获取到 GCS 中，而无需将其下载到运行代码的服务器上。

```
import (
    "cloud.google.com/go/storage"
    "fmt"
    "io"
    "net/http"
)func storeGCS(url, bucketName, fileName string) error {
    // Create GCS connection
    ctx := context.Background()
    client, err := storage.NewClient(ctx) // Connect to bucket
    bucket = client.Bucket(bucketName) // Get the url response
    if response, err := http.Get(url); err != nil {
        return fmt.Errorf("HTTP response error: %v", err)
    }
    *Defer response.Body.Close()* if response.StatusCode == http.StatusOK {
        // Setup the GCS object with the filename to write to
        obj := bucket.Object(fileName)

        *// w implements io.Writer.* w := obj.NewWriter(ctx)

       *// Copy file into GCS
       if* _, err := io.Copy(w, response.Body); err != nil {return fmt.Errorf("Failed to copy to bucket: %v", err)
       }

       *// Close, just like writing a file. File appears in GCS after* if err := w.Close(); err != nil {
           return fmt.Errorf("Failed to close: %v", err)
       }
    }
    return nil
}
```

关键线路包括 **io。复制**，它将获取 HTTP 响应中的内容，并直接复制到 bucket 中。这个文件是什么格式并不重要，因为它不需要读取它。它只需要获取响应中的内容，并将其复制到 GCS。我发现这非常有用，尤其是在处理压缩文件的时候。这也适用于图像文件，但实际上也适用于任何文件。

这假设在您使用的计算机上设置了凭据，因此它将能够登录到您的 GCP 并加载到 GCS 中。此外，该代码假设 bucket 存在。否则，您需要创建一个存储桶。

# Golang 示例创建 GCS 存储桶(如果不存在)

下面是检查 bucket 是否存在的示例代码，如果不存在就创建它。

```
import (
    "cloud.google.com/go/storage"
    "context"
    "fmt"
    "google.golang.org/api/iterator"
    "log"
)func CreateGCSBucket(bucketName, projectID string) error {
   // Setup context and client
    ctx := context.Background()
    client := storage.NewClient(ctx)

    *// Setup client bucket to work from* bucket = client.Bucket(bucketName)

    buckets := client.Buckets(ctx, projectID)
    for {
if bucketName == "" {
            return fmt.Errorf("BucketName entered is empty %v.", bucketName)
          }
       attrs, err := buckets.Next()
       *// Assume bucket not found if at Iterator end and create* if err == iterator.Done {
           *// Create bucket* if err := bucket.Create(ctx, projectID, &storage.BucketAttrs{
            Location: "US",
           }); err != nil {return fmt.Errorf("Failed to create bucket: %v", err)
           }
           log.Printf("Bucket %v created.\n", bucketName)
           return nil
        }
        if err != nil {
            return fmt.Errorf("Issues setting up Bucket(%q).Objects(): %v. Double check project id.", attrs.Name, err)
        }
        if attrs.Name == bucketName {
log.Printf("Bucket %v exists.\n", bucketName)
            return nil
         }
   }
}
```

需要注意的关键一点是与**客户争夺该项目的所有份额。桶，**循环的**使用**桶遍历每个桶名。接下来**并确认迭代器不在末尾，**迭代器。搞定**。如果是，那么用**桶创建一个桶。创建**但是如果您发现带有**属性的存储桶列表中提供的存储桶名称。Name == bucketName** 那么您不需要创建它。**

# 包裹

上面提供了两个 Go 代码示例，重点是如何使用 Go 将文件复制到 GCS 中，归结起来就是使用 **io。复制**。此外，如何检查一个 bucket 是否已经存在，如果不存在，那么创建它。一个重要的测试包是 [Go Storage](https://godoc.org/cloud.google.com/go/storage) 包，它提供了关于如何与 GCS 交互的更多细节，以及关于如何测试与 GCS 交互的 [Stiface](https://github.com/googleapis/google-cloud-go-testing/tree/master/storage/stiface) 。

这些代码片段来自我正在从事的一个名为 Project OCEAN(开源社区生态系统焦点)的项目，该项目致力于对技术开源社区的公共结构和影响进行建模。你可以在我们的 [GitHub repo](https://github.com/google/project-OCEAN) 中查看关于上述代码的更多细节。