# 使用 Python oauth2client 和 App Engine Standard

> 原文：<https://medium.com/google-cloud/using-python-oauth2client-with-app-engine-standard-fd121518965b?source=collection_archive---------1----------------------->

我最近开始开发我的第一个应用引擎应用，这是在谷歌的[Python 应用引擎标准环境](https://cloud.google.com/appengine/docs/standard/python/quickstart)快速入门之后。

我的应用程序需要能够使用 [OAuth 2.0](https://developers.google.com/api-client-library/python/guide/aaa_oauth) 验证一些 Google APIs。从 App Engine 中有几种方法可以做到这一点，但所有这些方法都要求您能够访问 [oauth2client](https://github.com/google/oauth2client) 。

我很自然地认为 **oauth2client** 会自动包含在 App Engine 标准环境中，因为它是作为[Google-API-python-client](https://developers.google.com/api-client-library/python/start/installation)的依赖项之一安装的。但是客户端库**默认[不包含](https://developers.google.com/api-client-library/python/start/installation#appengine)**。

> **App 引擎**
> 
> 因为 Python 客户端库没有安装在 [App Engine Python 运行时环境](https://cloud.google.com/appengine/docs/python/)中，所以它们必须[像第三方库一样出售到您的应用程序](https://cloud.google.com/appengine/docs/python/tools/libraries27#vendoring)中。

Google 提供了将第三方库安装到应用引擎应用程序中的文档。以下是我遵循的流程。

1.  在项目的根目录下创建一个新的空目录:`mkdir lib`
2.  创建 **requirements.txt** : `echo 'oauth2client' > requirements.txt`
3.  安装要求:`pip install -U -t lib -r requirements.txt`
4.  创建 **appengine_config.py** :

```
# appengine_config.py
**from google.appengine.ext import vendor**

# Add any libraries install in the "lib" folder.
**vendor.add('lib')**
```

现在，我可以在本地和部署时将来自 Google oauth2client 的模块和类包含在我的 App Engine 应用程序中。

**附加参考文献**

*   [管理 App Engine 上出售的软件包](http://blog.jonparrott.com/managing-vendored-packages-on-app-engine/) ( [乔恩·韦恩·帕罗特](http://www.jonparrott.com/)
*   [使用 pip](https://sookocheff.com/post/appengine/managing-app-engine-dependencies-with-pip/) 管理应用引擎依赖关系
*   [服务账户、已安装应用和 Appengine 的简单 Google API 认证示例](/google-cloud/simple-google-api-auth-samples-for-service-accounts-installed-application-and-appengine-da30ee4648)(介质上的 [@salmaan.rashid](/@salmaan.rashid) )
*   [无法使用 gcloud 运行 dev _ appserver.py】(堆栈溢出)](https://stackoverflow.com/questions/34584937/unable-to-run-dev-appserver-py-with-gcloud/34585485#34585485)
*   [在 GAE 无法使用 Google-cloud](https://stackoverflow.com/questions/39704367/unable-to-use-google-cloud-in-a-gae-app)(堆栈溢出)