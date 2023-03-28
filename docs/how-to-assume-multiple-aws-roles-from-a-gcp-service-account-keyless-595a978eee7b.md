# 如何从 GCP 无钥匙服务帐户承担多个 AWS 角色

> 原文：<https://medium.com/google-cloud/how-to-assume-multiple-aws-roles-from-a-gcp-service-account-keyless-595a978eee7b?source=collection_archive---------1----------------------->

包括了安全最佳实践“不要带备用密钥”

![](img/2814bc737bc1c7ee5906bc4955f9988e.png)

# TL；速度三角形定位法(dead reckoning)

本文展示了如何将一个 GCP 服务帐户集成到多个 AWS 角色，而无需在任何地方存储 sa 密钥。

# 开始

从现在起的几个月甚至未来几年，您将需要连接来自不同云提供商的不同基础架构组件。几周前，我遇到了一个客户的问题，他想将一些数据从 S3 存储桶转移到 GCP 一个测试组织内新创建的 GCS 存储桶。

没有比这更容易的了，这是我最初的想法。

即使这看起来是一个容易实现的任务，但构建一个**安全的工作流**也不是一件小事。我对此提出的要求是，我不想负责在某个地方存储(然后维护)一些访问/秘密密钥，也不想创建 AWS 帐户和角色。

**换句话说，我希望能够将文件从 S3 传输到 GCS 存储桶，而无需将 AWS 凭证存储在 AWS 之外的某个地方。**

我应该如何进行？我当然有不同的选择，实际上有不同的方法来存储和处理密钥，但我的目标是在两个提供者之间有一个总的区别，从第一个(AWS)不应该被存储到目的地之一，GCP。

我相信，在一个可信实体的基础上创建 AWS 角色的可能性这是一个解决问题的好方法，与网络身份提供商(在我们的情况下是谷歌)相关的可信身份无疑是一条出路。

工作流程如下所示:

> 1.创建一个 GCP 服务帐户，并获取唯一的 ID。
> 
> 2.在现有的 AWS 帐户中创建一个 AWS 角色，并将受信任的实体类型设置为 **Web Identity** ，选择 Google 作为提供者，并将 GCP SA 唯一 ID 粘贴到“受众”文本框中。
> 
> 2.分配所需的权限以完成创建。

# 由于帐户连接，我们现在能够使用 GCP SA 来请求临时 AWS 凭证，并使用这些凭证来承担 AWS 角色。

Avi Keinan 写了一篇关于[在不使用 IAM 键](https://www.doit-intl.com/assume-an-aws-role-from-a-google-cloud-without-using-iam-keys/)的情况下承担来自 Google Cloud 的 AWS 角色的精彩帖子，感谢 Avi 的灵感！

现在，我们有一个与 GCP SA **相关的 AWS 角色。**接下来是什么？

# 真实案例

**回到使用案例，有一件事我之前没有提到，在我面对的真实场景中，AWS 角色实际上是两个，而不是一个。**

第一个 AWS 角色应该用于承担第二个 AWS 角色的身份，第二个 AWS 角色具有访问 S3 存储桶的特定权限。

因此，我们需要从 GCP 服务帐户承担一个 AWS 角色，然后使用第一个 AWS 角色临时凭证承担第二个角色，这是唯一一个有权限访问 S3 存储桶的角色。棘手但也是一个有趣的挑战。

我写了一个简单的 Golang 应用程序，可以通过[Google professional services github repository](https://github.com/GoogleCloudPlatform/professional-services)获得，它将为我们提供两个 AWS 角色，一个是作为来自 GCP 服务帐户 的 [**AWS 角色之间连接的一部分，另一个是执行最终任务。**](https://github.com/GoogleCloudPlatform/professional-services/tree/main/tools/gcp-sa-to-assume-aws-arn-role)

```
//getAWSWebIdentityServiceCreds function to assume the Service Role once the identity creds have been retrieved
func getAWSWebIdentityServiceCreds(webIDcreds *sts.Credentials, role string) (*sts.AssumeRoleOutput, error) {accessKey := *webIDcreds.AccessKeyId
 secretKey := *webIDcreds.SecretAccessKey
 accessToken := *webIDcreds.SessionToken
 sess, err := session.NewSession(&aws.Config{
  Credentials: credentials.NewStaticCredentials(accessKey, secretKey, accessToken),
 })
 if err != nil {
  fmt.Println("NewSession Error", err)
  return nil, err
 }
 svc := sts.New(sess)
 roleToAssumeArn := role
 sessionName := "gcp_session"
 result, err := svc.AssumeRole(&sts.AssumeRoleInput{
  RoleArn:         &roleToAssumeArn,
  RoleSessionName: &sessionName,
 })
 if err != nil {
  fmt.Println("AssumeRole Error", err)
  return nil, err
 }
 return result, nil
}
```

即使深入研究代码可能很有趣，并且是一个试验一些 Go 代码甚至改进它的机会，因为它肯定不是完美的，它已经准备好被使用了。

gcp-auth 文件夹下的代码一旦编译，将接受两个参数作为输入，一个身份 ARN 角色和一个服务 ARN 角色。应该从 GCP 服务帐户使用身份 ARN 来对 AWS IAM APIs 进行身份验证，而一旦采用了身份 ARN，服务 ARN 就会用来对 S3 进行 API 调用。

为了完成这个过程，应该将一个默认的 AWS 凭证(如下所示)文件放在将使用二进制文件调用 AWS 的对象(VM、容器等)中:

```
[default]
credential_process = /usr/local/bin/gcp-auth --env=true
```

如示例所示，输入 ARNs 也可以作为环境变量提供，这将确保最终设置期间的更大安全性。

```
export GCP_AUTH_IDENTITY=arn:aws:iam::000000000:role/external-gcp-auth
export GCP_AUTH_SERVICE=arn:aws:iam::000000000:role/access-to-s3-from-gcp
```

正如我们在本文中看到的，请记住还有最后一步要做，将我们的 GCP 服务帐户连接到 AWS S3 ARN。

在创建 AWS ARN 的过程中，可以将其指定为可信身份，选择 Web 身份，然后选择 Google。通过将 GCP SA 唯一 ID 粘贴到观众框中，最后一步就完成了。

希望你喜欢这篇文章，我真的期待听到你的反馈！