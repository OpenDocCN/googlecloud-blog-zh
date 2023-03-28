# 从云构建中访问 Google Workspace 文档

> 原文：<https://medium.com/google-cloud/access-to-google-workspace-documents-from-cloud-build-3001c2883b66?source=collection_archive---------2----------------------->

![](img/f91489e9ebe94cab085ceb1a7af15110.png)

现代代码开发**基于自动化**，尤其是 CI/CD 管道(持续集成持续部署/交付)。在 Google Cloud 上，[**Cloud Build**](https://cloud.google.com/build)**是实现 CI/CD 的无服务器解决方案**，提供了许多特性和高度可定制的管道。

在代码打包期间，您可能需要**依赖于其他数据，而不仅仅是您的代码**，用于配置、丰富、定制、版本控制……新的数据源可以在数据库中，也可以在**Google Workspace 文档中**。

> 从 Cloud Build 访问 Google Workspace 文档并不容易！

# 访问 Google Workspace 文档

在尝试访问文档之前，让我们先编写一个小的 python 脚本**来读取 Google Sheet 文档**

```
import google.auth
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError

# The ID and range of a sample spreadsheet.
SAMPLE_SPREADSHEET_ID = 'YOUR_SHEET_ID'
SAMPLE_RANGE_NAME = 'DATA!A2:E' #Example

def main():
    *"""Shows basic usage of the Sheets API.
    Prints values from a sample spreadsheet.
    """* creds, _ = google.auth.default()
    try:
        service = build('sheets', 'v4', credentials=creds)
        # Call the Sheets API
        sheet = service.spreadsheets()
        result = sheet.values().get(spreadsheetId=SAMPLE_SPREADSHEET_ID,
                                    range=SAMPLE_RANGE_NAME).execute()
        values = result.get('values', [])

        if not values:
            print('No data found.')
            return

        print('Name, Major:')
        for row in values:
            # Print columns A and E
            print('%s, %s' % (row[0], row[4]))
    except HttpError as err:
        print(err)

if __name__ == '__main__':
    main()
```

安装依赖项(文件`requirements.txt`

```
google-api-python-client
google-auth# Install dependencies with: pip3 install -r requirements.txt
```

您可以进行第一次尝试。

```
python3 main.py
```

失败……`Insufficient scopes`。默认认证有**问题。**

当您在您的环境中使用命令`gcloud auth application-default login`、**验证自己时，您使用默认的作用域**，它不包括 Google Sheet 作用域。让我们来解决这个问题

要确定个人账户的范围，你只需**明确定义范围**和[添加你想要的内容](https://developers.google.com/identity/protocols/oauth2/scopes)以及谷歌云平台的内容。

```
gcloud auth application-default login \
--scopes='https://www.googleapis.com/auth/spreadsheets.readonly',\
https://www.googleapis.com/auth/cloud-platform'
```

并再次尝试您的代码。**这次成功了！！**

> 完美！！让我们在云构建中使用这个脚本！

# 天真的第一次尝试

要在云构建中运行 Python 脚本，您必须编写一个描述操作的`cloudbuild.yaml`文件。我们将使用 Python 图像的单个步骤。

```
steps:
  - name: 'python:slim-buster'
    script: |
      pip3 install -r requirements.txt
      python3 main.py
```

并运行构建；

```
gcloud builds submit
```

**哦不，** `Insufficient scopes`！！但是这一次，**你不能作用于当前账号**，因为它是元数据服务器提供的[。](https://cloud.google.com/compute/docs/metadata/overview)

让我们将代码更新为**以编程方式强制该范围**
*只更新* `*creds*` *变量定义。*

```
SCOPES = ['https://www.googleapis.com/auth/spreadsheets.readonly']
creds, _ = google.auth.default(scopes=SCOPES)
```

再试一次。**不是更好……**

> 无法确定默认服务帐户的范围。我们试试别的吧！

# 客户服务帐户尝试

对于一些服务，比如 Compute Engine，你不能在运行时改变默认服务账户**的作用域**。

*   你必须停止虚拟机， [**改变作用域**](https://cloud.google.com/compute/docs/access/service-accounts#accesscopesiam) **并重启它**。*太无聊了*

或者，这是最有趣也是我最雄心勃勃的选择

*   您可以 [**使用客户管理服务账户**](https://cloud.google.com/compute/docs/access/service-accounts#associating_a_service_account_to_an_instance) 。然后，您可以在运行时随意确定服务帐户的范围！

所以，让我们 [**使用一个带有云构建的客户管理服务账户**](https://cloud.google.com/build/docs/securing-builds/configure-user-specified-service-accounts) 。您必须创建一个服务帐户，并授予它正确的权限。

```
#Create the service account
gcloud iam service-accounts create custom-sa#Grant the permission 
gcloud projects add-iam-policy-binding \
--member="serviceAccount:custom-sa@**PROJECT_ID**.iam.gserviceaccount.com" \
--role="roles/storage.objectViewer" **PROJECT_ID**gcloud projects add-iam-policy-binding \
--member="serviceAccount:custom-sa@**PROJECT_ID**.iam.gserviceaccount.com" \
--role="roles/logging.logWriter" **PROJECT_ID**
```

*用自己的项目 ID* 替换 `*PROJECT_ID*`

*并像这样更新您的`cloudbuild.yaml`文件*

```
*steps:
  - name: 'python:slim-buster'
    script: |
        pip3 install -r requirements.txt
        python3 main.py
serviceAccount: 'projects/**PROJECT_ID**/serviceAccounts/custom-sa@**PROJECT_ID**.iam.gserviceaccount.com'
options:
  logging: CLOUD_LOGGING_ONLY*
```

*最后，做最后一次尝试……再失败一次 `Insufficient scopes`。 ***为什么？？****

> *在云构建中无法更改运行时服务帐户的范围。下一个选择是什么？*

# *模拟解决方案*

*因此，**元数据服务器无法生成具有不同作用域**的令牌。*这个问题与我们一开始遇到的用户帐户问题类似*。*无法在运行时更改令牌范围。
解决方案是从头开始重新生成具有正确作用域的令牌。**

*因此，下一个选项是做同样的事情，在令牌创建时，**在服务帐户**上请求具有正确范围的新令牌**。***

*为此，我们可以使用**一个名为** [**的特性来模仿**](https://cloud.google.com/iam/docs/impersonating-service-accounts) 。原理是**代表另一个账户生成一个令牌**(访问令牌或身份令牌)**。**
*并且，顺便继承它的所有权限**

*在代码中，你必须改变一些东西:*

*   *您必须**知道服务帐户电子邮件来模拟** *(出于生产目的，使用环境变量来提供)**
*   *您必须**创建模拟凭证**并使用它*

```
***import google.auth.impersonated_credentials**

from googleapiclient.discovery import build
from googleapiclient.errors import HttpError

# The ID and range of a sample spreadsheet.
SAMPLE_SPREADSHEET_ID = 'YOUR_SHEET_ID'
SAMPLE_RANGE_NAME = 'DATA!A2:E'

def main():
    *"""Shows basic usage of the Sheets API.
    Prints values from a sample spreadsheet.
    """* SCOPES = ['https://www.googleapis.com/auth/spreadsheets.readonly']**creds, _ = google.auth.default(scopes="https://www.googleapis.com/auth/cloud-platform")
    icreds = google.auth.impersonated_credentials.Credentials(
        source_credentials=creds,
        target_principal=custom-sa@PROJECT_ID.iam.gserviceaccount.com,
        target_scopes=SCOPES,
    )

**    try:
        service = build('sheets', 'v4', credentials=**icreds**)

        # Call the Sheets API
        sheet = service.spreadsheets()
        result = sheet.values().get(spreadsheetId=SAMPLE_SPREADSHEET_ID,
                                    range=SAMPLE_RANGE_NAME).execute()
        values = result.get('values', [])

        if not values:
            print('No data found.')
            return

        print('Name, Major:')
        for row in values:
            # Print columns A and E, which correspond to indices 0 and 4.
            print('%s, %s' % (row[0], row[4]))
    except HttpError as err:
        print(err)

if __name__ == '__main__':
    main()*
```

*为了允许云构建运行时服务帐户模拟目标服务帐户，必须允许**运行时服务帐户在目标服务帐户**上创建令牌。*

*因此，您必须像这样在运行时服务帐户上授予角色`Service Account Token Creator`*

```
*gcloud projects add-iam-policy-binding \
--member="serviceAccount:custom-sa@**PROJECT_ID**.iam.gserviceaccount.com" \
--role="roles/iam.serviceAccountTokenCreator" **PROJECT_ID***
```

*而且试一试… **又失败了！！***

*但是好消息是**这不是同一个错误！**这是来自 Google Sheet 的错误，因为目标服务帐户没有访问它的权限！*

***精彩**！！转到您的工作表文档，单击“共享”按钮，并将目标服务帐户电子邮件添加为工作表文档的查看者。*

*再试一次，这次**成功了！！***

****注:*** *不再需要客户管理服务账户。您可以切换回云构建默认服务帐户；不要忘记授予它正确的角色，让它模拟服务帐户。**

****注意 2:*** *不能冒充云构建默认服务账号本身。您只能模拟客户管理的服务帐户(您创建并附加到您的项目的服务帐户)**

# *不一致的安全解决方案*

*安全性是最重要的，谷歌云提供了非常强大和通用的安全选项。*

*然而，**这些选项从一个服务到另一个服务是不一致的**，并且需要专业知识、失败和以优雅的方式成功。
**因为那些困难**，很多用户不花时间，快速去找最快最丑的解决方案:他们用一个**服务账号密钥文件**(这是安全的反模式)！*

*我希望这篇文章能帮助你成功而优雅地管理你的安全问题，并在任何情况下保持安全。*