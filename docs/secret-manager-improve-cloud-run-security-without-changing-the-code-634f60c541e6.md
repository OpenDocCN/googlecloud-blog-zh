# Secret Manager:在不改变代码的情况下提高云运行安全性

> 原文：<https://medium.com/google-cloud/secret-manager-improve-cloud-run-security-without-changing-the-code-634f60c541e6?source=collection_archive---------0----------------------->

![](img/197708b3212d7f601e4dd2fa0a3ea663.png)

数据库凭证、API 密钥或私钥(例如 TLS 证书)是大多数应用程序工作所需的数据类型。这些数据是秘密的，知道的人越少，你的秘密就越大！

然而，我看到了，而且我继续看到太多次**秘密在存储库或部署脚本**中的纯文本。

> 为什么开发商不保密？

遗憾的是，**在**[**Secret Manager**](https://cloud.google.com/secret-manager/docs)**发布之前，没有用于安全存储机密**的托管解决方案，它们以纯文本形式存储在部署脚本中，因此部署的云运行服务也是如此。
直到今天，这种不良做法仍被开发商保留着。**团队通常知道这很糟糕，但是他们不知道(也许仍然不知道)如何做得更好。**

## 环境变量用法

因为你有几个环境，**秘密的值根据环境的不同而不同**。*事实上，您不会将 Prod 凭证用于您的开发数据库。*

每个开发人员都知道这些值必须是动态的，而不是硬编码的(*我希望如此！).* **最好的办法就是使用环境变量。秘密以明文的形式放在里面。**

# 秘密经理保守你的秘密，秘密

2019 年 12 月， **Secret Manager 已经公测发布，4 个月后 GA。** Secret Manager 允许您存储多个版本的密码，只有拥有正确 IAM 角色的帐户才能访问或更新它们。

本质是:**你从来没有在环境变量中设置纯文本值，只有一个秘密的引用，你在运行时代码加载它！**

您可以使用`gcloud`命令来管理秘密，或者您可以在代码中使用客户端库进行编程。

# 秘密管理器和遗留代码

Secret Manager 是用于任何新开发的解决方案。然而，对于现有的应用程序，您必须更新代码才能使用 Secret Manager。这可能是一个问题。

此外，一些应用程序在启动时依赖于环境变量，而**添加解密步骤是不可能的，否则会增加巨大的开销**，尤其是在遗留应用程序中。

# 透明使用云运行

云运行和容器的强大之处在于能够定制运行环境。

想法是在启动应用程序之前，检索秘密并**将秘密值加载到环境变量中。我创建了一个脚本来执行这个**

## 要求

在深入脚本之前，**在环境变量值中设置的现有秘密必须被创建到秘密管理器中**。为此，您可以使用`gcloud`命令

```
echo "my super secret" | gcloud beta secrets create \
  --data-file=- --replication-policy=automatic my-secret
```

*注:* `*--data-file*` *中的破折号*`*-*`*param 得到前面命令的结果，这里的* `*echo*` *为秘密*

然后，您必须用这种模式更改环境变量值

```
<prefix>[<procjectID>]/<secretName>[#<version>]
```

*   `<prefix>`将环境变量标记为必须恢复到秘密管理器中。*默认情况下，脚本使用值*
*   `<projectID>`是可选的。它定义了秘密存储在哪个项目中。如果缺少，则使用当前项目。*斜线* `*/*` *始终是必需的*
*   `<secretName>`是秘密进入秘密管理者的名字
*   `#<version>`是可选的。它将秘密的版本定义到秘密管理器中。如果丢失，将恢复最新版本。

*在我们的测试中，我们将使用这个值* `*secret:/my_secret#1*`

## 访问机密管理器

Cloud Run 使用的服务帐户必须能够访问 secrets into Secret Manager。我建议您**为每个云运行服务**创建一个特定的服务帐户。服务帐户将能够访问机密信息，**拥有细粒度授权是最佳实践**。

让我们首先创建一个服务帐户

```
gcloud iam service-accounts create cr-access-secret
```

然后，您可以在项目范围内授予服务帐户访问机密的权限。

```
gcloud projects add-iam-policy-binding \
--member=serviceAccount:cr-access-secret@<PROJECT_ID>.iam.gserviceaccount.com \
--role=roles/secretmanager.secretAccessor <PROJECT_ID>
```

***用您当前的项目 ID* 替换 `*<PROJECT_ID>*`**

***或者你可以**只授予服务帐号一个特定的秘密**。有几个秘密的话就是重复动作，但是安全性更高。***

```
*gcloud beta secrets add-iam-policy-binding  \
--member=serviceAccount:cr-access-secret@<PROJECT_ID>.iam.gserviceaccount.com \
--role=roles/secretmanager.secretAccessor my-secret*
```

****用您当前的项目 ID* 替换 `*<PROJECT_ID>*`***

## ****仔细看看`Dockerfile`****

****现在，测试的环境已经设置好了。如果你看一下`[Dockerfile](https://github.com/guillaumeblaquiere/secret-loader-medium/blob/master/Dockerfile)`，你会看到这最后两行****

```
**RUN wget https://storage.googleapis.com/secret-loader/start.sh \
     && chmod +x /start.sh
CMD ["/start.sh", "/server"]**
```

****我简单地下载了一个 **bash 脚本，并在上面添加了执行权限**。然后我启动这个脚本，*只有一个参数* : **我想在秘密加载后运行什么。******

## ****秘密装载****

****所有的过程都由`[start.sh](https://github.com/guillaumeblaquiere/secret-loader-medium/blob/master/start.sh)`脚本文件执行。让我们深入了解一下。****

****我从**开始定义** `**<prefix>**`默认值。*想改就改！*****

```
**secret_prefix="secret:"**
```

****然后，处理就很简单了****

*   ****扫描所有包含 T4 的**环境变量，提取每个变量的`key`和`value`******

```
**for i in $(printenv | grep ${secret_prefix})
do
  key=$(echo ${i} | cut -d'=' -f 1)
  val=$(echo ${i} | cut -d'=' -f 2-)**
```

*   ****然后我检查值是否以`<prefix>`开始。如果是这样，我**去掉**。*事实上，另一个环境变量可以匹配* `*grep*` *，但是没有前缀。*****

```
**if [[ ${val} == ${secret_prefix}* ]]
then
  val=$(echo ${val} | sed -e "s/${secret_prefix}//g")**
```

*   ****提取项目，如果不为空，**为** `**gcloud**` **命令**准备参数值****

```
**projectId=$(echo ${val} | cut -d'/' -f 1)
secret=$(echo ${val} | cut -d'/' -f 2)

if [[ -n ${projectId} ]]
then
  project="--project=${projectId}"
fi**
```

*   ******提取秘密的名称及其版本**。如果缺少版本，则使用值`latest`****

```
**secretName=$(echo ${secret} | cut -d'#' -f 1)
version="latest"
if [[ ${val} == *#* ]]
then
  version=$(echo ${val} | cut -d'#' -f 2)
fi**
```

*   ****最后，获取秘密并**将其加载到名为** `**key**` **的环境变量中******

```
**plain="$(gcloud beta secrets versions access --secret=${secretName} ${version} ${project})"
#For multiline management
export $key="$(echo $plain | sed -e 's/\n//g')"**
```

****仅此而已。在对环境变量**进行循环之后，我调用了脚本中唯一的一个参数。******

```
**#run the following command
${1}**
```

*****这是一个唯一的参数。在* `*Dockerfile*` *中设置* ***将整个命令及其参数放入同一个字符串*******

## ****构建容器并部署云运行服务****

****为了测试秘密的正确加载，我写了一个非常简单的 **Go 服务器，当你调用它时，它会显示所有的环境变量**。构建容器，例如使用云构建****

```
**gcloud builds submit -t gcr.io/<PROJECT_ID>/secret-loader**
```

*****用您当前的项目 ID* 替换 `*<PROJECT_ID>*`****

*****并使用 **secret 环境变量和授权服务帐户**部署服务*****

```
***gcloud run deploy --image=gcr.io/<PROJECT_ID>/secret-loader \
--platform=managed --region=us-central1 --allow-unauthenticated \
--service-account=cr-access-secret@<PROJECT_ID>.iam.gserviceaccount.com \
--set-env-vars=super_secret=secret:/my-secret#1 secret-loader***
```

******用您当前的项目 ID 替换* `*<PROJECT_ID>*` *。为了更简单的测试，我允许未经认证的用户。不强制，做最适合自己的！******

****最后，通过**点击部署后提供的 URL** 或者通过命令行测试您的部署****

```
**curl https://secret-loader.<project-hash>-uc.a.run.app**
```

# ****透明装载****

****现在你有了。通过使用这个**预启动脚本，秘密被加载到您的环境变量**中。您的遗留应用程序不需要更新，只有 Docker 文件发生了变化，最好的情况下只有 2 行。****

****用法**不限于云运行和容器**。您也可以在计算引擎中使用这个预启动脚本**。******

****重构、复杂性、延迟将不再是让应用程序不安全的借口。保守秘密是最重要的！****

****请根据您的要求随时更新、改编、更改脚本。我很乐意在评论中了解你的用法和更新。****

******2020 年 4 月 18 日更新******

# ****春云 GCP 解决方案****

****对于那些使用 Spring Cloud 的用户，有一个内置的解决方案。**[**spring-Cloud-GCP 项目**](https://cloud.spring.io/spring-cloud-static/spring-cloud-gcp/1.2.2.RELEASE/reference/html/) **为无缝集成谷歌云产品提供了一个大型库。********

******其中一个是专用于秘密管理器的，它允许你简单地通过 3 个简单的步骤的配置来轻松**加载你的秘密********

*   ******将依赖项添加到项目中******

*******带 Maven 的*******

```
****<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-gcp-starter-secretmanager</artifactId>
</dependency>****
```

*******带梯度*******

```
****dependencies {   compile group: 'org.springframework.cloud', name: 'spring-cloud-gcp-starter-secretmanager' }****
```

*   ******在`bootstrap.properties`文件中激活启动引导，并为你的密码定义一个前缀*(推荐)*******

```
****# Enable the bootstrap at startup
spring.cloud.gcp.secretmanager.bootstrap.enabled=true

# Optional prefix of the secret name
spring.cloud.gcp.secretmanager.secret-name-prefix=secrets.****
```

*   ******将您的秘密直接定义到`properties`文件中，无需更改代码******

```
****my_super_secret=${secrets.my_secret}****
```

******`*my_secret*` *是秘密管理器中的秘密名称，带有已定义的前缀，* `*my_super_secret*` *是变量的名称，您可以像这样在代码中注入:*******

```
****@Value(${my_super_secret})
private String mySuperSecret****
```

*******如果你喜欢更新你的代码，或者用这个库编码，你可以像这样直接使用 Secret(****我不推荐这样，你直接在你的代码*** *)*******

```
****@Value(${secrets.my_secret})
private String mySuperSecret****
```

********仅此而已，不需要任何代码**，只需要在属性文件中进行配置和变量重定义！[示例文件中的更多示例](https://github.com/spring-cloud/spring-cloud-gcp/tree/master/spring-cloud-gcp-samples/spring-cloud-gcp-secretmanager-sample)******