# 让我们在 2017 年加密和谷歌应用引擎

> 原文：<https://medium.com/google-cloud/lets-encrypt-and-google-app-engine-in-2017-7cfe0928768e?source=collection_archive---------0----------------------->

所以昨天我开始和 HTTPS 一起保护我的谷歌应用引擎(GAE) API。我已经设置了我的 Google App Engine 实例。我对“让我们加密”([https://letsencrypt.org/](https://letsencrypt.org/))有点熟悉，尽管我从未使用过它。我还基本掌握了加密和证书中使用的密码原语。

有很多关于这个的博客帖子，还有一堆 StackOverflow 的帖子，都有不同的说明。谷歌的官方版本似乎过时了，我在“让我们加密”上唯一能找到的是一个 bug，大意是“请添加对 GAE 的支持”。我尝试了一些，通过合并其中一些步骤(以及许多挫折和最终解决的困惑)，我让它工作了。我写这篇博文主要是为了自己(这样当我需要在几个月后重新加密时，我就知道怎么做了)，但希望有人会觉得有用。

【https://issuetracker.google.com/issues/35900034】***更新:*** *自动化证书管理即将推出:*[](https://issuetracker.google.com/issues/35900034)**)**

## *准备工作:*

1.  *LetsEncrypt 通过自动验证您是否拥有您声称拥有的域名来颁发 SSL 证书。(后面更多关于如何验证。)*
2.  *LetsEncrypt 客户端后来被重命名为 certbot，由 EFF:[https://certbot.eff.org/](https://certbot.eff.org/)维护*
3.  *LetsEncrypt 允许您在一个证书中覆盖多达 100 个域/子域。*
4.  *LetsEncrypt 实现了一个名为 ACME(自动管理证书环境)的协议，允许您永久更新他们的证书。*
5.  *LetsEncrypt 只发行 3 个月的证书。*
6.  *当 LetsEncrypt ' [支持托管提供商](https://community.letsencrypt.org/t/web-hosting-who-support-lets-encrypt/6920)'时，这意味着您可以使用他们的自动化软件来更新您的证书。如果您的提供商(*cough* Google App Engine *cough*)不受支持，您仍然可以获得 LetsEncrypt 证书(只需要多花一点力气)，并且您需要每三个月手动更新一次。*
7.  *(不是必要的信息，但知道这个很好)亚马逊显然有一个[证书管理器](https://aws.amazon.com/certificate-manager/)，可以为你自动化整个过程(和更新)。*

## *说明:*

1.  *安装 CertBot，我用过(运行在 Mac OSX 上):*

```
*brew install certbot*
```

*虽然有很多种安装方式:[https://certbot.eff.org/docs/install.html](https://certbot.eff.org/docs/install.html)*

*2.决定是否要通过 DNS(向您的 DNS 添加 TXT 记录，与验证 Google Analytics 或 Google 网站管理员工具的所有权相同)或 HTTP(向您的应用程序上传文件)来验证您的域。*

***对于 HTTP:***

```
*sudo certbot certonly --manual*
```

***对于 DNS(注意:DNS 可能需要一段时间更新，所以这可能比 HTTP 慢):***

```
*sudo certbot certonly --manual --preferred-challenges dns*
```

*3.遵照指示！CertBot 将要求您输入一个由空格或逗号分隔的域列表。然后，它会一个接一个地要求您通过您在上一步中选择的方法来验证它们(在您完成这些步骤时不要关闭终端，然后它会发出一个新的挑战)。*

***对于 HTTP:***

*它将要求您在 yourdomain.com/.well-known/acme-challenge/ABC 创建一个值为 XYZ 的文件(注意:ABC 和 XYZ 只是占位符，不是实际值)。在您的 google app 目录中创建名为 letsencrypt 的文件夹，在其中放置一个名为 ABC 的文件，内容为 XYZ，并将以下处理程序添加到您的 app.yaml 中:*

```
*handlers:
- url: /.well-known/acme-challenge
  static_dir: letsencrypt...*
```

*然后部署 app (gcloud app deploy app.yaml)！最后验证它的工作(通过访问 yourdomain.com/.well-known/acme-challenge/ABC，并确保它不 404)。如果这有效，那么在等待的 certbot 终端上按 enter 键。*

***对于 DNS:***

*它将要求您向 _acme-challenge.yourdomain.com 或 _ acme-challenge . your subdomain . your domain . com 添加一条 TXT 记录，内容为 XYZ(注意:XYZ 是一个占位符，不是一个真实值)。去你的域名提供商那里添加记录，因为名字很便宜*

*类型:TXT
主机:_acme-challenge(或 _ acme-challenge . yoursubdomain)
值:XYZ*

*然后保存它，等待它传播。您将能够知道它何时完成，因为值 XYZ 将显示在 DIG 终端命令的输出中:*

```
*$ dig -t txt _acme-challenge.yourdomain.comOR $ dig -t txt _acme-challenge.yoursubdomain.yourdomain.com*
```

*一旦出现，在 certbot 终端中按 enter 键。*

*4.恭喜你。Certbot 刚刚祝贺您获得了全新的证书，网址为:/etc/lets encrypt/live/your domain . com/full chain . PEM*

*前往 Google 的 App Engines SSL 证书页面(App Engine >设置> SSL 证书，或只是[https://console . cloud . Google . com/App Engine/Settings/Certificates](https://console.cloud.google.com/appengine/settings/certificates))，在这里你会被提示复制并粘贴证书和私钥(注意:你也可以上传文件，但这比较困难，因为只有 root 拥有/etc/letsencrypt/live/的读取权限，而 Chrome 没有 root 权限)。证书可以按原样上传，您可以使用以下命令将其复制到剪贴板:*

```
*$ sudo cat /etc/letsencrypt/live/yourdomain.com/fullchain.pem | pbcopy*
```

*在你上传私钥之前，你必须把它转换成 RSA 私钥(不同之处请看这里:[https://stackoverflow.com/a/20065522/781199](https://stackoverflow.com/a/20065522/781199))。为此，请运行以下命令:*

```
*$ sudo openssl rsa -inform pem -in /etc/letsencrypt/live/yourdomain.com/privkey.pem -outform pem > /etc/letsencrypt/live/yourdomain.com/rsaprivatekey.pem*
```

*(注意:如果您想要写出这样的文件，您将需要 root 权限，因为只有 root 在/etc/letsencrypt/live/上具有写权限，为此您可以使用“$ sudo su root”)。*

*然后复制并粘贴到 Google App Engine 上的私钥框中:*

```
*$ sudo cat /etc/letsencrypt/live/yourdomain.com/rsaprivatekey.pem | pbcopy*
```

*搞定了。你的网站现在被 HTTPS 保护了！别忘了 3 个月后更新你的证书🔐*