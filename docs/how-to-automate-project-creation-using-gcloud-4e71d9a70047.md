# 如何使用 gcloud 自动创建项目

> 原文：<https://medium.com/google-cloud/how-to-automate-project-creation-using-gcloud-4e71d9a70047?source=collection_archive---------0----------------------->

我们将为大约 30 人举办 Cloud ML 培训课程，并希望给他们每个人一个原始的谷歌云项目。还有，我想让账单来找我。显然，我不想在项目创建过程中点击鼠标，所以我编写了脚本。

如果你需要为一群人创建单独的谷歌云项目，请随意使用我的[脚本](https://github.com/GoogleCloudPlatform/training-data-analyst/tree/master/blogs/gcloudprojects)。即使你没有这种特殊的需求，本文也展示了如何使用 [**gcloud**](https://cloud.google.com/sdk/gcloud/) 来自动化重复的任务。这是一项需要掌握的有用技能。

# 如何运行脚本

首先，您需要找到您的帐单帐户 id。打开[云壳](https://cloud.google.com/shell/)运行:

```
gcloud alpha billing accounts list
```

记帐帐户将采用 0x 0x 0x 0 x-0x 0x 0x 0 x-0x 0x 0x 0 x 的形式。然后，通过运行以下命令创建项目:

```
curl [https://raw.githubusercontent.com/GoogleCloudPlatform/training-data-analyst/master/blogs/gcloudprojects/create_projects.sh](https://raw.githubusercontent.com/GoogleCloudPlatform/training-data-analyst/master/blogs/gcloudprojects/create_projects.sh) -o create_projects.shchmod +x create_projects.sh./create_projects.sh 0X0X0X-0X0X0X-0X0X0X learnml-20170106  somebody@gmail.com someother@gmail.com
```

create_projects.sh 的第一个参数是账单 id。第二个是项目前缀——所有创建的项目都有这个前缀。因为项目 id 必须是唯一的，所以选择一个相对唯一的前缀。然后，只需提供电子邮件地址列表。

# 整个剧本

这就是 create _ projects . sh[[github link](https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/blogs/gcloudprojects/create_projects.sh)]的样子:

```
#!/bin/bashif [ "$#" -lt 3 ]; then
   echo "Usage:  ./create_projects.sh billingid project-prefix  email1 [email2 [email3 ...]]]"
   echo "   eg:  ./create_projects.sh 0X0X0X-0X0X0X-0X0X0X learnml-20170106  [somebody@gmail.com](mailto:somebody@gmail.com) [someother@gmail.com](mailto:someother@gmail.com)"
   exit
fiACCOUNT_ID=$1
shift
PROJECT_PREFIX=$1
shift
EMAILS=$@gcloud components update
gcloud components install alphafor EMAIL in $EMAILS; do
   PROJECT_ID=$(echo "${PROJECT_PREFIX}-${EMAIL}" | sed 's/@/-/g' | sed 's/\./-/g' | cut -c 1-30)
   echo "Creating project $PROJECT_ID for $EMAIL ... " # Create project
   gcloud alpha projects create $PROJECT_ID # Add user to project
   gcloud alpha projects get-iam-policy $PROJECT_ID --format=json > iam.json.orig
   cat iam.json.orig | sed s'/"bindings": \[/"bindings": \[ \{"members": \["user:'$EMAIL'"\],"role": "roles\/editor"\},/g' > iam.json.new
   gcloud alpha projects set-iam-policy $PROJECT_ID iam.json.new # Set billing id of project
   gcloud alpha billing accounts projects link $PROJECT_ID --account-id=$ACCOUNT_IDdone
```

让我们一块一块来看。前几行只是确保用户已经传入了所需的参数(账单 id、项目前缀和电子邮件地址列表)。

# 1.设置

这些是使用普通 bash 语法从命令行参数中提取的(shift 使用第一个参数，$@是现在保留的命令行参数数组):

```
ACCOUNT_ID=$1
shift
PROJECT_PREFIX=$1
shift
EMAILS=$@
```

我接下来更新 gcloud 并安装 alpha 模块，因为项目创建功能目前在 public alpha:

```
gcloud components update
gcloud components install alpha
```

# 2.唯一的项目 id

接下来，我一封一封地浏览邮件，并为每封邮件创建项目:

```
for EMAIL in $EMAILS; do
   PROJECT_ID=$(echo "${PROJECT_PREFIX}-${EMAIL}" | sed 's/@/-/g' | sed 's/\./-/g' | cut -c 1-30)
   echo "Creating project $PROJECT_ID for $EMAIL ... " ...done
```

我使用项目前缀和电子邮件地址为每个用户提供一个单独的项目 id。因为项目 id 不能包含@符号或点，所以我用连字符替换了它们，并且因为项目 id 被限制在 30 个字符以内(不要问我为什么)，所以我在 30 个字符的限制内去掉了名称。

# 3.创建项目

现在，我们准备创建项目本身。为了创建一个项目，我使用 gcloud 命令:

```
gcloud alpha projects create $PROJECT_ID
```

# 4.添加用户

现在，这个项目已经由我创建，并以我为所有者。我需要添加我的培训课程中作为编辑的人的电子邮件地址(这将让他们做大多数事情，但我仍然是所有者):

```
gcloud alpha projects get-iam-policy $PROJECT_ID --format=json > iam.json.orig

cat iam.json.orig | sed s'/"bindings": \[/"bindings": \[ \{"members": \["user:'$EMAIL'"\],"role": "roles\/editor"\},/g' > iam.json.new

gcloud alpha projects set-iam-policy $PROJECT_ID iam.json.new
```

上面的代码获取项目的当前身份访问管理(IAM)策略，可能如下所示:

```
{
  "bindings": [
    {
      "members": [
        "user:[myemail@google.com](mailto:vlakshmanan@google.com)"
      ],
      "role": "roles/owner"
    }
  ],
  "etag": "BwVFSYizPdc=",
  "version": 1
}
```

然后，我使用 sed 替换 bindings 行，以便新用户作为编辑者被添加:

```
"bindings": [ {"members": ["user:somebody2@gmail.com"],"role": "roles/editor"},
```

# 5.演员表

最后，我将新创建的项目与我的账单 id 关联起来，这样账单就会出现在我面前:

```
gcloud alpha billing accounts projects link $PROJECT_ID --account-id=$ACCOUNT_ID
```

就是这样！

# 清理

培训结束后，我将使用 delete _ projects . sh[[github link](https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/blogs/gcloudprojects/delete_projects.sh)]循环删除这些项目，其关键行是:

```
for EMAIL in $EMAILS; do
    PROJECT_ID=$(echo "${PROJECT_PREFIX}-${EMAIL}" | sed 's/@/-/g' | sed 's/\./-/g' | cut -c 1-30) gcloud alpha projects delete $PROJECT_IDdone
```

**注意:**只为你完全信任的一群人提供这样的项目，比如你公司内部的人，因为他们所做的一切都要向你收费…