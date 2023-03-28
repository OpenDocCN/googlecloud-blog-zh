# 工作流的第 17 天:使用 SendGrid 发送电子邮件

> 原文：<https://medium.com/google-cloud/day-17-with-workflows-sending-an-email-with-sendgrid-ceb0c706ded2?source=collection_archive---------3----------------------->

对于通知目的，尤其是以异步方式，电子邮件是一个很好的解决方案。我想在[谷歌云工作流程](https://cloud.google.com/workflows)中添加一个电子邮件通知步骤。由于 GCP 没有电子邮件服务，我查看了云中可用的各种电子邮件服务:SendGrid、Mailgun、Mailjet，甚至快速运行了一个 Twitter，以查看野外的人们在使用什么。我用 SendGrid 做了实验，这个过程非常简单，因为我可以通过创建一个 API 密匙，用 cURL 命令发送我的第一封电子邮件，快速地[开始](https://docs.sendgrid.com/for-developers/sending-email/api-getting-started)。

现在，我需要从我的工作流定义中调用这个 API。这其实也很简单。让我们看看实际情况:

```
- retrieve_api_key:
    assign:
        - SENDGRID_API_KEY: "MY_API_KEY"
- send_email:
    call: http.post
    args:
        url: https://api.sendgrid.com/v3/mail/send
        headers:
            Content-Type: "application/json"
            Authorization: ${"Bearer " + SENDGRID_API_KEY}
        body:
            personalizations:
                - to:
                    - email: to@example.com
            from:
                email: from@example.com
            subject: Sending an email from Workflows
            content:
                - type: text/plain
                  value: Here's the body of my email
    result: email_result
- return_result:
    return: ${email_result.body}
```

在 **retrieve_api_key** 步骤中，我简单地硬编码了 SendGrid API 键。然而，您当然可以将该秘密存储在 Secret Manager 中，然后通过 Workflows Secret Manager 连接器获取密钥(这可能值得专门撰写一篇文章！)

然后，在 **send_email** 步骤中，我准备向 SendGrid API 端点发送 HTTP POST 请求。我使用 SendGrid API 键指定内容类型，当然还有授权。接下来，我准备请求的正文，描述我的电子邮件，带有一个我在 SendGrid 中定义的注册电子邮件用户的 **from** 字段，一个对应于收件人的 **to** 字段，一个电子邮件**主题**和**正文**(这里只是纯文本)。差不多就是这样了！我只是将 SendGrid 文档中的 [cURL 示例](https://app.sendgrid.com/guide/integrate/langs/curl)中发送的 JSON 主体翻译成了 YAML(使用一个方便的 JSON 到 YAML 的转换)

*原载于 https://glaforge.appspot.com*[](https://glaforge.appspot.com/article/sending-an-email-with-sendgrid-from-workflows)**。**