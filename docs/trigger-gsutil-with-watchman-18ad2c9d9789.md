# 用 Watchman 触发 gsutil

> 原文：<https://medium.com/google-cloud/trigger-gsutil-with-watchman-18ad2c9d9789?source=collection_archive---------2----------------------->

[‘gsutil](https://cloud.google.com/storage/docs/gsutil)’是一个将文件从本地传输到 GCS 的工具，反之亦然。它提供了很大的功能，如易于使用的界面，并行文件传输，多部分上传等。它是一个命令行工具，这意味着它需要手动触发或从 cron 等其他进程来调度文件传输。

在某些用例中，用户希望在将文件写入给定目录后，立即将文件上传到给定的 GCS bucket。为了处理这种情况，我们可以用 [watchman](https://github.com/facebook/watchman) inotify 循环运行 great‘gsutil’。Watchman 将确保在文件上传到 watchman 监控的给定目录后立即触发 gsutil 命令。

# **安装守夜人**

使用以下步骤下载并安装 watchman。

```
# Download Watchmanwget [https://github.com/facebook/watchman/releases/download/v2022.06.06.00/watchman-v2022.06.06.00-linux.zip](https://github.com/facebook/watchman/releases/download/v2022.06.06.00/watchman-v2022.06.06.00-linux.zip)# Install Binaryunzip watchman-*-linux.zipcd watchman-[v2022.06.06.00](https://github.com/facebook/watchman/releases/download/v2022.06.06.00/watchman-v2022.06.06.00-linux.zip)-linuxsudo mkdir -p /usr/local/var/run/watchmansudo cp bin/* /usr/local/binsudo cp lib/* /usr/local/libsudo chmod 755 /usr/local/bin/watchmansudo chmod 2777 /usr/local/var/run/watchman
```

# 配置 Watchman

1.  创建一个自定义配置文件/etc/watchman.json

```
touch /etc/watchman.json
```

2.创建一个由 watchman 监控的目录

```
mkdir ~/upload
```

3.配置 watchman 来监控上传目录

```
# Run below command/usr/local/bin/watchman watch ~/upload/ # It will generate output similar to below one**# Output**{"version": "20220605.192726.0","watch": "/home/singhpradeepk/upload","watcher": "inotify"}
```

4.确认目录正由看守人监控

```
# Run below command/usr/local/bin/watchman watch-list # It will generate output similar to below one**# Output**{"version": "20220605.192726.0","roots": ["/home/singhpradeepk/upload"]}
```

5.用下面的内容写一个脚本 gsutil_rsync.sh。这个脚本将由 watchman 执行，将“上传”目录与 GCS bucket 同步。

```
#!/usr/bin/env bash# Change bucket name with desired value from your environment./usr/bin/gsutil rsync -rdc /home/singhpradeepk/upload gs://sample-bucket
```

6.使 gsutil_rsync.sh 可执行

```
chmod +x gsutil_rsync.sh
```

7.创建一个 watchman 触发器来运行该脚本，以便将任何更改上传到受监控的目录

```
# Run below command/usr/local/bin/watchman -j <<-EOT> ["trigger", "/home/singhpradeepk/upload", { "name": "gcs", "command": ["/home/singhpradeepk/gsutil_rsync.sh"]}]> EOT # It will generate output similar to below one# Output{"version": "20220605.192726.0","triggerid": "gcs","disposition": "created"}
```

7.测试看守人

现在，在“upload”目录中创建一个文件，并检查您的环境的日志文件，该文件应该位于与我的文件类似的位置，即传输日志的**'/usr/local/var/run/watchman/singhpradeepk-state/log '**。检查 GCS 存储桶以确认成功转移。

希望您将在您的环境中使用它来传输文件。

欢迎评论和建议！！