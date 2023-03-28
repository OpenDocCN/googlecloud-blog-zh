# 使用 Python 将 GCP 项目导入您的组织

> 原文：<https://medium.com/google-cloud/importing-gcp-projects-into-your-organization-with-python-aa9627bd8f12?source=collection_archive---------1----------------------->

# 问题:“遗留项目”(组织前)

*注:截止到*[*2021 年 2 月 26 日*](https://cloud.google.com/resource-manager/docs/release-notes#February_26_2021) *，Google 已经为* [*组织间迁移项目*](https://cloud.google.com/resource-manager/docs/project-migration) *做了一个自助服务流程，可供公众预览。*

如果你使用 GCP 与 G 套件(包括[云身份](https://support.google.com/a/answer/7319251?hl=en))帐户，你已经熟悉[如何使用组织](https://cloud.google.com/resource-manager/docs/quickstart-organizations)更容易[集中管理你所有的谷歌云资源](https://cloudplatform.googleblog.com/2017/01/centrally-manage-all-your-Google-Cloud-resources-with-Cloud-Resource-Manager.html)。

然而，如果你在[组织资源](https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy#organization)变得[普遍可用](https://cloudplatform.googleblog.com/2016/10/whats-new-with-Google-Cloud-Resource-Manager-and-other-IAM-news.html)之前就开始使用 GCP，你可能有很多项目是在你的域的组织资源存在之前由你的 G Suite 用户**创建的。这些项目目前不在你的控制之下。**

您的用户可以选择加入[将现有项目迁移到组织](https://cloud.google.com/resource-manager/docs/migrating-projects-billing)中，只要在您的组织中授予他们[项目创建者](https://cloud.google.com/iam/docs/understanding-roles#predefined_roles)角色。但是，作为 GCP 组织的管理员，您很难找到所有这些“遗留项目”并将它们“导入”到您的组织资源中。

# 解决方案:用 Python 实现自动化

自动化这个过程(即使对于成千上万的用户或项目)是非常容易的，只要你能在你的 G Suite 域中[创建一个服务帐户](https://developers.google.com/api-client-library/python/auth/service-accounts#creatinganaccount)和[授权的域范围权限](https://developers.google.com/api-client-library/python/auth/service-accounts#delegatingauthority)。*(如果不是您，您可能需要联系您的 G Suite 管理员。)*

## 过程

这是为了发现您的领域的所有遗留项目并将其导入到您的组织资源中而要遵循的一般过程。

1.  使用具有域范围权限的服务帐户和具有超级管理员角色的用户检索 G Suite 域中所有用户的**列表。**
2.  作为域中的每个用户，检索项目及其 IAM 策略绑定的列表**,以查找用户所拥有的没有组织或文件夹作为其父级的项目。**
3.  对于步骤#2 中找到的每个项目(作为拥有项目的用户)**添加服务帐户作为项目的所有者**。
4.  作为服务帐户，**将项目添加到组织资源**。

## 服务帐户设置

为了用 Python 做到这一点，我们需要一个对 G Suite 域和 GCP 组织资源有适当访问权限的[服务帐户](https://developers.google.com/api-client-library/python/auth/service-accounts)。

1.  [创建服务账户](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#creatinganaccount)(选择*启用 G Suite 全域委托*)
2.  保存服务帐户 JSON 文件(例如`serviceaccount.json`)
3.  [委派服务帐户的全域性权限](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#delegatingauthority)使用以下作用域:`https://www.google.apis.com/auth/admin.directory.user, https://www.google.apis.com/auth/cloud-platform`
4.  在 [GCP 控制台](https://console.cloud.google.com/apis/dashboard) : `[Admin SDK](https://console.cloud.google.com/apis/api/admin.googleapis.com/overview), [Cloud Resource Manager API](https://console.cloud.google.com/apis/api/cloudresourcemanager.googleapis.com/overview)`中启用 API
5.  授予服务帐户在组织资源中的**项目创建者**角色。

## 使用 Python 验证

现在我们有了一个拥有所有正确权限的服务帐户，我们可以创建凭证，并且[准备以 G Suite 域中超级管理员角色的用户身份进行授权调用](https://developers.google.com/api-client-library/python/auth/service-accounts#authorizingrequests)。

```
from oauth2client.service_account import ServiceAccountCredentialssuperadmin = '**google@mydomain.com**'scopes = [
    '**https://www.google.apis.com/auth/admin.directory.user**',
    '**https://www.google.apis.com/auth/cloud-platform**'
]

credentials = ServiceAccountCredentials.from_json_keyfile_name(
    '**serviceaccount.json**',
    scopes
)
admin_creds = credentials.create_delegated(superadmin)
```

## 正在检索所有域用户

现在，我们可以使用这些 admin 凭证对 Admin SDK 目录 API 进行身份验证调用，以获得域中所有用户的列表。

*注意:使用* [*Admin SDK 目录 API users.list*](https://developers.google.com/admin-sdk/directory/v1/reference/users/list) *方法。*

```
from apiclient.discovery import build

admin = build('admin', 'directory_v1', credentials=**admin_creds**)params = {
    'customer': 'my_customer',
    'fields': fields,
}users = admin.users()
request = users.list(**params)**domain_users = []**
while request is not None:
    users_list = request.execute()
    **domain_users += users_list.get('users', [])**
    request = users.list_next(request, users_list)
```

## 正在检索所有项目

有了所有域用户的列表，我们现在可以“成为”每个用户来获得他们所有项目的列表。

*注意:使用* [*云资源管理器的*](https://cloud.google.com/resource-manager/reference/rest/v1/projects/list) *方法。*

```
# loop through all of the users in the domain
for user in **domain_users**:
    **email = user['primaryEmail']** # create credentials as the user
    credentials = ServiceAccountCredentials.from_json_keyfile_name(
        '**serviceaccount.json**',
        scopes
    )
    **user_creds** = credentials.create_delegated(**email**) # connect to cloud resource manager as the user
    crm = build(
        'cloudresourcemanager', 
        'v1', 
        credentials=**user_creds** ) # get a list of all projects visible to the user
    projects = crm.projects()
    request = projects.list(**params) **user_projects = []**
    while request is not None:
        projects_list = request.execute()
        **user_projects += projects_list.get('projects', [])**
        request = projects.list_next(request, projects_list)
```

## 查找要导入的用户项目

对于每个用户，一旦我们获得了可见项目的列表，我们需要将列表缩小到只有**活动的**项目，这些项目没有**父**(组织或文件夹)并且由用户拥有**。这将要求我们为每个没有父项目的活动项目获取 **IAM 策略绑定**。**

*注意:使用* [*云资源管理器的*](https://cloud.google.com/resource-manager/reference/rest/v1/projects/getIamPolicy) *端点。*

```
# loop through each of the projects visible to the user
for project in **user_projects**: **# skip projects that are not active**
    if project['lifecycleState'] != 'ACTIVE':
        continue **# skip projects that already have a parent**
    if not project.get('parent', {}):
        continue **# get the IAM policy bindings for the project**
    project_id = p['projectId']
    params = {
        'resource': project_id,
        'body': {},
    }
 **policy = crm.projects().getIamPolicy(**params).execute()**
    bindings = policy.get('bindings', []) owner = False **# find out if the user is an owner of the project**
    for b in bindings: **# skip bindings other than owner**
        if b['role'] != 'roles/owner':
            continue **# see if user is one of the owners**
        if 'user:%s' % (**email**) in b['members']:
            owner = True # if user is owner, add the project to import list
    if owner: # add bindings to the project record
 **project['bindings'] = bindings** * # add the service account as a project owner...* *# add the project to the organization...*
```

## 将服务帐户添加为项目所有者

既然我们知道了哪些项目需要导入到组织资源中，我们仍然需要正确的权限以便能够添加项目。我们再次以用户的身份这样做。

**我们无法通过 API 将超级管理员添加为所有者。**向项目添加所有者会生成一个电子邮件通知，用户必须接受该通知才能被授予角色(这只能从控制台完成)。

如果项目已经在一个组织中，并且被添加为所有者的用户与该组织在同一个域中，则使用 API 添加用户作为所有者仅**是可能的**。 [**服务账号可以不受任何限制直接成为项目业主**](https://google-api-client-libraries.appspot.com/documentation/cloudresourcemanager/v1/python/latest/cloudresourcemanager_v1.projects.html#setIamPolicy) 。

*注意:使用* [*云资源管理器 projects . setiampolicy*](https://cloud.google.com/resource-manager/reference/rest/v1/projects/setIamPolicy)*端点。*

```
**# set service account user name**
user = 'serviceAccount:%s' % (credentials.service_account_email)**newbindings = []**update = False**# check bindings for owner role and update if necessary**
for b in project['bindings']:
    if b['role'] == 'roles/owner' and user not in b['members']:
        b['members'].append(user)
        update = True
    **newbindings.append(b)**# **update the project IAM policy if necessary**
if update:
    body = {'policy': {'bindings': **newbindings**}}
    params = {'resource': project_id, 'body': body}

    # set the IAM policy for the project
    **crm.projects().setIamPolicy(**params).execute()**
```

## 将项目添加到组织资源

既然服务帐户是所有者(并且已经是组织中的项目创建者)，我们可以使用服务帐户的凭据将项目添加到组织中。

```
# set the organization ID **organization_id = '123456789012'****# create credentials as the service account**
credentials = ServiceAccountCredentials.from_json_keyfile_name(
    '**serviceaccount.json**',
    scopes
)# removing bindings from record before updating
del project['bindings']**# add a "parent" to the project record**
project['parent'] = {
    'type': 'organization',
    'id': **organization_id**
}params = {
    'projectId': project_id,
    'body': body,
}crm = build(
    'cloudresourcemanager', 
    'v1', 
    credentials=**credentials** )# update project parent to the organization_id
**crm.projects().update(**params).execute()**
```

**搞定！**由您的 G Suite 用户之一拥有但没有父项目的每个活动项目现在都是您的组织资源的成员，您可以开始管理您的所有用户的 GCP 资源了！

# 示例: [import_projects.py](https://github.com/lukwam/gcp-tools/blob/master/import_projects.py)

我已经在我的[**GCP-tools GitHub Repo**](https://github.com/lukwam/gcp-tools/)中添加了一个**新工具**，名为[**import _ projects . py**](https://github.com/lukwam/gcp-tools/blob/master/import_projects.py)**，**，它捆绑了本文描述的所有功能。使用正确配置的服务帐户和 G Suite 域中超级管理员的电子邮件地址运行它，将允许您执行以下所有操作:

1.  检索您的域中的所有用户
2.  对于每个用户，检索他们的所有项目
3.  筛选活动的“遗留项目”(归您的用户所有，但不在您的组织中)
4.  显示要导入的项目列表
5.  请求权限以继续导入项目
6.  对于每个要导入的项目，将服务帐户添加为所有者，并
7.  将项目添加到组织中。

下面是一个使用中的例子*(注意，****google@mydomain.com****是对 mydomain.com G Suite 域拥有超级管理员权限的用户名)*:

```
> **./import_projects.py google@mydomain.com**
Retrieving users from Google Admin SDK Directory API...
Found 13 users in domain.

Scanning all users for projects without a parent...
  lukwam@mydomain.com:
    * Test Project 1: test-project01 [367391543796] (ACTIVE)
    * Test Project 2: test-project02 [746509631927] (ACTIVE)
    * Test Project 3: test-project03 [216921026845] (ACTIVE)

Found 3 projects to import:
   * test-project01 (lukwam@mydomain.com)
   * test-project03 (lukwam@mydomain.com)
   * test-project02 (lukwam@mydomain.com)

Preparing to move 3 projects into org: mydomain.com...
   ---> Are you sure you want to continue? [y/N]: y

Organization: mydomain.com [685481217344] (customer: C0392o3bz)

   * test-project01:
      + added gcp-tools@ltk-gcp-tools.iam.gserviceaccount.com as project owner.
      + added project to organization 685481217344.

   * test-project02:
      + added gcp-tools@ltk-gcp-tools.iam.gserviceaccount.com as project owner.
      + added project to organization 685481217344.

   * test-project03:
      + added gcp-tools@ltk-gcp-tools.iam.gserviceaccount.com as project owner.
      + added project to organization 685481217344.

Done.
```

在这个项目的 [GitHub Wiki](https://github.com/lukwam/gcp-tools/wiki/Importing-Projects) 中可以获得 **import_projects.py** 的完整文档。bug 和功能请求可以通过 [GitHub Issues](https://github.com/lukwam/gcp-tools/issues) 提交。

# 参考资料:

*   [G 套件](https://gsuite.google.com/)
*   [谷歌云平台](https://cloud.google.com/)
*   [谷歌云平台博客](https://cloudplatform.googleblog.com/)
*   [管理 SDK 目录 API](https://developers.google.com/admin-sdk/directory/)
*   [云资源管理器](https://cloud.google.com/resource-manager/)
*   "[谷歌云资源管理器的新功能，以及其他 IAM 新闻](https://cloudplatform.googleblog.com/2016/10/whats-new-with-Google-Cloud-Resource-Manager-and-other-IAM-news.html) " ( [@grapesfrog](https://twitter.com/grapesfrog) )
*   "[用云资源管理器](https://cloudplatform.googleblog.com/2017/01/centrally-manage-all-your-Google-Cloud-resources-with-Cloud-Resource-Manager.html)集中管理你所有的谷歌云资源"( [@cavallimarco](https://twitter.com/cavallimarco) )
*   "[用谷歌云平台资源层级](https://cloudplatform.googleblog.com/2017/05/mapping-your-organization-with-the-Google-Cloud-Platform--resource-hierarchy.html)映射您的组织"( [@cavallimarco](https://twitter.com/cavallimarco) )