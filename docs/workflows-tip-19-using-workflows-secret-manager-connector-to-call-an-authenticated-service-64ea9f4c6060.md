# 工作流提示#19:使用工作流的 Secret Manager 连接器调用经过身份验证的服务

> 原文：<https://medium.com/google-cloud/workflows-tip-19-using-workflows-secret-manager-connector-to-call-an-authenticated-service-64ea9f4c6060?source=collection_archive---------3----------------------->

Workflows 允许您调用 API，无论是来自 Google Cloud 还是托管在 Google Cloud 上，或者任何外部 API。例如，几天前，我们在[上看到了一个如何使用 SendGrid API 从工作流](https://glaforge.appspot.com/article/sending-an-email-with-sendgrid-from-workflows)发送电子邮件的例子。然而，在那篇文章中，我将 API 键硬编码到我的工作流中，这是一个糟糕的做法。相反，我们可以在[秘密管理器](https://cloud.google.com/secret-manager)中存储秘密。Workflows 为 Secret Manager 提供了一个特定的[连接器，以及一个访问机密的有用方法。](https://cloud.google.com/workflows/docs/reference/googleapis/secretmanager/Overview)

在本文中，我们将了解两件事:

*   如何使用工作流连接器访问存储在 Secret Manager 中的机密
*   如何调用需要基本认证的 API

让我们访问我对我需要调用的 API 进行基本授权调用所需的秘密:

```
- get_secret_user:
    call: googleapis.secretmanager.v1.projects.secrets.versions.accessString
    args:
      secret_id: basicAuthUser
    result: secret_user- get_secret_password:
    call: googleapis.secretmanager.v1.projects.secrets.versions.accessString
    args:
      secret_id: basicAuthPassword
    result: secret_password
```

用户登录名和密码现在存储在变量中，我可以在工作流中重用这些变量。我将创建 Base64 编码的 user:password 字符串，该字符串需要传入授权头:

```
- assign_user_password:
    assign:
    - encodedUserPassword: ${base64.encode(text.encode(secret_user + ":" + secret_password))}
```

有了我编码的 user:password 字符串，我现在可以通过添加带有基本身份验证的授权头来调用我的 API(这里是一个云函数)(并返回函数的输出):

```
- call_function:
    call: http.get
    args:
        url: [https://example.com/api](https://europe-west1-workflows-days.cloudfunctions.net/basicAuthFn)
        headers:
            Authorization: ${"Basic " + encodedUserPassword}
    result: fn_output- return_result:
    return: ${fn_output.body}
```

Workflows 具有内置的 [OAuth2 和 OIDC 支持，用于对 Google 托管的 API、函数和云运行服务进行身份验证](https://cloud.google.com/workflows/docs/authentication#making_authenticated_requests)，但了解如何调用其他经过身份验证的服务也很有用，比如那些需要基本身份验证或其他无记名令牌的服务。

*最初发表于*[*https://glaforge.appspot.com*](https://glaforge.appspot.com/article/using-the-secret-manager-connector-for-workflows-to-call-an-authenticated-service)*。*