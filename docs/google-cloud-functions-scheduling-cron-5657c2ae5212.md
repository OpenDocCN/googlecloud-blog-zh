# 谷歌云功能:日程安排(Cron)

> 原文：<https://medium.com/google-cloud/google-cloud-functions-scheduling-cron-5657c2ae5212?source=collection_archive---------1----------------------->

![](img/75d0da55912af21f7ab01580709a78a5.png)

在[之前的一篇文章](/@viatsko/deploying-google-cloud-functions-in-5-easy-steps-21f6d837c6bb)中，我解释了部署谷歌云功能的 5 个简单步骤。在本文中，我将解释如何在 Google Cloud Standard App Engine 平台上为它设置 cron。

问题是 NodeJS 只在 App Engine Flexible 上可用，只是为了用 cron 踢 GCF 而设置 Flexible 实例没有意义。

相反，我们可以创建一个简单的 python 应用程序，并将其部署在 App Engine 标准平台上。

我们需要向应用程序添加 3 个文件:

app.yaml

```
runtime: python27
api_version: 1
threadsafe: truehandlers:
- url: /.*
  script: main.appskip_files:
  - ^node_modules/.*
```

cron.yaml

```
cron:
    - description: "regular job"
      url: /hourly
      schedule: every 1 hours
```

main.py

```
import webapp2
import urllib2class HourlyCronPage(webapp2.RequestHandler):
    def get(self):
        response = urllib2.urlopen('[<url_of_your_cloud_function>'](https://us-central1-open-source-awards.cloudfunctions.net/fetch-trending'))self.response.write(response.read())app = webapp2.WSGIApplication([
    ('/hourly', HourlyCronPage),
], debug=True)
```

创建这些文件后，部署 cronjob 运行程序的步骤如下:

```
gcloud app deploy
gcloud app deploy cron.yaml
```

尽情享受吧！🎉

你好，我是瓦莱里。我在阿姆斯特丹生活和写作。我写了这篇文章，所有观点都是我自己的。如果你喜欢读它，一定要在推特上关注我【https://twitter.com/viatsko 