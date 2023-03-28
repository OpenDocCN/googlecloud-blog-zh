# 一个方便的本地开发技巧，非常适合 Google 的 PHP 图像

> 原文：<https://medium.com/google-cloud/a-handy-local-dev-tip-that-works-great-with-googles-php-image-76d4acd2d789?source=collection_archive---------2----------------------->

首先，让我强调一下…如果您因为文件权限而在使用 docker 卷进行本地开发时遇到问题，这实际上可能会对您有所帮助！因此，我将尽可能保持说明的通用性，这样您就可以使用任何技术组合来完成这项工作。

# 设置 Docker 容器

在我的例子中，创建以下 docker 文件很简单:

```
FROM gcr.io/google-appengine/php:latest
```

> **注意:**对任何 docker 映像使用:latest 标签都不是好的做法，因为可能会发布新版本的容器，这会破坏未来的构建或部署。如果您计划将 Docker 容器部署到生产环境中，您应该使用大头针标签。

这个映像所做的是将 does 文件所在目录中的任何内容复制到容器的/app 目录中。然后检查/app 中的 composer.json 文件，查看指定的 php 版本(如果有的话),然后使用 nginx 作为反向代理，在 supervisord 下运行一个 php-fpm 实例。

在您的特定设置中，您可能只需要一个用 mod_php 启动 apache2 的容器。无论您的设置是什么，您都需要知道下一步将运行 php 的进程是 apache2 还是 php-fpm。

# 找到关于 docker 图像的更多信息

现在我们有了 docker 文件，我们想运行它，然后跳到容器内部看看发生了什么:

```
$ docker build -t myapp .
...
$ docker run -v "/${PWD#*/}":/app myapp
2413fb3709b05939f04cf2e92f7d0897fc2596f9ad0b8a9ea855c7bfebaae892$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                 NAMES
2413fb3709b0        myapp               "docker-entrypoint.s…"   5 seconds ago         Up 5 seconds                                  kicking_tyrant$ docker exec -it kicking_tyrant bash
root@2413fb3709b0:/app# top
top - 10:53:54 up  5:21,  0 users,  load average: 0.05, 0.02, 0.00
Tasks:   9 total,   1 running,   8 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.7 us,  0.7 sy,  0.0 ni, 98.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  4037124 total,   206692 free,  1885512 used,  1944920 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  1807612 avail MemPID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND       
    1 root      20   0   55736  17820   6820 S   0.0  0.4   0:00.19 supervisord   
   23 root      20   0  366168  22488  18208 S   0.0  0.6   0:00.06 php-fpm       
   24 root      20   0  172780  14548  11560 S   0.0  0.4   0:00.03 nginx         
   25 www-data  20   0  174168   6660   2356 S   0.0  0.2   0:00.00 nginx         
   26 www-data  20   0  174168   6660   2356 S   0.0  0.2   0:00.00 nginx         
   27 www-data  20   0  366168   7416   3132 S   0.0  0.2   0:00.00 php-fpm       
   28 www-data  20   0  366168   7416   3132 S   0.0  0.2   0:00.00 php-fpm       
   29 root      20   0   18240   3244   2796 S   0.0  0.1   0:00.02 bash          
   38 root      20   0   36628   3040   2620 R   0.0  0.1   0:00.00 top
```

这里需要注意的重要一点是，php-fpm 和 nginx 一样是作为 www-data 运行的。如果这是一个不同的设置，您可能会看到 php-fpm + apache 或只是 apache2(或 httpd2)。

现在我们知道了它是在哪个用户上运行的，我们需要确定 www-data 的用户 id 是什么，让我们看看我们在主机上的用户 id 是什么:

```
root@2413fb3709b0:/app# id -u www-data
33root@2413fb3709b0:/app# exit
$ id -u
1000
```

现在我们知道了容器用于 www-data 用户的默认用户 id。现在我们可以回到 docker 文件，并修改它，以包括一些额外的好处，为我们的黑客准备。

# 为我们的档案增添美好

让我们在 docker 文件中添加两条简单的语句，如下所示:

```
FROM gcr.io/google-appengine/php:latestARG WWW_DATA_UID=33
RUN usermod -u $WWW_DATA_UID www-data && groupmod -g $WWW_DATA_UID www-data
```

它的作用是在 docker 映像构建过程中查找名为 WWW_DATA_UID 的构建参数，然后将 www-data 的用户 ID 和组 id 更改为 WWW_DATA_UID 的值。如果我们不指定任何东西，它使用我们发现图像附带的默认值(33)，所以它是生产安全的(tm)。

# 这里是奇迹发生的地方

我们拼图的最后一块是在本地构建 docker 文件时添加一些额外的优点。让我们使用上面的运行示例，但修改为传入 WWW_DATA_UID 构建参数:

```
$ docker build -t myapp --build-arg WWW_DATA_UID=$(id -u) .
...
$ docker run -v "/${PWD#*/}":/app myapp
2413fb3709b05939f04cf2e92f7d0897fc2596f9ad0b8a9ea855c7bfebaae892
```

瞧啊。如果我们现在看一下容器内部，我们会看到 www-data 的用户 id 已经更改，因此容器外部文件的用户 id 与容器内部文件的用户 id 相匹配(让我们假设容器第二次仍具有相同的名称):

```
$ docker exec -it kicking_tyrant bash
root@2413fb3709b0:/app# id -u www-data
1000
```

# 加分！

这就是它的全部，但是如果你使用的图像和我在例子中使用的完全一样，[看一下我以前的文章](https://link.medium.com/1P1gHRgaAY)，这将帮助你避免讨厌的(你不能写这个文件)错误，因为 docker 图像试图锁定所有的文件。

请在下面评论，让我知道这是如何为你工作的！